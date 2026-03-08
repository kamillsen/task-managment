# Rol/Yetki Yonetim Sistemi (Guncellenmiş Plan)

## 1. GENEL BAKIS

Bu dokumantasyon, Mini Jira projesi icin rol/yetki yonetim sistemini aciklar.
Sistem, **RBAC (Role-Based Access Control)** ile **Permission-based Authorization**'i birlestiren hibrit bir yaklasim kullanir.

### Temel Prensipler:

1. **Sadece spesifik yetkiler** - Wildcard yok (Project.All, Project.Create.* gibi seyler yok)
2. **Action seviyesinde kontrol** - Controller seviyesinde AuthorizePermission yok
3. **A.B.C formati zorunlu** - Her yetki 3 parcadan olusur, istisnasiz
4. **Otomatik kesif** - Sistem kodu tarar, yeni yetkileri bulur
5. **Admin onayi** - Kesif edilen yetkiler admin onayi olmadan aktif olmaz
6. **Her yetki bilincli atanir** - Sessiz yetki genislemesi yok

### Neden Wildcard Yok?

Wildcard (Project.All.All gibi) sorun olusturur:

```
Bugun PM'e Project.All.All verdin (10 yetki var, hepsi uygun)
6 ay sonra gelistirici ekledi: Project.Billing.Invoice
PM bunu da yapabiliyor → Admin haberi bile yok
```

Bu "sessiz yetki genislemesi" guvenlik acigidir.
Wildcard'in performans kazanci yok (HashSet.Contains() zaten O(1)).
RAM farki onemsiz (~4KB vs ~0.5KB per user).

---

## 2. DATABASE TABLOLARI

### 2.1 Tablolar

```sql
-- Kullanicilar
Users
├── ID (PK)
├── Email
├── PasswordHash
├── FullName
├── IsActive
├── CreatedAt
└── UpdatedAt

-- Roller
Roles
├── ID (PK)
├── Name (Admin, ProjectManager, Developer)
└── Description

-- Yetkiler
Permissions
├── ID (PK)
├── Name (Project.View.List)     -- Tam yetki adi
├── MainCategory (Project)       -- Ilk parca (gruplama icin)
├── SubCategory (View)           -- Ikinci parca (gruplama icin)
├── ActionName (List)            -- Ucuncu parca
├── Description
├── Status (Active/Inactive/Deprecated)
├── CreatedAt
└── CreatedBy

-- Kullanici-Rol Iliskisi
UserRoles
├── UserID (FK)
└── RoleID (FK)

-- Rol-Yetki Iliskisi
RolePermissions
├── RoleID (FK)
└── PermissionID (FK)

-- Kullanici-Direkt Yetki Iliskisi
UserPermissions
├── UserID (FK)
└── PermissionID (FK)

-- Kesfedilen Yetkiler (Onay bekleyenler)
DiscoveredPermissions
├── ID (PK)
├── Name (A.B.C formatinda)
├── MainCategory
├── SubCategory
├── ActionName
├── DiscoveredAt
├── DiscoveredBy ("System")
├── Status (Pending, Approved, Ignored)
└── Description
```

### 2.2 Gruplama Icin Kullanim

MainCategory ve SubCategory alanlari admin panelde gruplama, filtreleme ve toplu secim icin kullanilir:

```
Arama:      Name LIKE '%export%'
Filtreleme: WHERE MainCategory = 'Project' AND SubCategory = 'View'
Toplu Secim: Bir kategorideki mevcut yetkileri tek tek secer (wildcard degil)
```

### 2.3 Yetki Atama Yollari

Kullanici yetkiye 2 yoldan ulasir:

```
Yol 1: Rol uzerinden (dolayli)
  Kullanici → Rol → Yetkiler
  Kamil → Admin → [Project.View.List, Task.Create.New, ...]

Yol 2: Direkt yetki atama
  Kullanici → Yetki (rol araciligi olmadan)
  Ayse → Project.Export.PDF (sadece bu tek yetki)
```

---

## 3. ENTITY MODELLER

### 3.1 Permission.cs

```csharp
namespace MiniJira.Domain.Entities;

public class Permission
{
    public int Id { get; set; }
    public string Name { get; set; }           // "Project.View.List"
    public string MainCategory { get; set; }   // "Project"
    public string SubCategory { get; set; }    // "View"
    public string ActionName { get; set; }     // "List"
    public string Description { get; set; }
    public PermissionStatus Status { get; set; } = PermissionStatus.Active;
    public DateTime CreatedAt { get; set; }
    public string CreatedBy { get; set; }

    public virtual ICollection<Role> Roles { get; set; } = new List<Role>();
    public virtual ICollection<User> DirectUsers { get; set; } = new List<User>();

    public bool IsActive => Status == PermissionStatus.Active;
}

public enum PermissionStatus
{
    Active,     // Aktif, kullanilabilir
    Inactive,   // Pasif, kimse kullanamaz (gecici kapatma)
    Deprecated  // Koddan kaldirildi
}
```

### 3.2 DiscoveredPermission.cs

```csharp
namespace MiniJira.Domain.Entities;

public class DiscoveredPermission
{
    public int Id { get; set; }
    public string Name { get; set; }           // "Project.Archive.Old"
    public string MainCategory { get; set; }   // "Project"
    public string SubCategory { get; set; }    // "Archive"
    public string ActionName { get; set; }     // "Old"
    public DateTime DiscoveredAt { get; set; }
    public string DiscoveredBy { get; set; }   // "System"
    public DiscoveryStatus Status { get; set; } = DiscoveryStatus.Pending;
    public string Description { get; set; }
}

public enum DiscoveryStatus
{
    Pending,    // Admin onayi bekliyor
    Approved,   // Onaylandi, Permissions tablosuna eklendi
    Ignored     // Reddedildi
}
```

### 3.3 User.cs ve Role.cs

```csharp
namespace MiniJira.Domain.Entities;

public class User : IdentityUser
{
    public string FullName { get; set; }
    public virtual ICollection<Role> Roles { get; set; } = new List<Role>();
    public virtual ICollection<Permission> DirectPermissions { get; set; } = new List<Permission>();
}

public class Role
{
    public int Id { get; set; }
    public string Name { get; set; }           // "Admin", "ProjectManager", "Developer"
    public string Description { get; set; }
    public virtual ICollection<Permission> Permissions { get; set; } = new List<Permission>();
    public virtual ICollection<User> Users { get; set; } = new List<User>();
}
```

---

## 4. AUTHORIZE PERMISSION ATTRIBUTE

### 4.1 AuthorizePermissionAttribute

```csharp
namespace MiniJira.WebAPI.Attributes;

[AttributeUsage(AttributeTargets.Method, AllowMultiple = true)]
public class AuthorizePermissionAttribute : Attribute
{
    public string Name { get; }

    public AuthorizePermissionAttribute(string name)
    {
        Name = name;

        var parts = name.Split('.');
        if (parts.Length != 3)
        {
            throw new ArgumentException(
                $"Permission name must be in A.B.C format (3 parts). " +
                $"Example: 'Project.View.List'. Given: '{name}'");
        }

        if (string.IsNullOrWhiteSpace(parts[0]) ||
            string.IsNullOrWhiteSpace(parts[1]) ||
            string.IsNullOrWhiteSpace(parts[2]))
        {
            throw new ArgumentException(
                $"Permission parts cannot be empty. Format: A.B.C. Given: '{name}'");
        }
    }
}
```

**Not:** `AttributeTargets.Method` — sadece action seviyesinde kullanilabilir, controller seviyesinde kullanilamaz.

### 4.2 AuthorizePermissionFilter

```csharp
namespace MiniJira.WebAPI.Filters;

public class AuthorizePermissionFilter : IAuthorizationFilter
{
    private readonly string _permission;

    public AuthorizePermissionFilter(string permission)
    {
        _permission = permission;
    }

    public void OnAuthorization(AuthorizationFilterContext context)
    {
        if (!context.HttpContext.User.Identity.IsAuthenticated)
        {
            context.Result = new UnauthorizedResult();
            return;
        }

        var userIdClaim = context.HttpContext.User.FindFirstValue(ClaimTypes.NameIdentifier);

        if (string.IsNullOrEmpty(userIdClaim) || !int.TryParse(userIdClaim, out int userId))
        {
            context.Result = new UnauthorizedResult();
            return;
        }

        var permissionService = context.HttpContext.RequestServices
            .GetRequiredService<IPermissionService>();

        var hasPermission = permissionService
            .HasPermissionAsync(userId, _permission)
            .GetAwaiter().GetResult();

        if (!hasPermission)
        {
            context.Result = new ForbidResult();
        }
    }
}
```

---

## 5. PERMISSION SERVICE

### 5.1 IPermissionCache Interface (Redis Abstraction)

```csharp
namespace MiniJira.Application.Services;

public interface IPermissionCache
{
    Task<HashSet<string>> GetAsync(int userId);
    Task SetAsync(int userId, HashSet<string> permissions);
    Task RemoveAsync(int userId);
}
```

### 5.2 RedisPermissionCache Implementation

```csharp
using System.Text.Json;
using Microsoft.Extensions.Caching.Distributed;

namespace MiniJira.Infrastructure.Services;

public class RedisPermissionCache : IPermissionCache
{
    private readonly IDistributedCache _cache;
    private readonly DistributedCacheEntryOptions _cacheOptions;

    public RedisPermissionCache(IDistributedCache cache)
    {
        _cache = cache;
        _cacheOptions = new DistributedCacheEntryOptions
        {
            AbsoluteExpirationRelativeToNow = TimeSpan.FromMinutes(15)
        };
    }

    public async Task<HashSet<string>> GetAsync(int userId)
    {
        var json = await _cache.GetStringAsync($"userpermissions:{userId}");

        if (json == null)
            return null;

        return JsonSerializer.Deserialize<HashSet<string>>(json);
    }

    public async Task SetAsync(int userId, HashSet<string> permissions)
    {
        var json = JsonSerializer.Serialize(permissions);
        await _cache.SetStringAsync($"userpermissions:{userId}", json, _cacheOptions);
    }

    public async Task RemoveAsync(int userId)
    {
        await _cache.RemoveAsync($"userpermissions:{userId}");
    }
}
```

### 5.3 IPermissionService Interface

```csharp
namespace MiniJira.Application.Services;

public interface IPermissionService
{
    Task<bool> HasPermissionAsync(int userId, string permissionName);
    Task<List<string>> GetUserPermissionsAsync(int userId);
    Task<bool> IsInRoleAsync(int userId, string roleName);
    Task<List<string>> GetUserRolesAsync(int userId);
    Task InvalidateUserCacheAsync(int userId);
}
```

### 5.4 PermissionService Implementation

```csharp
namespace MiniJira.Infrastructure.Services;

public class PermissionService : IPermissionService
{
    private readonly AppDbContext _db;
    private readonly IPermissionCache _cache;

    public PermissionService(AppDbContext db, IPermissionCache cache)
    {
        _db = db;
        _cache = cache;
    }

    public async Task<bool> HasPermissionAsync(int userId, string permissionName)
    {
        var userPermissions = await _cache.GetAsync(userId);

        if (userPermissions == null)
        {
            userPermissions = await GetUserPermissionsFromDbAsync(userId);
            await _cache.SetAsync(userId, userPermissions);
        }

        return userPermissions.Contains(permissionName);
    }

    private async Task<HashSet<string>> GetUserPermissionsFromDbAsync(int userId)
    {
        // Rol uzerinden gelen yetkiler (sadece name, sadece Active)
        var rolePermissions = await _db.UserRoles
            .Where(ur => ur.UserId == userId)
            .SelectMany(ur => ur.Role.RolePermissions)
            .Where(rp => rp.Permission.Status == PermissionStatus.Active)
            .Select(rp => rp.Permission.Name)
            .Distinct()
            .ToListAsync();

        // Direkt atanmis yetkiler (sadece name, sadece Active)
        var directPermissions = await _db.UserPermissions
            .Where(up => up.UserId == userId)
            .Where(up => up.Permission.Status == PermissionStatus.Active)
            .Select(up => up.Permission.Name)
            .Distinct()
            .ToListAsync();

        var allPermissions = new HashSet<string>(rolePermissions);
        allPermissions.UnionWith(directPermissions);

        return allPermissions;
    }

    public async Task<List<string>> GetUserPermissionsAsync(int userId)
    {
        var userPermissions = await _cache.GetAsync(userId);

        if (userPermissions == null)
        {
            userPermissions = await GetUserPermissionsFromDbAsync(userId);
            await _cache.SetAsync(userId, userPermissions);
        }

        return userPermissions.ToList();
    }

    public async Task<bool> IsInRoleAsync(int userId, string roleName)
    {
        return await _db.UserRoles
            .Where(ur => ur.UserId == userId)
            .AnyAsync(ur => ur.Role.Name == roleName);
    }

    public async Task<List<string>> GetUserRolesAsync(int userId)
    {
        return await _db.UserRoles
            .Where(ur => ur.UserId == userId)
            .Select(ur => ur.Role.Name)
            .ToListAsync();
    }

    public async Task InvalidateUserCacheAsync(int userId)
    {
        await _cache.RemoveAsync(userId);
    }
}
```

### 5.5 Performans Detaylari

```
DB Sorgusu: Sadece permission name'leri cekilir (Include yok, projection)
  Eski: ~50-100KB (tum entity'ler)
  Yeni: ~2KB (sadece string listesi)

Cache: Redis, user bazinda tek key
  Key: "userpermissions:{userId}"
  Value: JSON string (HashSet<string> serialize edilmis)
  Sure: 15 dakika
  Kontrol: Deserialize → HashSet.Contains() → O(1)

Redis Avantajlari:
  - Uygulama RAM'ini kullanmaz (Redis kendi sunucusunda)
  - Birden fazla servis ayni cache'i paylasir
  - Uygulama yeniden baslatilsa bile cache kaybolmaz
  - Asama 2'ye geciste hazir (tum servisler ayni Redis'e bakar)
```

---

## 6. PERMISSION REGISTRY SERVICE (Otomatik Kesif)

### 6.1 IPermissionRegistry Interface

```csharp
namespace MiniJira.Application.Services;

public interface IPermissionRegistry
{
    Task<List<DiscoveredPermission>> DiscoverNewPermissionsAsync();
    Task<Permission> ApprovePermissionAsync(int discoveredPermissionId, string approvedBy, List<int> roleIds);
    Task IgnorePermissionAsync(int discoveredPermissionId, string ignoredBy);
    Task UpdatePermissionStatusAsync(int permissionId, PermissionStatus status, string reason, string updatedBy);
    (string Main, string Sub, string Action) ParsePermissionName(string permissionName);
}
```

### 6.2 PermissionRegistry Implementation

```csharp
namespace MiniJira.Infrastructure.Services;

public class PermissionRegistry : IPermissionRegistry
{
    private readonly AppDbContext _db;
    private readonly IPermissionCache _cache;
    private readonly ILogger<PermissionRegistry> _logger;

    public PermissionRegistry(AppDbContext db, IPermissionCache cache, ILogger<PermissionRegistry> logger)
    {
        _db = db;
        _cache = cache;
        _logger = logger;
    }

    public (string Main, string Sub, string Action) ParsePermissionName(string permissionName)
    {
        var parts = permissionName.Split('.');

        if (parts.Length != 3)
        {
            throw new InvalidPermissionFormatException(
                $"Permission '{permissionName}' must be in A.B.C format. " +
                $"Example: 'Project.View.List'. Found {parts.Length} parts.");
        }

        if (string.IsNullOrWhiteSpace(parts[0]) ||
            string.IsNullOrWhiteSpace(parts[1]) ||
            string.IsNullOrWhiteSpace(parts[2]))
        {
            throw new InvalidPermissionFormatException(
                "Permission parts cannot be empty. Format: A.B.C");
        }

        return (parts[0].Trim(), parts[1].Trim(), parts[2].Trim());
    }

    public async Task<List<DiscoveredPermission>> DiscoverNewPermissionsAsync()
    {
        _logger.LogInformation("Starting permission discovery...");

        // 1. Koddaki tum [AuthorizePermission] attribute'larini bul
        var permissionsInCode = ScanCodeForPermissions();

        // 2. DB'deki mevcut permission'lari getir
        var existingPermissions = await _db.Permissions
            .Select(p => p.Name)
            .ToListAsync();

        // 3. Daha once kesfedilmis olanlari getir
        var existingDiscoveries = await _db.DiscoveredPermissions
            .Select(d => d.Name)
            .ToListAsync();

        // 4. Gercekten yeni olanlari bul
        var trulyNew = permissionsInCode
            .Except(existingPermissions)
            .Except(existingDiscoveries)
            .ToList();

        // 5. Parse et ve kaydet
        var discoveries = new List<DiscoveredPermission>();

        foreach (var permName in trulyNew)
        {
            try
            {
                var (main, sub, action) = ParsePermissionName(permName);

                discoveries.Add(new DiscoveredPermission
                {
                    Name = permName,
                    MainCategory = main,
                    SubCategory = sub,
                    ActionName = action,
                    DiscoveredAt = DateTime.UtcNow,
                    DiscoveredBy = "System",
                    Status = DiscoveryStatus.Pending,
                    Description = $"Discovered from code at {DateTime.UtcNow}"
                });
            }
            catch (InvalidPermissionFormatException ex)
            {
                _logger.LogWarning(ex, $"Skipping invalid permission format: {permName}");
            }
        }

        if (discoveries.Any())
        {
            await _db.DiscoveredPermissions.AddRangeAsync(discoveries);
            await _db.SaveChangesAsync();
            _logger.LogInformation($"Discovered {discoveries.Count} new permissions");
        }

        return discoveries;
    }

    // Onaylama + Rol atama tek adimda
    public async Task<Permission> ApprovePermissionAsync(
        int discoveredPermissionId, string approvedBy, List<int> roleIds)
    {
        var discovery = await _db.DiscoveredPermissions
            .FirstOrDefaultAsync(d => d.Id == discoveredPermissionId &&
                                     d.Status == DiscoveryStatus.Pending);

        if (discovery == null)
            throw new NotFoundException("Pending permission not found");

        // Permission olustur
        var permission = new Permission
        {
            Name = discovery.Name,
            MainCategory = discovery.MainCategory,
            SubCategory = discovery.SubCategory,
            ActionName = discovery.ActionName,
            Description = $"Approved by {approvedBy} at {DateTime.UtcNow}",
            CreatedAt = DateTime.UtcNow,
            CreatedBy = approvedBy,
            Status = PermissionStatus.Active
        };

        await _db.Permissions.AddAsync(permission);
        discovery.Status = DiscoveryStatus.Approved;

        // Secilen rollere ata
        if (roleIds.Any())
        {
            var roles = await _db.Roles
                .Where(r => roleIds.Contains(r.Id))
                .ToListAsync();

            foreach (var role in roles)
            {
                role.Permissions.Add(permission);
            }

            // Bu rollerdeki kullanicilarin cache'ini temizle
            var userIds = await _db.UserRoles
                .Where(ur => roleIds.Contains(ur.RoleId))
                .Select(ur => ur.UserId)
                .Distinct()
                .ToListAsync();

            foreach (var userId in userIds)
            {
                await InvalidateUserCacheAsync(userId);
            }
        }

        await _db.SaveChangesAsync();
        _logger.LogInformation($"Permission approved: {permission.Name} by {approvedBy}");

        return permission;
    }

    public async Task IgnorePermissionAsync(int discoveredPermissionId, string ignoredBy)
    {
        var discovery = await _db.DiscoveredPermissions
            .FirstOrDefaultAsync(d => d.Id == discoveredPermissionId &&
                                     d.Status == DiscoveryStatus.Pending);

        if (discovery == null)
            throw new NotFoundException("Pending permission not found");

        discovery.Status = DiscoveryStatus.Ignored;
        await _db.SaveChangesAsync();
        _logger.LogInformation($"Permission ignored: {discovery.Name} by {ignoredBy}");
    }

    public async Task UpdatePermissionStatusAsync(
        int permissionId, PermissionStatus status, string reason, string updatedBy)
    {
        var permission = await _db.Permissions.FindAsync(permissionId);

        if (permission == null)
            throw new NotFoundException("Permission not found");

        permission.Status = status;
        await _db.SaveChangesAsync();

        // Bu yetkiye sahip tum kullanicilarin cache'ini temizle
        var userIds = await _db.Users
            .Where(u => u.Roles.Any(r => r.Permissions.Any(p => p.Id == permissionId)) ||
                       u.DirectPermissions.Any(p => p.Id == permissionId))
            .Select(u => u.Id)
            .ToListAsync();

        foreach (var userId in userIds)
        {
            await InvalidateUserCacheAsync(userId);
        }

        _logger.LogInformation(
            $"Permission {permission.Name} status changed to {status} by {updatedBy}");
    }

    private async Task InvalidateUserCacheAsync(int userId)
    {
        await _cache.RemoveAsync(userId);
    }

    private List<string> ScanCodeForPermissions()
    {
        var permissions = new List<string>();
        var assembly = Assembly.GetExecutingAssembly();

        var controllerTypes = assembly.GetTypes()
            .Where(t => typeof(ControllerBase).IsAssignableFrom(t));

        foreach (var controllerType in controllerTypes)
        {
            var methods = controllerType.GetMethods(
                BindingFlags.Public | BindingFlags.Instance | BindingFlags.DeclaredOnly);

            foreach (var method in methods)
            {
                var attrs = method.GetCustomAttributes<AuthorizePermissionAttribute>(true);
                foreach (var attr in attrs)
                {
                    if (!string.IsNullOrEmpty(attr.Name))
                        permissions.Add(attr.Name);
                }
            }
        }

        return permissions.Distinct().ToList();
    }
}
```

---

## 7. ADMIN CONTROLLER'LAR

### 7.1 PermissionsController (Kesif ve Durum Yonetimi)

```csharp
namespace MiniJira.WebAPI.Controllers.Admin;

[Route("api/admin/permissions")]
[ApiController]
[Authorize]
public class PermissionsController : ControllerBase
{
    private readonly IPermissionRegistry _permissionRegistry;
    private readonly AppDbContext _db;

    public PermissionsController(IPermissionRegistry permissionRegistry, AppDbContext db)
    {
        _permissionRegistry = permissionRegistry;
        _db = db;
    }

    // Yeni yetkileri kesfet
    [HttpGet("discover")]
    [AuthorizePermission("Admin.Permission.Discover")]
    public async Task<IActionResult> DiscoverPermissions()
    {
        await _permissionRegistry.DiscoverNewPermissionsAsync();

        var pending = await _db.DiscoveredPermissions
            .Where(d => d.Status == DiscoveryStatus.Pending)
            .OrderByDescending(d => d.DiscoveredAt)
            .Select(d => new
            {
                d.Id, d.Name, d.MainCategory, d.SubCategory,
                d.ActionName, d.DiscoveredAt, d.Description
            })
            .ToListAsync();

        return Ok(pending);
    }

    // Onayla + Rollere ata (tek adimda)
    [HttpPost("{id}/approve")]
    [AuthorizePermission("Admin.Permission.Approve")]
    public async Task<IActionResult> ApprovePermission(int id, [FromBody] ApproveRequest request)
    {
        var approvedBy = User.FindFirstValue(ClaimTypes.Name);

        var permission = await _permissionRegistry
            .ApprovePermissionAsync(id, approvedBy, request.RoleIds);

        return Ok(new
        {
            Message = $"Permission '{permission.Name}' approved",
            Permission = new { permission.Id, permission.Name },
            AssignedToRoles = request.RoleIds
        });
    }

    // Reddet
    [HttpPost("{id}/ignore")]
    [AuthorizePermission("Admin.Permission.Ignore")]
    public async Task<IActionResult> IgnorePermission(int id)
    {
        var ignoredBy = User.FindFirstValue(ClaimTypes.Name);
        await _permissionRegistry.IgnorePermissionAsync(id, ignoredBy);
        return Ok(new { Message = "Permission ignored" });
    }

    // Durumu guncelle (Active/Inactive)
    [HttpPut("{id}/status")]
    [AuthorizePermission("Admin.Permission.Status")]
    public async Task<IActionResult> UpdateStatus(int id, [FromBody] UpdateStatusRequest request)
    {
        var updatedBy = User.FindFirstValue(ClaimTypes.Name);
        await _permissionRegistry.UpdatePermissionStatusAsync(
            id, request.Status, request.Reason, updatedBy);
        return Ok(new { Message = $"Status updated to {request.Status}" });
    }

    // Tum yetkileri listele (gruplu)
    [HttpGet]
    [AuthorizePermission("Admin.Permission.View")]
    public async Task<IActionResult> GetAllPermissions()
    {
        var permissions = await _db.Permissions
            .OrderBy(p => p.MainCategory)
            .ThenBy(p => p.SubCategory)
            .ThenBy(p => p.Name)
            .Select(p => new
            {
                p.Id, p.Name, p.MainCategory, p.SubCategory,
                p.ActionName, p.Status, p.Description
            })
            .ToListAsync();

        return Ok(permissions);
    }
}

public class ApproveRequest
{
    public List<int> RoleIds { get; set; } = new();
}

public class UpdateStatusRequest
{
    public PermissionStatus Status { get; set; }
    public string Reason { get; set; }
}
```

### 7.2 RolesController (Rol ve Yetki Atama)

```csharp
namespace MiniJira.WebAPI.Controllers.Admin;

[Route("api/admin/roles")]
[ApiController]
[Authorize]
public class RolesController : ControllerBase
{
    private readonly AppDbContext _db;
    private readonly IPermissionService _permissionService;

    public RolesController(AppDbContext db, IPermissionService permissionService)
    {
        _db = db;
        _permissionService = permissionService;
    }

    // Tum yetkileri getir (rol atama ekrani icin, gruplu)
    [HttpGet("permissions")]
    [AuthorizePermission("Admin.Role.View")]
    public async Task<IActionResult> GetAllPermissions()
    {
        var permissions = await _db.Permissions
            .Where(p => p.Status == PermissionStatus.Active)
            .OrderBy(p => p.MainCategory)
            .ThenBy(p => p.SubCategory)
            .Select(p => new
            {
                p.Id, p.Name, p.MainCategory, p.SubCategory,
                p.ActionName, p.Description
            })
            .ToListAsync();

        return Ok(permissions);
    }

    // Tum rolleri listele
    [HttpGet]
    [AuthorizePermission("Admin.Role.View")]
    public async Task<IActionResult> GetAllRoles()
    {
        var roles = await _db.Roles
            .Include(r => r.Permissions)
            .Select(r => new
            {
                r.Id, r.Name, r.Description,
                Permissions = r.Permissions
                    .Where(p => p.Status == PermissionStatus.Active)
                    .Select(p => p.Name)
                    .ToList()
            })
            .ToListAsync();

        return Ok(roles);
    }

    // Rol yetkilerini guncelle
    [HttpPost("{roleId}/permissions")]
    [AuthorizePermission("Admin.Role.Manage")]
    public async Task<IActionResult> UpdateRolePermissions(
        int roleId, [FromBody] List<string> permissionNames)
    {
        var role = await _db.Roles
            .Include(r => r.Permissions)
            .FirstOrDefaultAsync(r => r.Id == roleId);

        if (role == null) return NotFound();

        // Sadece Active yetkileri ata
        var permissions = await _db.Permissions
            .Where(p => permissionNames.Contains(p.Name) &&
                       p.Status == PermissionStatus.Active)
            .ToListAsync();

        role.Permissions.Clear();
        role.Permissions = permissions;
        await _db.SaveChangesAsync();

        // Bu role sahip kullanicilarin cache'ini temizle
        var userIds = await _db.UserRoles
            .Where(ur => ur.RoleId == roleId)
            .Select(ur => ur.UserId)
            .ToListAsync();

        foreach (var userId in userIds)
        {
            await _permissionService.InvalidateUserCacheAsync(userId);
        }

        return Ok(new { Message = "Rol yetkileri guncellendi" });
    }

    // Kullaniciya rol ata
    [HttpPost("{roleId}/users/{userId}")]
    [AuthorizePermission("Admin.Role.Assign")]
    public async Task<IActionResult> AssignRoleToUser(int roleId, int userId)
    {
        var role = await _db.Roles.FindAsync(roleId);
        var user = await _db.Users.Include(u => u.Roles).FirstOrDefaultAsync(u => u.Id == userId);

        if (role == null || user == null) return NotFound();

        if (!user.Roles.Contains(role))
        {
            user.Roles.Add(role);
            await _db.SaveChangesAsync();
            await _permissionService.InvalidateUserCacheAsync(userId);
        }

        return Ok(new { Message = "Rol atandi" });
    }

    // Kullanicidan rol kaldir
    [HttpDelete("{roleId}/users/{userId}")]
    [AuthorizePermission("Admin.Role.Remove")]
    public async Task<IActionResult> RemoveRoleFromUser(int roleId, int userId)
    {
        var user = await _db.Users.Include(u => u.Roles).FirstOrDefaultAsync(u => u.Id == userId);
        var role = await _db.Roles.FindAsync(roleId);

        if (role == null || user == null) return NotFound();

        user.Roles.Remove(role);
        await _db.SaveChangesAsync();
        await _permissionService.InvalidateUserCacheAsync(userId);

        return Ok(new { Message = "Rol kaldirildi" });
    }
}
```

---

## 8. CONTROLLER KULLANIM ORNEKLERI (CQRS Ayrimi)

Controller'lar CQRS pattern'ina gore ayrilir. Okuma ve yazma islemleri ayri controller'larda.
Controller seviyesinde AuthorizePermission YOKTUR. Sadece action seviyesinde kullanilir.

### 8.1 Project Controller'lari

```csharp
// Okuma islemleri
[ApiController]
[Route("api/projects")]
[Authorize]
public class ProjectViewsController : ControllerBase
{
    [HttpGet]
    [AuthorizePermission("Project.View.List")]
    public IActionResult GetProjects() { ... }

    [HttpGet("{id}")]
    [AuthorizePermission("Project.View.Detail")]
    public IActionResult GetProjectById(int id) { ... }

    [HttpGet("{id}/members")]
    [AuthorizePermission("Project.View.Members")]
    public IActionResult GetProjectMembers(int id) { ... }

    [HttpGet("{id}/statistics")]
    [AuthorizePermission("Project.View.Statistics")]
    public IActionResult GetProjectStatistics(int id) { ... }
}

// Yazma islemleri
[ApiController]
[Route("api/projects")]
[Authorize]
public class ProjectCommandsController : ControllerBase
{
    [HttpPost]
    [AuthorizePermission("Project.Create.New")]
    public IActionResult CreateProject([FromBody] CreateProjectDto dto) { ... }

    [HttpPut("{id}")]
    [AuthorizePermission("Project.Edit.Any")]
    public IActionResult UpdateProject(int id, [FromBody] UpdateProjectDto dto) { ... }

    [HttpDelete("{id}")]
    [AuthorizePermission("Project.Delete.Remove")]
    public IActionResult DeleteProject(int id) { ... }

    [HttpPost("{id}/export")]
    [AuthorizePermission("Project.Export.PDF")]
    public IActionResult ExportProjectToPdf(int id) { ... }
}
```

### 8.2 Task Controller'lari

```csharp
[ApiController]
[Route("api/tasks")]
[Authorize]
public class TaskViewsController : ControllerBase
{
    [HttpGet]
    [AuthorizePermission("Task.View.List")]
    public IActionResult GetTasks() { ... }

    [HttpGet("{id}")]
    [AuthorizePermission("Task.View.Detail")]
    public IActionResult GetTaskById(int id) { ... }
}

[ApiController]
[Route("api/tasks")]
[Authorize]
public class TaskCommandsController : ControllerBase
{
    [HttpPost]
    [AuthorizePermission("Task.Create.New")]
    public IActionResult CreateTask([FromBody] CreateTaskDto dto) { ... }

    [HttpPut("{id}")]
    [AuthorizePermission("Task.Edit.Any")]
    public IActionResult UpdateTask(int id, [FromBody] UpdateTaskDto dto) { ... }

    [HttpPut("{id}/status")]
    [AuthorizePermission("Task.Status.Update")]
    public IActionResult UpdateTaskStatus(int id, [FromBody] UpdateStatusDto dto) { ... }

    [HttpPost("{id}/assign")]
    [AuthorizePermission("Task.Assign.User")]
    public IActionResult AssignTask(int id, [FromBody] AssignTaskDto dto) { ... }
}
```

---

## 9. SEED DATA

```csharp
namespace MiniJira.Infrastructure.Data;

public static class SeedData
{
    public static void Initialize(AppDbContext context)
    {
        if (context.Roles.Any()) return;

        var permissions = new List<Permission>
        {
            // Project yetkileri
            P("Project.View.List", "Proje listesini goruntule"),
            P("Project.View.Detail", "Proje detayini goruntule"),
            P("Project.View.Members", "Proje uyelerini goruntule"),
            P("Project.View.Statistics", "Proje istatistiklerini goruntule"),
            P("Project.Create.New", "Yeni proje olustur"),
            P("Project.Edit.Any", "Projeyi duzenle"),
            P("Project.Delete.Remove", "Projeyi sil"),
            P("Project.Export.PDF", "Projeyi PDF olarak export et"),

            // Task yetkileri
            P("Task.View.List", "Gorev listesini goruntule"),
            P("Task.View.Detail", "Gorev detayini goruntule"),
            P("Task.Create.New", "Yeni gorev olustur"),
            P("Task.Edit.Any", "Gorevi duzenle"),
            P("Task.Status.Update", "Gorev durumunu guncelle"),
            P("Task.Assign.User", "Gorev kullaniciya ata"),

            // User yetkileri
            P("User.View.List", "Kullanici listesini goruntule"),
            P("User.View.Detail", "Kullanici detayini goruntule"),
            P("User.Edit.Profile", "Kullanici profilini duzenle"),

            // Admin yetkileri
            P("Admin.Role.View", "Rolleri goruntule"),
            P("Admin.Role.Manage", "Rol yetkilerini yonet"),
            P("Admin.Role.Assign", "Kullaniciya rol ata"),
            P("Admin.Role.Remove", "Kullanicidan rol kaldir"),
            P("Admin.Permission.View", "Yetkileri goruntule"),
            P("Admin.Permission.Discover", "Yeni yetkileri kesfet"),
            P("Admin.Permission.Approve", "Kesfedilen yetkiyi onayla"),
            P("Admin.Permission.Ignore", "Kesfedilen yetkiyi reddet"),
            P("Admin.Permission.Status", "Yetki durumunu guncelle"),
        };

        context.Permissions.AddRange(permissions);
        context.SaveChanges();

        // Admin: tum yetkiler
        var adminRole = new Role
        {
            Name = "Admin",
            Description = "Sistem yoneticisi, tum yetkilere sahip",
            Permissions = permissions
        };

        // PM: Project + Task + User.View
        var pmRole = new Role
        {
            Name = "ProjectManager",
            Description = "Proje yoneticisi",
            Permissions = permissions
                .Where(p => p.MainCategory == "Project" ||
                           p.MainCategory == "Task" ||
                           p.Name == "User.View.List" ||
                           p.Name == "User.View.Detail")
                .ToList()
        };

        // Developer: sadece goruntuleme + gorev olusturma
        var devRole = new Role
        {
            Name = "Developer",
            Description = "Yazilim gelistirici",
            Permissions = permissions
                .Where(p => p.Name == "Project.View.List" ||
                           p.Name == "Project.View.Detail" ||
                           p.Name == "Task.View.List" ||
                           p.Name == "Task.View.Detail" ||
                           p.Name == "Task.Create.New" ||
                           p.Name == "Task.Status.Update")
                .ToList()
        };

        context.Roles.AddRange(adminRole, pmRole, devRole);
        context.SaveChanges();
    }

    private static Permission P(string name, string description)
    {
        var parts = name.Split('.');
        return new Permission
        {
            Name = name,
            MainCategory = parts[0],
            SubCategory = parts[1],
            ActionName = parts[2],
            Description = description,
            Status = PermissionStatus.Active,
            CreatedAt = DateTime.UtcNow,
            CreatedBy = "System"
        };
    }
}
```

---

## 10. DEPENDENCY INJECTION (Program.cs)

```csharp
var builder = WebApplication.CreateBuilder(args);

// Redis cache
builder.Services.AddStackExchangeRedisCache(options =>
{
    options.Configuration = builder.Configuration.GetConnectionString("Redis");
});

// Permission servisleri
builder.Services.AddSingleton<IPermissionCache, RedisPermissionCache>();
builder.Services.AddScoped<IPermissionService, PermissionService>();
builder.Services.AddScoped<IPermissionRegistry, PermissionRegistry>();

var app = builder.Build();

using (var scope = app.Services.CreateScope())
{
    var context = scope.ServiceProvider.GetRequiredService<AppDbContext>();
    SeedData.Initialize(context);
}

app.Run();
```

appsettings.json:

```json
{
  "ConnectionStrings": {
    "DefaultConnection": "Host=localhost;Port=5432;Database=taskmanagement;...",
    "Redis": "localhost:6379"
  }
}
```

docker-compose.yml'a Redis container eklenir:

```yaml
redis:
  image: redis:7-alpine
  ports:
    - "6379:6379"
  volumes:
    - redis_data:/data
```

---

## 11. ADMIN PANEL GORUNUMU

### 11.1 Yetki Atama Ekrani

```
Rol: ProjectManager - Yetki Atama

Filtre: [Project v] [Tumu v]  Ara: [____________]

  Project
    View                              [Tumunu Sec]
      [x] Project.View.List
      [x] Project.View.Detail
      [ ] Project.View.Members
      [ ] Project.View.Statistics
    Create                            [Tumunu Sec]
      [x] Project.Create.New
    Edit                              [Tumunu Sec]
      [ ] Project.Edit.Any
    Delete                            [Tumunu Sec]
      [ ] Project.Delete.Remove

  Task
    View                              [Tumunu Sec]
      [x] Task.View.List
      [x] Task.View.Detail
    Create                            [Tumunu Sec]
      [x] Task.Create.New

                            [Kaydet]
```

"Tumunu Sec" butonu o kategorideki mevcut yetkileri tek tek secer.
Wildcard degildir. Ileride yeni yetki eklenirse otomatik secilmez.

### 11.2 Yeni Yetki Kesfi Ekrani

```
Yeni Kesfedilen Yetkiler

+-----------------------------+------------------+
| Yetki                       | Islem            |
+-----------------------------+------------------+
| Project.Archive.Old         | [Onayla] [Reddet]|
| Task.Comment.Add            | [Onayla] [Reddet]|
+-----------------------------+------------------+

Onayla tiklaninca:
+------------------------------------------+
| Yetki: Project.Archive.Old               |
|                                          |
| Hangi rollere atansin?                   |
|   [x] Admin                             |
|   [ ] ProjectManager                    |
|   [ ] Developer                         |
|                                          |
|              [Onayla ve Ata]             |
+------------------------------------------+
```

Onaylama ve rol atama tek adimda yapilir.

---

## 12. CACHE INVALIDATION

Cache su durumlarda temizlenir:

```
1. Rol yetkileri degistiginde
   → O role sahip TUM kullanicilarin cache'i temizlenir

2. Kullaniciya rol atandiginda/kaldirildiginda
   → O kullanicinin cache'i temizlenir

3. Permission durumu degistiginde (Active → Inactive)
   → O yetkiye sahip TUM kullanicilarin cache'i temizlenir

4. Yeni yetki onaylanip rollere atandiginda
   → O rollere sahip kullanicilarin cache'i temizlenir
```

---

## 13. OLCEKLEME PLANI

### 13.1 Cache Stratejisi

Bastan Redis kullanilir. Olcek farketmez, her durumda hazir:

```
Redis Avantajlari:
  - Uygulama RAM'ini kullanmaz
  - Birden fazla servis ayni cache'i paylasir (Asama 2 icin hazir)
  - Uygulama yeniden baslatilsa bile cache kaybolmaz
  - Yetki degistiginde tum servisler aninda guncel veriyi gorur

Cache Yapisi:
  Key: "userpermissions:{userId}"
  Value: JSON string (HashSet<string>)
  TTL: 15 dakika

RAM Hesabi (Redis tarafinda):
  200 yetki x ~30 byte = ~6 KB per user (JSON formatinda)
  1.000 user = 6 MB
  100.000 user = 600 MB
  Redis bunu rahat tasir
```

### 13.2 DB Stratejisi (2 Asamali)

```
Asama 1 (Simdi - Mini Jira):
  Shared DB, Separate Schema
  Identity Service → identity schema
  Project Service  → project schema
  Task Service     → task schema
  Ayni PostgreSQL sunucusu

Asama 2 (Baska projelerde kullanmak istediginde):
  Identity Service → ayri DB, ayri microservis
  Hazir bir auth servisi olarak elinde bulunur
  Yeni projeye baslayinca ayni servisi baglarsin
  Diger servisler Identity Service'e HTTP ile sorar
  Kod degismez, sadece connection string ve DI kaydi degisir
```

### 13.3 Servisler Arasi Yetki Kontrolu (Asama 2 icin)

Bu bolum Asama 2 icin gecerlidir (Identity Service ayri DB/microservis oldugunda).
Asama 1'de ayni DB kullanildigi icin direkt sorgu yeterlidir.

DB ayrildiginda diger servisler yetkileri HTTP ile sorar:

```
Task Service → GET /api/auth/permissions/{userId} → Identity Service
Task Service → cevabi cache'ler → Contains() ile kontrol eder
```

```csharp
// Asama 1: DB'den direkt oku
builder.Services.AddScoped<IPermissionService, DbPermissionService>();

// Asama 2: HTTP ile Identity Service'e sor
builder.Services.AddScoped<IPermissionService, HttpPermissionService>();
```

Interface ayni kaldigi icin uygulama kodu, controller'lar, filter'lar degismez.

### 13.4 JWT Token Yapisi ve Yetki Tasima (Asama 2 icin)

Bu bolum Asama 2 icin gecerlidir.

JWT'de yetki bilgisi TUTULMAZ. Token yalin kalir:

```json
{
  "sub": "5",
  "name": "Ahmet",
  "role": "ProjectManager",
  "exp": 1234567890
}
```

Neden?
- 200 yetki token'a koyulursa ~8KB → HTTP header limitine carpar
- Yetki degisikligi aninda yansimaz (token suresi dolmadan)
- Yetkiler PermissionService uzerinden sorulur (cache ile hizli)

Peki yetkiler nasil tasinir? JWT sadece userId'yi tasir, bu yeterli:

```
JWT tasiyor      → KIM oldugunu (userId)
Identity Service → NE YAPABILECEGINI (permissions)
```

Akis:

```
1. Ahmet login oldu → JWT aldi (icinde sub: 5 var)

2. Ahmet → POST /api/tasks → Task Service'e istek atti
   Header: Authorization: Bearer eyJhbGciOi...

3. Task Service JWT'yi parse etti → userId: 5 aldi

4. Task Service: "5 nolu adamin yetkileri ne? Sorayim"
   → GET /api/auth/permissions/5 → Identity Service

5. Identity Service → kendi DB'sinden cekti → dondu:
   ["Task.View.List", "Task.Create.New", "Project.View.List"]

6. Task Service → bu listeyi cache'ledi (15 dk)

7. Task Service → Contains("Task.Create.New") → true → islem yapilir

Sonraki isteklerde:
   Cache'den bak → var → Identity Service'e sormaya gerek yok
   Cache suresi dolunca → tekrar sor → cache'i yenile
```

JWT bir kimlik karti gibi dusunulebilir. Ustunde adin ve numaranin yaziyor.
Ama hangi odalara girebilecegin guvenlik masasinda (Identity Service) kayitli.

---

## 14. SISTEM AKISI (Ozet)

```
1. GELISTIRICI:
   [AuthorizePermission("Project.Archive.Old")] yazar
   Format: A.B.C zorunlu, baska format derlenmez

2. SISTEM (Otomatik Kesif):
   Kodu tarar → yeni yetkiyi bulur
   DiscoveredPermissions tablosuna yazar (Status: Pending)

3. ADMIN:
   Panelden gorur → onaylar + rollere atar (tek adimda)
   Veya reddeder

4. RUNTIME (Kullanici istek attiginda):
   Filter: "Bu endpoint Project.Archive.Old istiyor"
   PermissionService: Cache'den HashSet al → Contains() → true/false
   Sonuc: 200 OK veya 403 Forbidden

5. CACHE INVALIDATION:
   Yetki/rol degistiginde ilgili kullanicilarin cache'i temizlenir
   Sonraki istekte DB'den tekrar cekilir
```

---

## 15. KURULUM ADIMLARI

```bash
# 1. Migration olustur
dotnet ef migrations add AddPermissionSystem --project Infrastructure --startup-project WebAPI

# 2. Migration uygula
dotnet ef database update --project Infrastructure --startup-project WebAPI
```

```bash
# 3. Redis'i baslat (docker-compose icinde)
docker-compose up -d redis
```

```csharp
// 4. Program.cs'e ekle
builder.Services.AddStackExchangeRedisCache(options =>
{
    options.Configuration = builder.Configuration.GetConnectionString("Redis");
});
builder.Services.AddSingleton<IPermissionCache, RedisPermissionCache>();
builder.Services.AddScoped<IPermissionService, PermissionService>();
builder.Services.AddScoped<IPermissionRegistry, PermissionRegistry>();

// 5. Seed data calistir
using (var scope = app.Services.CreateScope())
{
    var context = scope.ServiceProvider.GetRequiredService<AppDbContext>();
    SeedData.Initialize(context);
}
```

```csharp
// 6. Controller'lara attribute ekle
[AuthorizePermission("Project.View.List")]   // 3 parca zorunlu
public IActionResult GetProjects() { ... }
```
