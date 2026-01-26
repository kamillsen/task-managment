# 🚀 **"BEN İYİYİM" DEDİRTECEK ROL/YETKİ YÖNETİM SİSTEMİ**

## 📋 **GENEL BAKIŞ**

Bu dokümantasyon, Mini Jira projesi için **gerçekten modern, geliştirilebilir ve "wow" dedirtecek** bir merkezi rol/yetki yönetim sistemini açıklar. Sistem, **RBAC (Role-Based Access Control)** ile **Permission-based Authorization'ı** birleştiren hibrit bir yaklaşım kullanır.

---

## 🏗️ **1. DATABASE TABLOLARI**

### **Temel Tablo Yapısı**

```sql
-- Users Tablosu (Mevcut Identity tablosu genişletilmiş)
Users
├── ID (PK)
├── Email
├── Name
└── ...

-- Roller Tablosu
Roles
├── ID (PK)
├── Name (Admin, ProjectManager, Developer)
└── Description

-- Yetkiler Tablosu
Permissions
├── ID (PK)
├── Name (Project.Create, Project.View, Task.Assign)
├── Category (Project, Task, User)
└── Description

-- Kullanıcı-Rol İlişkisi (Many-to-Many)
UserRoles
├── UserID (FK)
└── RoleID (FK)

-- Rol-Yetki İlişkisi (Many-to-Many)
RolePermissions
├── RoleID (FK)
└── PermissionID (FK)

-- Kullanıcı-Direkt Yetki İlişkisi (Many-to-Many)
UserPermissions
├── UserID (FK)
└── PermissionID (FK)
```

**Not:** `UserPermissions` tablosu, kullanıcılara rol dışında direkt yetki vermek için kullanılır (hibrit sistem).

---

## 💻 **2. KOD YAPISI - ÇOK BASİT ANLAŞILIR**

### **2.1 Domain Entities**

#### **Role.cs**

```csharp
namespace MiniJira.Domain.Entities;

public class Role
{
    public int Id { get; set; }
    public string Name { get; set; } // "Admin", "ProjectManager", "Developer"
    public string Description { get; set; }
    
    // BU ÖNEMLİ: Rollerin yetkileri
    public virtual ICollection<Permission> Permissions { get; set; } = new List<Permission>();
    
    // Kullanıcılar (Many-to-Many)
    public virtual ICollection<User> Users { get; set; } = new List<User>();
}
```

#### **Permission.cs**

```csharp
namespace MiniJira.Domain.Entities;

public class Permission
{
    public int Id { get; set; }
    public string Name { get; set; } // "Project.Create", "Project.ViewAll"
    public string Category { get; set; } // "Project", "Task", "User"
    public string Description { get; set; }
    
    // Roller (Many-to-Many)
    public virtual ICollection<Role> Roles { get; set; } = new List<Role>();
    
    // Direkt kullanıcılar (Many-to-Many)
    public virtual ICollection<User> DirectUsers { get; set; } = new List<User>();
}
```

#### **User.cs (Genişletilmiş)**

```csharp
using Microsoft.AspNetCore.Identity;

namespace MiniJira.Domain.Entities;

public class User : IdentityUser
{
    public string FullName { get; set; }
    
    // Kullanıcının rolleri
    public virtual ICollection<Role> Roles { get; set; } = new List<Role>();
    
    // Direkt yetkileri (rol dışında ek yetki)
    public virtual ICollection<Permission> DirectPermissions { get; set; } = new List<Permission>();
}
```

---

## 🔧 **3. PERMISSION SERVICE - BEYNİMİZ**

### **3.1 IPermissionService Interface**

```csharp
namespace MiniJira.Application.Services;

public interface IPermissionService
{
    Task<bool> HasPermissionAsync(int userId, string permissionName);
    Task<List<string>> GetUserPermissionsAsync(int userId);
    Task<bool> IsInRoleAsync(int userId, string roleName);
    Task<List<string>> GetUserRolesAsync(int userId);
}
```

### **3.2 PermissionService Implementation**

```csharp
using Microsoft.EntityFrameworkCore;
using MiniJira.Application.Services;
using MiniJira.Infrastructure.Data;

namespace MiniJira.Infrastructure.Services;

public class PermissionService : IPermissionService
{
    private readonly AppDbContext _db;
    
    public PermissionService(AppDbContext db)
    {
        _db = db;
    }
    
    public async Task<bool> HasPermissionAsync(int userId, string permissionName)
    {
        // 1. Kullanıcıyı rolleri ve yetkileriyle birlikte getir
        var user = await _db.Users
            .Include(u => u.Roles)
                .ThenInclude(r => r.Permissions)
            .Include(u => u.DirectPermissions)
            .FirstOrDefaultAsync(u => u.Id == userId);
        
        if (user == null) return false;
        
        // 2. Tüm yetkileri topla
        var allPermissions = new List<string>();
        
        // Rol yetkileri
        foreach (var role in user.Roles)
        {
            allPermissions.AddRange(role.Permissions.Select(p => p.Name));
        }
        
        // Direkt yetkiler
        allPermissions.AddRange(user.DirectPermissions.Select(p => p.Name));
        
        // 3. İstenen yetki var mı kontrol et
        return allPermissions.Contains(permissionName);
    }
    
    public async Task<List<string>> GetUserPermissionsAsync(int userId)
    {
        var user = await _db.Users
            .Include(u => u.Roles)
                .ThenInclude(r => r.Permissions)
            .Include(u => u.DirectPermissions)
            .FirstOrDefaultAsync(u => u.Id == userId);
        
        if (user == null) return new List<string>();
        
        var permissions = new List<string>();
        
        // Rol yetkileri
        foreach (var role in user.Roles)
        {
            permissions.AddRange(role.Permissions.Select(p => p.Name));
        }
        
        // Direkt yetkiler
        permissions.AddRange(user.DirectPermissions.Select(p => p.Name));
        
        // Tekrarları kaldır
        return permissions.Distinct().ToList();
    }
    
    public async Task<bool> IsInRoleAsync(int userId, string roleName)
    {
        var user = await _db.Users
            .Include(u => u.Roles)
            .FirstOrDefaultAsync(u => u.Id == userId);
        
        if (user == null) return false;
        
        return user.Roles.Any(r => r.Name == roleName);
    }
    
    public async Task<List<string>> GetUserRolesAsync(int userId)
    {
        var user = await _db.Users
            .Include(u => u.Roles)
            .FirstOrDefaultAsync(u => u.Id == userId);
        
        if (user == null) return new List<string>();
        
        return user.Roles.Select(r => r.Name).ToList();
    }
}
```

---

## 🛡️ **4. ATTRIBUTE İLE KOLAY KULLANIM**

### **4.1 AuthorizePermissionAttribute**

```csharp
using Microsoft.AspNetCore.Authorization;
using Microsoft.AspNetCore.Mvc;
using Microsoft.AspNetCore.Mvc.Filters;
using Microsoft.Extensions.DependencyInjection;
using MiniJira.Application.Services;
using System.Security.Claims;

namespace MiniJira.WebAPI.Attributes;

[AttributeUsage(AttributeTargets.Method | AttributeTargets.Class)]
public class AuthorizePermissionAttribute : AuthorizeAttribute, IAuthorizationFilter
{
    private readonly string _permission;
    
    public AuthorizePermissionAttribute(string permission)
    {
        _permission = permission;
    }
    
    public void OnAuthorization(AuthorizationFilterContext context)
    {
        // 1. Kullanıcı giriş yapmış mı?
        if (!context.HttpContext.User.Identity.IsAuthenticated)
        {
            context.Result = new UnauthorizedResult();
            return;
        }
        
        // 2. UserId'yi al
        var userIdClaim = context.HttpContext.User.FindFirstValue(ClaimTypes.NameIdentifier);
        
        if (string.IsNullOrEmpty(userIdClaim) || !int.TryParse(userIdClaim, out int userId))
        {
            context.Result = new UnauthorizedResult();
            return;
        }
        
        // 3. PermissionService'i al
        var permissionService = context.HttpContext.RequestServices
            .GetRequiredService<IPermissionService>();
        
        // 4. Yetki kontrolü yap
        var hasPermission = permissionService
            .HasPermissionAsync(userId, _permission).GetAwaiter().GetResult();
        
        if (!hasPermission)
        {
            context.Result = new ForbidResult();
        }
    }
}
```

---

## 🎮 **5. CONTROLLER'DA KULLANIM - ÇOK BASİT!**

### **5.1 ProjectsController Örneği**

```csharp
using Microsoft.AspNetCore.Authorization;
using Microsoft.AspNetCore.Mvc;
using MiniJira.WebAPI.Attributes;

namespace MiniJira.WebAPI.Controllers;

[ApiController]
[Route("api/[controller]")]
[Authorize] // Önce login olmalı
public class ProjectsController : ControllerBase
{
    [HttpGet]
    [AuthorizePermission("Project.View")] // Sadece bu yetkisi olanlar
    public IActionResult GetProjects()
    {
        // Kod burada
        return Ok(new { message = "Projeler listelendi" });
    }
    
    [HttpPost]
    [AuthorizePermission("Project.Create")] // Sadece proje oluşturma yetkisi olanlar
    public IActionResult CreateProject([FromBody] CreateProjectDto dto)
    {
        // Proje oluşturma kodu
        return Ok(new { message = "Proje oluşturuldu" });
    }
    
    [HttpPut("{id}")]
    [AuthorizePermission("Project.Edit")] // Sadece düzenleme yetkisi olanlar
    public IActionResult UpdateProject(int id, [FromBody] UpdateProjectDto dto)
    {
        // Proje güncelleme kodu
        return Ok(new { message = "Proje güncellendi" });
    }
    
    [HttpDelete("{id}")]
    [AuthorizePermission("Project.Delete")] // Sadece silme yetkisi olanlar
    public IActionResult DeleteProject(int id)
    {
        // Proje silme kodu
        return Ok(new { message = "Proje silindi" });
    }
}
```

### **5.2 TasksController Örneği**

```csharp
using Microsoft.AspNetCore.Authorization;
using Microsoft.AspNetCore.Mvc;
using MiniJira.WebAPI.Attributes;

namespace MiniJira.WebAPI.Controllers;

[ApiController]
[Route("api/[controller]")]
[Authorize]
public class TasksController : ControllerBase
{
    [HttpGet]
    [AuthorizePermission("Task.View")]
    public IActionResult GetTasks()
    {
        return Ok(new { message = "Görevler listelendi" });
    }
    
    [HttpPost]
    [AuthorizePermission("Task.Create")]
    public IActionResult CreateTask([FromBody] CreateTaskDto dto)
    {
        return Ok(new { message = "Görev oluşturuldu" });
    }
    
    [HttpPost("{id}/assign")]
    [AuthorizePermission("Task.Assign")]
    public IActionResult AssignTask(int id, [FromBody] AssignTaskDto dto)
    {
        return Ok(new { message = "Görev atandı" });
    }
}
```

---

## 👨‍💼 **6. ADMIN PANELİ İLE ROL/YETKİ YÖNETİMİ**

### **6.1 Admin RolesController**

```csharp
using Microsoft.AspNetCore.Authorization;
using Microsoft.AspNetCore.Mvc;
using Microsoft.EntityFrameworkCore;
using MiniJira.Application.Services;
using MiniJira.Infrastructure.Data;
using MiniJira.WebAPI.Attributes;

namespace MiniJira.WebAPI.Controllers.Admin;

[Route("api/admin/[controller]")]
[AuthorizePermission("Role.Manage")] // Sadece rol yönetme yetkisi olan admin
public class RolesController : ControllerBase
{
    private readonly AppDbContext _db;
    
    public RolesController(AppDbContext db)
    {
        _db = db;
    }
    
    [HttpGet("permissions")]
    public IActionResult GetAllPermissions()
    {
        // Tüm yetkileri getir (veritabanından)
        var permissions = _db.Permissions
            .Select(p => new
            {
                p.Id,
                p.Name,
                p.Category,
                p.Description
            })
            .ToList();
        
        return Ok(permissions);
    }
    
    [HttpGet]
    public IActionResult GetAllRoles()
    {
        var roles = _db.Roles
            .Include(r => r.Permissions)
            .Select(r => new
            {
                r.Id,
                r.Name,
                r.Description,
                Permissions = r.Permissions.Select(p => p.Name).ToList()
            })
            .ToList();
        
        return Ok(roles);
    }
    
    [HttpGet("{roleId}")]
    public IActionResult GetRole(int roleId)
    {
        var role = _db.Roles
            .Include(r => r.Permissions)
            .FirstOrDefault(r => r.Id == roleId);
        
        if (role == null)
            return NotFound();
        
        return Ok(new
        {
            role.Id,
            role.Name,
            role.Description,
            Permissions = role.Permissions.Select(p => p.Name).ToList()
        });
    }
    
    [HttpPost("{roleId}/permissions")]
    public async Task<IActionResult> UpdateRolePermissions(int roleId, [FromBody] List<string> permissionNames)
    {
        var role = await _db.Roles
            .Include(r => r.Permissions)
            .FirstOrDefaultAsync(r => r.Id == roleId);
        
        if (role == null)
            return NotFound();
        
        // Mevcut yetkileri temizle
        role.Permissions.Clear();
        
        // Yeni yetkileri ekle
        var permissions = await _db.Permissions
            .Where(p => permissionNames.Contains(p.Name))
            .ToListAsync();
        
        role.Permissions = permissions;
        
        await _db.SaveChangesAsync();
        
        return Ok(new { message = "Rol yetkileri güncellendi" });
    }
    
    [HttpPost("{roleId}/users/{userId}")]
    public async Task<IActionResult> AssignRoleToUser(int roleId, int userId)
    {
        var role = await _db.Roles.FindAsync(roleId);
        var user = await _db.Users.FindAsync(userId);
        
        if (role == null || user == null)
            return NotFound();
        
        if (!user.Roles.Contains(role))
        {
            user.Roles.Add(role);
            await _db.SaveChangesAsync();
        }
        
        return Ok(new { message = "Rol kullanıcıya atandı" });
    }
    
    [HttpDelete("{roleId}/users/{userId}")]
    public async Task<IActionResult> RemoveRoleFromUser(int roleId, int userId)
    {
        var role = await _db.Roles.FindAsync(roleId);
        var user = await _db.Users
            .Include(u => u.Roles)
            .FirstOrDefaultAsync(u => u.Id == userId);
        
        if (role == null || user == null)
            return NotFound();
        
        user.Roles.Remove(role);
        await _db.SaveChangesAsync();
        
        return Ok(new { message = "Rol kullanıcıdan kaldırıldı" });
    }
}
```

---

## ⚛️ **7. REACT ADMIN PANELİ (Görsel Yetki Yönetimi)**

### **7.1 AdminRolYonetimi Component**

```typescript
// src/pages/admin/AdminRolYonetimi.tsx
import React, { useState, useEffect } from 'react';
import { Box, Typography, Grid, Card, CardContent, Checkbox, FormControlLabel, Button } from '@mui/material';

interface Permission {
  id: number;
  name: string;
  category: string;
  description: string;
}

interface Role {
  id: number;
  name: string;
  description: string;
  permissions: string[];
}

export default function AdminRolYonetimi() {
  const [roller, setRoller] = useState<Role[]>([]);
  const [yetkiler, setYetkiler] = useState<Permission[]>([]);
  const [seciliRol, setSeciliRol] = useState<Role | null>(null);

  // Yetkileri getir
  useEffect(() => {
    fetch('/api/admin/roles/permissions', {
      headers: {
        'Authorization': `Bearer ${localStorage.getItem('token')}`
      }
    })
      .then(res => res.json())
      .then(data => setYetkiler(data));
  }, []);

  // Rolleri getir
  useEffect(() => {
    fetch('/api/admin/roles', {
      headers: {
        'Authorization': `Bearer ${localStorage.getItem('token')}`
      }
    })
      .then(res => res.json())
      .then(data => setRoller(data));
  }, []);

  // Rol seçildiğinde
  const handleRolSec = (rol: Role) => {
    setSeciliRol(rol);
  };

  // Yetki değişince
  const handleYetkiDegistir = async (yetkiAdi: string, checked: boolean) => {
    if (!seciliRol) return;

    const guncelYetkiler = checked
      ? [...seciliRol.permissions, yetkiAdi]
      : seciliRol.permissions.filter(y => y !== yetkiAdi);

    try {
      const response = await fetch(`/api/admin/roles/${seciliRol.id}/permissions`, {
        method: 'POST',
        headers: {
          'Content-Type': 'application/json',
          'Authorization': `Bearer ${localStorage.getItem('token')}`
        },
        body: JSON.stringify(guncelYetkiler)
      });

      if (response.ok) {
        setSeciliRol({ ...seciliRol, permissions: guncelYetkiler });
        // Rolleri güncelle
        setRoller(roller.map(r => 
          r.id === seciliRol.id 
            ? { ...r, permissions: guncelYetkiler }
            : r
        ));
      }
    } catch (error) {
      console.error('Yetki güncellenirken hata:', error);
    }
  };

  // Kategorilere göre yetkileri grupla
  const yetkilerByCategory = yetkiler.reduce((acc, yetki) => {
    if (!acc[yetki.category]) {
      acc[yetki.category] = [];
    }
    acc[yetki.category].push(yetki);
    return acc;
  }, {} as Record<string, Permission[]>);

  return (
    <Box sx={{ p: 4 }}>
      <Typography variant="h4" gutterBottom>
        Rol ve Yetki Yönetimi
      </Typography>

      <Grid container spacing={3} sx={{ mt: 2 }}>
        {/* Roller Listesi */}
        <Grid item xs={12} md={4}>
          <Card>
            <CardContent>
              <Typography variant="h6" gutterBottom>
                Roller
              </Typography>
              {roller.map(rol => (
                <Card
                  key={rol.id}
                  sx={{
                    p: 2,
                    mb: 2,
                    cursor: 'pointer',
                    border: seciliRol?.id === rol.id ? '2px solid #1976d2' : '1px solid #e0e0e0',
                    '&:hover': { bgcolor: 'action.hover' }
                  }}
                  onClick={() => handleRolSec(rol)}
                >
                  <Typography variant="subtitle1" fontWeight="bold">
                    {rol.name}
                  </Typography>
                  <Typography variant="body2" color="text.secondary">
                    {rol.description}
                  </Typography>
                  <Typography variant="caption" color="text.secondary">
                    {rol.permissions.length} yetki
                  </Typography>
                </Card>
              ))}
            </CardContent>
          </Card>
        </Grid>

        {/* Yetkiler */}
        <Grid item xs={12} md={8}>
          <Card>
            <CardContent>
              <Typography variant="h6" gutterBottom>
                {seciliRol ? `${seciliRol.name} - Yetkiler` : 'Lütfen bir rol seçin'}
              </Typography>

              {seciliRol && Object.entries(yetkilerByCategory).map(([category, categoryYetkiler]) => (
                <Box key={category} sx={{ mb: 3 }}>
                  <Typography variant="subtitle1" fontWeight="bold" sx={{ mb: 1 }}>
                    {category}
                  </Typography>
                  {categoryYetkiler.map(yetki => (
                    <FormControlLabel
                      key={yetki.id}
                      control={
                        <Checkbox
                          checked={seciliRol.permissions.includes(yetki.name)}
                          onChange={(e) => handleYetkiDegistir(yetki.name, e.target.checked)}
                        />
                      }
                      label={
                        <Box>
                          <Typography variant="body2" fontWeight="medium">
                            {yetki.name}
                          </Typography>
                          <Typography variant="caption" color="text.secondary">
                            {yetki.description}
                          </Typography>
                        </Box>
                      }
                      sx={{ display: 'block', mb: 1 }}
                    />
                  ))}
                </Box>
              ))}
            </CardContent>
          </Card>
        </Grid>
      </Grid>
    </Box>
  );
}
```

---

## 🌱 **8. SEED DATA - BAŞLANGIÇ ROL/YETKİLERİ**

### **8.1 SeedData.cs**

```csharp
using Microsoft.EntityFrameworkCore;
using MiniJira.Domain.Entities;
using MiniJira.Infrastructure.Data;

namespace MiniJira.Infrastructure.Data;

public static class SeedData
{
    public static void Initialize(AppDbContext context)
    {
        if (!context.Roles.Any())
        {
            // 1. Yetkileri oluştur
            var permissions = new List<Permission>
            {
                // Proje Yetkileri
                new Permission { Name = "Project.View", Category = "Project", Description = "Projeleri görüntüle" },
                new Permission { Name = "Project.Create", Category = "Project", Description = "Proje oluştur" },
                new Permission { Name = "Project.Edit", Category = "Project", Description = "Proje düzenle" },
                new Permission { Name = "Project.Delete", Category = "Project", Description = "Proje sil" },
                new Permission { Name = "Project.ViewAll", Category = "Project", Description = "Tüm projeleri görüntüle" },
                
                // Görev Yetkileri
                new Permission { Name = "Task.View", Category = "Task", Description = "Görevleri görüntüle" },
                new Permission { Name = "Task.Create", Category = "Task", Description = "Görev oluştur" },
                new Permission { Name = "Task.Edit", Category = "Task", Description = "Görev düzenle" },
                new Permission { Name = "Task.Delete", Category = "Task", Description = "Görev sil" },
                new Permission { Name = "Task.Assign", Category = "Task", Description = "Görev ata" },
                new Permission { Name = "Task.ViewAll", Category = "Task", Description = "Tüm görevleri görüntüle" },
                
                // Kullanıcı Yetkileri
                new Permission { Name = "User.View", Category = "User", Description = "Kullanıcıları görüntüle" },
                new Permission { Name = "User.Edit", Category = "User", Description = "Kullanıcı düzenle" },
                new Permission { Name = "User.Delete", Category = "User", Description = "Kullanıcı sil" },
                new Permission { Name = "Role.Manage", Category = "User", Description = "Rol ve yetki yönetimi" },
            };
            
            context.Permissions.AddRange(permissions);
            context.SaveChanges();
            
            // 2. Rolleri oluştur
            var adminRole = new Role 
            { 
                Name = "Admin",
                Description = "Sistem yöneticisi, tüm yetkilere sahip",
                Permissions = permissions // Admin tüm yetkilere sahip
            };
            
            var pmRole = new Role
            {
                Name = "ProjectManager",
                Description = "Proje yöneticisi",
                Permissions = permissions
                    .Where(p => p.Name.Contains("Project") || 
                               p.Name.Contains("Task") ||
                               p.Name == "User.View")
                    .ToList()
            };
            
            var devRole = new Role
            {
                Name = "Developer",
                Description = "Yazılım geliştirici",
                Permissions = permissions
                    .Where(p => p.Name == "Project.View" ||
                               p.Name == "Task.View" ||
                               p.Name == "Task.Create" ||
                               p.Name == "Task.Edit")
                    .ToList()
            };
            
            context.Roles.AddRange(adminRole, pmRole, devRole);
            context.SaveChanges();
        }
    }
}
```

### **8.2 Program.cs veya Startup.cs'de Çağırma**

```csharp
using MiniJira.Infrastructure.Data;

var app = builder.Build();

// Seed data'yı çalıştır
using (var scope = app.Services.CreateScope())
{
    var context = scope.ServiceProvider.GetRequiredService<AppDbContext>();
    SeedData.Initialize(context);
}

app.Run();
```

---

## ⚡ **9. PERFORMANS İÇİN CACHELEME**

### **9.1 CachedPermissionService**

```csharp
using Microsoft.Extensions.Caching.Memory;
using MiniJira.Application.Services;

namespace MiniJira.Infrastructure.Services;

public class CachedPermissionService : IPermissionService
{
    private readonly IPermissionService _innerService;
    private readonly IMemoryCache _cache;
    private readonly TimeSpan _cacheExpiration = TimeSpan.FromMinutes(5);
    
    public CachedPermissionService(IPermissionService innerService, IMemoryCache cache)
    {
        _innerService = innerService;
        _cache = cache;
    }
    
    public async Task<bool> HasPermissionAsync(int userId, string permissionName)
    {
        var cacheKey = $"permissions:{userId}:{permissionName}";
        
        // Önce cache'ten bak
        if (_cache.TryGetValue(cacheKey, out bool cachedResult))
        {
            return cachedResult;
        }
        
        // Cache'te yoksa asıl servisten al
        var result = await _innerService.HasPermissionAsync(userId, permissionName);
        
        // Cache'le
        _cache.Set(cacheKey, result, _cacheExpiration);
        
        return result;
    }
    
    public async Task<List<string>> GetUserPermissionsAsync(int userId)
    {
        var cacheKey = $"userpermissions:{userId}";
        
        if (_cache.TryGetValue(cacheKey, out List<string> cachedPermissions))
        {
            return cachedPermissions;
        }
        
        var permissions = await _innerService.GetUserPermissionsAsync(userId);
        
        // Kullanıcı yetkilerini 10 dakika cache'le
        _cache.Set(cacheKey, permissions, TimeSpan.FromMinutes(10));
        
        return permissions;
    }
    
    public async Task<bool> IsInRoleAsync(int userId, string roleName)
    {
        var cacheKey = $"userrole:{userId}:{roleName}";
        
        if (_cache.TryGetValue(cacheKey, out bool cachedResult))
        {
            return cachedResult;
        }
        
        var result = await _innerService.IsInRoleAsync(userId, roleName);
        _cache.Set(cacheKey, result, _cacheExpiration);
        
        return result;
    }
    
    public async Task<List<string>> GetUserRolesAsync(int userId)
    {
        var cacheKey = $"userroles:{userId}";
        
        if (_cache.TryGetValue(cacheKey, out List<string> cachedRoles))
        {
            return cachedRoles;
        }
        
        var roles = await _innerService.GetUserRolesAsync(userId);
        _cache.Set(cacheKey, roles, TimeSpan.FromMinutes(10));
        
        return roles;
    }
}
```

### **9.2 Dependency Injection (Program.cs)**

```csharp
using Microsoft.Extensions.Caching.Memory;
using MiniJira.Application.Services;
using MiniJira.Infrastructure.Services;

// Memory cache ekle
builder.Services.AddMemoryCache();

// PermissionService'i kaydet
builder.Services.AddScoped<IPermissionService, PermissionService>();

// Cache wrapper'ı kullanmak istersen (opsiyonel)
// builder.Services.Decorate<IPermissionService, CachedPermissionService>();
```

---

## 🎯 **10. SİSTEM MİMARİSİ - GÖRSEL AÇIKLAMA**

### **10.1 Sistem Akış Diyagramı**

```
┌─────────────────────────────────────────────┐
│           ADMIN PANELİ (React)              │
│  ✅ Proje Görüntüle                         │
│  ✅ Proje Oluştur                           │
│  ☑️ Proje Sil        ← TIKLAYARAK AÇ/KAPA   │
│  ✅ Görev Ata                               │
│  ☑️ Kullanıcı Sil                           │
└─────────────────────────────────────────────┘
            ↓ (API Call)
┌─────────────────────────────────────────────┐
│         .NET BACKEND API                    │
│  [AuthorizePermission("Project.Delete")]    │
│  public IActionResult DeleteProject() {     │
│      // Sadece yetkisi olanlar buraya gelir │
│  }                                          │
└─────────────────────────────────────────────┘
            ↓ (Database Check)
┌─────────────────────────────────────────────┐
│           VERİTABANI                        │
│  ROL: Admin                                 │
│  └─ Yetkiler:                               │
│     • Project.View                          │
│     • Project.Create                        │
│     • Project.Delete   ← BU VAR MI? EVET!   │
│     • Task.Assign                           │
└─────────────────────────────────────────────┘
```

### **10.2 Veri Akışı**

```
1. Kullanıcı Login → JWT Token alır (UserId içerir)
2. Request → [AuthorizePermission("Project.Delete")]
3. Attribute → PermissionService.HasPermissionAsync(userId, "Project.Delete")
4. PermissionService → Database'den kullanıcının rolleri ve yetkilerini getirir
5. Yetki kontrolü → Varsa true, yoksa false
6. true → Request devam eder
7. false → 403 Forbidden döner
```

---

## ✅ **11. AVANTAJLARI**

### **Neden Bu Sistem "WOW" Dedirtir?**

1. **✅ Merkezi Yönetim:** Admin panelinden tıkla, yetki ver/kaldır
2. **✅ Esnek:** Roller + direkt yetkiler (hibrit sistem)
3. **✅ Performanslı:** Cache mekanizması var
4. **✅ Temiz Kod:** Controller'lar sadece `[AuthorizePermission("XXX")]`
5. **✅ Geliştirilebilir:** Yeni yetki eklemek çok kolay
6. **✅ Test Edilebilir:** Mock'laması kolay
7. **✅ Scalable:** Binlerce kullanıcı ve yetkiyi destekler
8. **✅ Industry Standard:** RBAC + Permission-based yaklaşımı

---

## 📝 **12. İŞE ALIM GÖRÜŞMESİNDE ANLATACAKLARIN**

### **Teknik Açıklama:**

> "Ben **RBAC (Role-Based Access Control)** ile **Permission-based Authorization'ı** birleştirdim. 
> - **Roller** var (Admin, PM, Developer)
> - Her rolün **yetkileri** var (Project.Create, Task.Assign gibi)
> - Admin panelinden **tıklayarak yetki atanabiliyor**
> - Backend'de her endpoint `[AuthorizePermission("izin.adı")]` ile korunuyor
> - **Cache mekanizması** ile performans optimizasyonu yaptım
> - **Hibrit sistem:** Hem role-based hem permission-based"

### **CV'ye Yazılacaklar:**

- ✅ **RBAC + Permission-based Auth sistemi geliştirdim**
- ✅ **Merkezi yetki yönetimi ile admin paneli**
- ✅ **.NET Attribute-based authorization**
- ✅ **React + .NET entegrasyonu**
- ✅ **Performance optimization (caching)**
- ✅ **Hibrit yetkilendirme sistemi (Role + Direct Permission)**

---

## 🚀 **13. KURULUM ADIMLARI**

### **Adım 1: Database Tablolarını Oluştur**

```bash
# Entity Framework Migration oluştur
dotnet ef migrations add AddRolePermissionSystem --project Infrastructure --startup-project WebAPI

# Migration'ı uygula
dotnet ef database update --project Infrastructure --startup-project WebAPI
```

### **Adım 2: Seed Data Ekle**

```csharp
// Program.cs veya Startup.cs'de
SeedData.Initialize(context);
```

### **Adım 3: PermissionService'i Kaydet**

```csharp
// Program.cs
builder.Services.AddScoped<IPermissionService, PermissionService>();
builder.Services.AddMemoryCache(); // Cache için
```

### **Adım 4: Controller'lara Attribute Ekle**

```csharp
[AuthorizePermission("Project.Create")]
public IActionResult CreateProject() { ... }
```

### **Adım 5: React Admin Paneli Yap**

```bash
# React component oluştur
# AdminRolYonetimi.tsx dosyasını ekle
```

---

## 🔥 **14. PORTFOLYO İÇİN ÖNEMİ**

Bu sistemle kesinlikle fark edilirsin! Hem temiz kod, hem modern pattern'ler, hem de gerçek dünya problemi çözmüş olursun.

### **Gösterilecek Özellikler:**

1. **Admin Panelinden Yetki Yönetimi:** Canlı demo'da göster
2. **Attribute-based Authorization:** Kod örneği göster
3. **Cache Mekanizması:** Performans testi yap
4. **Hibrit Sistem:** Hem rol hem direkt yetki örneği göster

---

## 📚 **15. EK KAYNAKLAR**

### **Öğrenilecek Kavramlar:**

- **RBAC (Role-Based Access Control)**
- **Permission-based Authorization**
- **Attribute-based Authorization**
- **Caching Strategies**
- **Many-to-Many Relationships (EF Core)**
- **Authorization Filters (ASP.NET Core)**

### **İlgili Dokümantasyon:**

- [ASP.NET Core Authorization](https://learn.microsoft.com/en-us/aspnet/core/security/authorization/)
- [Entity Framework Core Relationships](https://learn.microsoft.com/en-us/ef/core/modeling/relationships)
- [Memory Caching in ASP.NET Core](https://learn.microsoft.com/en-us/aspnet/core/performance/caching/memory)

---

## 🎉 **SONUÇ**

Bu sistem, **production-ready**, **scalable** ve **maintainable** bir yetkilendirme çözümüdür. Modern yazılım geliştirme pratiklerini takip eder ve gerçek dünya problemlerini çözer.

**Başarılar! 🚀**
