
# ASP.NET Core Authentication and Authorization from Scratch
Date 01-11-2024 

This guide explains how to create authentication logic in ASP.NET Core without using the Identity scaffolding, covering user registration, login, logout, and role/claim-based authorization.

---

## 1. Set Up the ASP.NET Core Project

1. Create a new ASP.NET Core project:
   ```bash
   dotnet new webapp -o AuthExample
   cd AuthExample
   ```

2. Open the project in Visual Studio or Visual Studio Code.

## 2. Add Database Context and Models

We'll use Entity Framework Core with SQLite.

1. Install necessary packages:
   ```bash
   dotnet add package Microsoft.EntityFrameworkCore.Sqlite
   dotnet add package Microsoft.EntityFrameworkCore.Design
   ```

2. Create a `User` model with properties for user data:

   ```csharp
   // Models/User.cs
   public class User
   {
       public int Id { get; set; }
       public string Username { get; set; }
       public string PasswordHash { get; set; }
       public string Role { get; set; } // Add role for authorization
   }
   ```

3. Set up `AppDbContext`:

   ```csharp
   // Data/AppDbContext.cs
   using Microsoft.EntityFrameworkCore;
   using AuthExample.Models;

   public class AppDbContext : DbContext
   {
       public AppDbContext(DbContextOptions<AppDbContext> options) : base(options) { }
       public DbSet<User> Users { get; set; }
   }
   ```

4. Register the database context in `Program.cs`:

   ```csharp
   // Program.cs
   using AuthExample.Data;
   using Microsoft.EntityFrameworkCore;

   var builder = WebApplication.CreateBuilder(args);

   builder.Services.AddControllersWithViews();
   builder.Services.AddDbContext<AppDbContext>(options =>
       options.UseSqlite("Data Source=AuthExample.db"));

   var app = builder.Build();
   ```

5. Run migrations:

   ```bash
   dotnet ef migrations add InitialCreate
   dotnet ef database update
   ```

## 3. User Registration and Authentication Logic

1. **Password Hashing**:

   ```csharp
   // Helpers/PasswordHelper.cs
   using System.Security.Cryptography;
   using System.Text;

   public static class PasswordHelper
   {
       public static string HashPassword(string password)
       {
           using (var sha256 = SHA256.Create())
           {
               byte[] bytes = sha256.ComputeHash(Encoding.UTF8.GetBytes(password));
               return Convert.ToBase64String(bytes);
           }
       }
   }
   ```

2. **Register Action**:

   ```csharp
   // Controllers/AccountController.cs
   using AuthExample.Data;
   using AuthExample.Models;
   using Microsoft.AspNetCore.Mvc;

   public class AccountController : Controller
   {
       private readonly AppDbContext _context;

       public AccountController(AppDbContext context)
       {
           _context = context;
       }

       [HttpGet]
       public IActionResult Register() => View();

       [HttpPost]
       public async Task<IActionResult> Register(string username, string password, string role)
       {
           var user = new User
           {
               Username = username,
               PasswordHash = PasswordHelper.HashPassword(password),
               Role = role // Set role on registration
           };

           _context.Users.Add(user);
           await _context.SaveChangesAsync();

           return RedirectToAction("Login");
       }
   }
   ```

3. **Login Action with Claims**:

   ```csharp
   using Microsoft.AspNetCore.Authentication;
   using Microsoft.AspNetCore.Authentication.Cookies;
   using System.Security.Claims;

   public class AccountController : Controller
   {
       // Existing code ...

       [HttpGet]
       public IActionResult Login() => View();

       [HttpPost]
       public async Task<IActionResult> Login(string username, string password)
       {
           var passwordHash = PasswordHelper.HashPassword(password);
           var user = _context.Users.FirstOrDefault(u => u.Username == username && u.PasswordHash == passwordHash);

           if (user == null)
           {
               ModelState.AddModelError("", "Invalid username or password.");
               return View();
           }

           var claims = new List<Claim>
           {
               new Claim(ClaimTypes.Name, user.Username),
               new Claim(ClaimTypes.Role, user.Role) // Add user role as a claim
           };

           var claimsIdentity = new ClaimsIdentity(claims, CookieAuthenticationDefaults.AuthenticationScheme);
           var authProperties = new AuthenticationProperties { IsPersistent = true };

           await HttpContext.SignInAsync(CookieAuthenticationDefaults.AuthenticationScheme, new ClaimsPrincipal(claimsIdentity), authProperties);

           return RedirectToAction("Index", "Home");
       }
   }
   ```

## 4. Configure Authentication Middleware

1. Add Authentication Services in `Program.cs`:

   ```csharp
   builder.Services.AddAuthentication(CookieAuthenticationDefaults.AuthenticationScheme)
       .AddCookie(options =>
       {
           options.LoginPath = "/Account/Login";
       });
   ```

2. Enable Authentication Middleware:

   ```csharp
   app.UseAuthentication();
   app.UseAuthorization();
   ```

## 5. Role and Claim-based Authorization

1. Protect routes by specifying roles:

   ```csharp
   using Microsoft.AspNetCore.Authorization;

   [Authorize(Roles = "Admin")]
   public class AdminController : Controller
   {
       public IActionResult Index() => View();
   }
   ```

2. **Authorize based on Claims**: For more granular control, you can authorize actions based on claims instead of roles:

   ```csharp
   [Authorize(Policy = "IsAdmin")]
   public IActionResult SomeAction()
   {
       return View();
   }
   ```

3. **Register Authorization Policies** in `Program.cs`:

   ```csharp
   builder.Services.AddAuthorization(options =>
   {
       options.AddPolicy("IsAdmin", policy => policy.RequireClaim(ClaimTypes.Role, "Admin"));
   });
   ```

4. Add `[Authorize]` to any controller or action to require authentication.

## 6. Logout Functionality

Add a logout action to sign the user out.

```csharp
[HttpPost]
public async Task<IActionResult> Logout()
{
    await HttpContext.SignOutAsync(CookieAuthenticationDefaults.AuthenticationScheme);
    return RedirectToAction("Login", "Account");
}
```

## 7. Create Views

Create basic views for **Register**, **Login**, and **Logout** in `Views/Account`.

```html
<!-- Views/Account/Login.cshtml -->
<form method="post" asp-action="Login">
    <input type="text" name="username" placeholder="Username" required />
    <input type="password" name="password" placeholder="Password" required />
    <button type="submit">Login</button>
</form>
```

---

## Summary

1. **User Registration**: Hash passwords and save user data in the database.
2. **User Login**: Verify the hashed password, then sign in the user with a cookie.
3. **Authentication Middleware**: Use cookies to authenticate users and protect routes.
4. **Role/Claim-based Authorization**: Authorize actions or controllers based on user roles or custom claims.

By following these steps, you’ll have a complete authentication and authorization setup with ASP.NET Core from scratch, allowing for custom roles and claims to secure your application.
