
# ASP.NET Core JWT Authentication with Roles, Claims, and Policies
01-11-2024

This guide covers creating JWT authentication in an ASP.NET Core application, including user login, role-based authorization, and custom claims and policies.

---

## 1. Set Up ASP.NET Core Project

1. **Create a new project**:

   ```bash
   dotnet new webapi -o JwtAuthExample
   cd JwtAuthExample
   ```

2. **Install Packages**:

   ```bash
   dotnet add package Microsoft.AspNetCore.Authentication.JwtBearer
   dotnet add package Microsoft.EntityFrameworkCore.Sqlite
   dotnet add package Microsoft.EntityFrameworkCore.Design
   ```

## 2. Configure the Database and Models

1. **Create `User` model**:

   ```csharp
   // Models/User.cs
   public class User
   {
       public int Id { get; set; }
       public string Username { get; set; }
       public string PasswordHash { get; set; }
       public string Role { get; set; } // To store roles
   }
   ```

2. **Set up DbContext**:

   ```csharp
   // Data/AppDbContext.cs
   using Microsoft.EntityFrameworkCore;
   using JwtAuthExample.Models;

   public class AppDbContext : DbContext
   {
       public DbSet<User> Users { get; set; }

       public AppDbContext(DbContextOptions<AppDbContext> options) : base(options) { }
   }
   ```

3. **Register DbContext** in `Program.cs`:

   ```csharp
   builder.Services.AddDbContext<AppDbContext>(options =>
       options.UseSqlite("Data Source=JwtAuthExample.db"));
   ```

4. **Run Migrations**:

   ```bash
   dotnet ef migrations add InitialCreate
   dotnet ef database update
   ```

## 3. JWT Authentication Configuration

1. **Add JWT Authentication Configuration** in `Program.cs`:

   ```csharp
   using Microsoft.IdentityModel.Tokens;
   using System.Text;

   builder.Services.AddAuthentication("Bearer")
       .AddJwtBearer(options =>
       {
           options.TokenValidationParameters = new TokenValidationParameters
           {
               ValidateIssuer = true,
               ValidateAudience = true,
               ValidateLifetime = true,
               ValidateIssuerSigningKey = true,
               ValidIssuer = "YourIssuer",
               ValidAudience = "YourAudience",
               IssuerSigningKey = new SymmetricSecurityKey(Encoding.UTF8.GetBytes("Your_Secret_Key")),
           };
       });

   builder.Services.AddAuthorization(options =>
   {
       options.AddPolicy("AdminOnly", policy => policy.RequireClaim("Role", "Admin"));
   });
   ```

   > **Note**: Replace `"Your_Secret_Key"` with a secure, randomly generated key.

2. **Add Authentication Middleware**:

   ```csharp
   var app = builder.Build();
   app.UseAuthentication();
   app.UseAuthorization();
   ```

## 4. User Registration, Login, and JWT Generation

1. **Password Hashing**:

   ```csharp
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

2. **JWT Token Creation**:

   ```csharp
   using System.IdentityModel.Tokens.Jwt;
   using System.Security.Claims;
   using Microsoft.IdentityModel.Tokens;
   using System.Text;

   public class JwtTokenService
   {
       public string GenerateToken(User user)
       {
           var claims = new[]
           {
               new Claim(ClaimTypes.Name, user.Username),
               new Claim(ClaimTypes.Role, user.Role),
               new Claim("Role", user.Role)
           };

           var key = new SymmetricSecurityKey(Encoding.UTF8.GetBytes("Your_Secret_Key"));
           var creds = new SigningCredentials(key, SecurityAlgorithms.HmacSha256);

           var token = new JwtSecurityToken(
               issuer: "YourIssuer",
               audience: "YourAudience",
               claims: claims,
               expires: DateTime.Now.AddMinutes(30),
               signingCredentials: creds);

           return new JwtSecurityTokenHandler().WriteToken(token);
       }
   }
   ```

3. **Login Action**:

   ```csharp
   [HttpPost("login")]
   public IActionResult Login(string username, string password)
   {
       var user = _context.Users.SingleOrDefault(u => u.Username == username && u.PasswordHash == PasswordHelper.HashPassword(password));
       if (user == null) return Unauthorized();

       var token = _jwtTokenService.GenerateToken(user);
       return Ok(new { token });
   }
   ```

## 5. Role and Claim-based Authorization

1. **Protect Endpoints Based on Roles**:

   ```csharp
   [Authorize(Roles = "Admin")]
   [HttpGet("admin-data")]
   public IActionResult GetAdminData()
   {
       return Ok("This is admin data");
   }
   ```

2. **Policy-Based Authorization**:

   ```csharp
   [Authorize(Policy = "AdminOnly")]
   [HttpGet("policy-protected")]
   public IActionResult GetPolicyProtectedData()
   {
       return Ok("This data requires Admin role");
   }
   ```

3. **Define Policies in `Program.cs`**:

   ```csharp
   builder.Services.AddAuthorization(options =>
   {
       options.AddPolicy("AdminOnly", policy => policy.RequireClaim("Role", "Admin"));
   });
   ```

## 6. Testing the JWT Authentication

Use a tool like **Postman** or **curl** to test authentication:

1. **Login**:

   Send a POST request to `/login` with a valid `username` and `password`.
   
2. **Access Protected Endpoints**:

   For protected endpoints, include the JWT token in the `Authorization` header:

   ```
   Authorization: Bearer <Your_JWT_Token>
   ```

---

## Summary

1. **JWT Setup**: Configure JWT authentication with a secret key and token validation settings.
2. **Claims and Roles**: Use claims to embed role-based information and create authorization policies.
3. **Protect Routes**: Use `[Authorize]` attributes with roles or policies to secure your endpoints.

With this setup, your ASP.NET Core app will handle authentication and authorization using JWT, allowing for flexible role and claim management.
