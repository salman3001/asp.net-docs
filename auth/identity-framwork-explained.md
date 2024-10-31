
# Understanding ASP.NET Core Identity Framework
01-11-2024

ASP.NET Core Identity is a membership system that provides login functionality, user management, roles, and claims. This guide delves into how Identity works behind the scenes.

---

## 1. Identity Architecture Overview

The Identity system is built on a layered architecture and includes the following components:

- **Identity Core**: Provides the basic user and role management, including user creation, authentication, and authorization.
- **Stores**: Handle data persistence. Commonly, `UserStore` and `RoleStore` interact with a database.
- **Services**: Support password hashing, token management, and security operations.
- **Entity Framework (EF)**: Used to interact with the database and manage user tables.

Each component follows the dependency injection pattern, making it modular and allowing easy replacement with custom implementations.

## 2. Core Components

1. **User and Role Managers**:
   - `UserManager<TUser>` and `RoleManager<TRole>` manage user and role operations.
   - Each manager uses a corresponding store (`UserStore`, `RoleStore`) to persist data.

2. **SignInManager**:
   - `SignInManager` provides authentication and session management, supporting features like two-factor authentication (2FA) and external logins.

3. **Identity User and Role**:
   - `IdentityUser` and `IdentityRole` are the default classes for users and roles.
   - These classes can be extended by creating custom properties.

4. **Password Hasher and Security Stamp**:
   - Passwords are hashed and stored securely.
   - A security stamp identifies the version of user credentials, ensuring tokens are invalidated on password or security changes.

## 3. Identity Database Schema

The default schema created by ASP.NET Core Identity includes tables like:

- **AspNetUsers**: Stores user information such as username, email, and hashed password.
- **AspNetRoles**: Stores role names.
- **AspNetUserRoles**: Links users with roles.
- **AspNetUserClaims**: Stores user claims for custom data (e.g., permissions).
- **AspNetUserLogins**: Manages external logins for social authentication.

Each table plays a role in managing user identity and permissions.

## 4. How Identity Authentication Works

1. **User Registration**:
   - `UserManager` hashes the password and creates a new entry in the `AspNetUsers` table.

2. **Password Verification**:
   - During login, the `SignInManager` checks the hashed password stored in the database.
   - If correct, an authentication cookie or JWT token is issued to maintain the session.

3. **Claims and Tokens**:
   - When authenticated, the user is assigned claims (e.g., roles, permissions).
   - Claims are embedded in tokens (JWT) or cookies, allowing role and claim-based authorization.

## 5. Role and Claim-based Authorization

1. **Roles**:
   - Roles are predefined groups (e.g., Admin, User) assigned to users via the `UserManager`.
   - ASP.NET Core Identity includes role-based authorization to protect specific endpoints.

2. **Claims**:
   - Claims are key-value pairs that hold information about the user’s identity and permissions.
   - Claims-based authorization offers fine-grained access control by evaluating custom claims (e.g., "Permission", "Department").

3. **Policies**:
   - Policies allow combining roles and claims for complex authorization requirements.
   - Policies are registered in `Program.cs` and referenced in `[Authorize(Policy = "...")]` attributes.

## 6. Extending ASP.NET Core Identity

### Custom User and Role Models

- ASP.NET Core Identity can be customized by extending the `IdentityUser` and `IdentityRole` classes.
- Add custom properties to store additional data, such as `DateOfBirth` or `ProfilePictureUrl`.

Example:

```csharp
public class ApplicationUser : IdentityUser
{
    public DateTime DateOfBirth { get; set; }
}
```

### Custom Stores

- Replace the default `UserStore` and `RoleStore` implementations to use a different data store (e.g., NoSQL database).
- Implement `IUserStore` and `IRoleStore` interfaces to create custom storage providers.

### Custom Authentication and Security Features

- You can add multi-factor authentication, email confirmation, or account lockout policies to enhance security.
- ASP.NET Core Identity supports configuring these options in `Program.cs`.

Example:

```csharp
builder.Services.Configure<IdentityOptions>(options =>
{
    options.Password.RequireDigit = true;
    options.Lockout.MaxFailedAccessAttempts = 5;
});
```

## 7. Authentication Cookies and Tokens

- **Authentication Cookies**: Used for session management in server-rendered applications.
- **JWT Tokens**: Used in APIs to provide stateless authentication.

Identity can generate either a session cookie or a JWT token based on the project’s configuration.

## 8. Dependency Injection and Customization

- The Identity services are injected using dependency injection, allowing custom configurations.
- For example, you can replace `IUserValidator` or `IPasswordHasher` for custom validation or hashing logic.

## Summary

ASP.NET Core Identity provides a robust framework for managing users, roles, and authentication. Its modular design and extensive configurability allow developers to easily customize and extend Identity to meet complex security and authorization requirements.
