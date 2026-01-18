# .NET API Project - GitHub Copilot Instructions (N-tier Architecture)

> **Purpose**: This document provides comprehensive coding standards and best practices for .NET 8+ Web API projects using N-tier architecture. Follow these guidelines to ensure consistent, high-quality code generation across all team members and AI-assisted development tools.

---

## 📋 Table of Contents
1. [Project Structure](#project-structure)
2. [N-tier Architecture](#n-tier-architecture)
3. [Naming Conventions](#naming-conventions)
4. [Entity Framework Core](#entity-framework-core)
5. [JWT Authentication](#jwt-authentication)
6. [Dependency Injection](#dependency-injection)
7. [LINQ Best Practices](#linq-best-practices)
8. [API Controllers](#api-controllers)
9. [Error Handling](#error-handling)
10. [Logging](#logging)
11. [Testing with XUnit](#testing-with-xunit)
12. [Do's and Don'ts](#dos-and-donts)

---

## 📁 Project Structure

### Standard N-tier Solution Structure
```
YourProject.sln
│
├── src/
│   │
│   ├── YourProject.API/                    # Presentation Layer (Web API)
│   │   ├── Controllers/
│   │   ├── Filters/
│   │   ├── Middleware/
│   │   ├── Extensions/
│   │   ├── Program.cs
│   │   ├── appsettings.json
│   │   └── appsettings.Development.json
│   │
│   ├── YourProject.Core/                   # Business Logic Layer (BLL)
│   │   ├── DTOs/                           # Data Transfer Objects
│   │   │   ├── Requests/
│   │   │   └── Responses/
│   │   ├── Interfaces/                     # Service & Repository Interfaces
│   │   │   ├── IServices/
│   │   │   └── IRepositories/
│   │   ├── Services/                       # Business Logic Implementations
│   │   ├── Exceptions/                     # Custom Exceptions
│   │   ├── Helpers/                        # Utility Classes
│   │   └── Constants/                      # Application Constants
│   │
│   └── YourProject.Infrastructure/         # Data Access Layer (DAL)
│       ├── Data/
│       │   ├── ApplicationDbContext.cs
│       │   ├── DbInitializer.cs
│       │   └── Configurations/             # EF Core Entity Configurations
│       ├── Entities/                       # Database Models/Entities
│       ├── Repositories/                   # Repository Implementations
│       ├── Migrations/                     # EF Core Migrations
│       └── Identity/                       # Identity & JWT
│           ├── ApplicationUser.cs
│           └── JwtTokenGenerator.cs
│
└── tests/
    ├── YourProject.UnitTests/              # Unit Tests
    │   ├── Services/
    │   ├── Repositories/
    │   └── Controllers/
    │
    └── YourProject.IntegrationTests/       # Integration Tests
        └── API/
```

### Layer Dependencies (Critical)
```
┌─────────────────────┐
│   API (Web Layer)   │  ← Entry Point
└──────────┬──────────┘
           │ references
           ↓
┌─────────────────────┐
│  Core (BLL)         │  ← Business Logic
└──────────┬──────────┘
           │ references
           ↓
┌─────────────────────┐
│ Infrastructure (DAL)│  ← Data Access
└─────────────────────┘
```

**Dependency Rules:**
- ✅ **API** references **Core** and **Infrastructure**
- ✅ **Core** references **Infrastructure** (for interfaces only)
- ✅ **Infrastructure** implements interfaces from **Core**
- ❌ **Never** let Infrastructure reference API
- ❌ **Never** create circular dependencies

---

## 🏛️ N-tier Architecture

### Layer Responsibilities

#### 1. **API Layer (Presentation)**
**Purpose**: Handle HTTP requests/responses, routing, authentication, and validation

**Contains:**
- Controllers
- Middleware (Exception handling, Logging, CORS)
- Filters (Authorization, Validation)
- Request/Response models
- Dependency injection configuration

**Example Structure:**
```csharp
// Controllers/UsersController.cs
[ApiController]
[Route("api/[controller]")]
[Authorize]
public class UsersController : ControllerBase
{
    private readonly IUserService _userService;
    private readonly ILogger<UsersController> _logger;

    public UsersController(IUserService userService, ILogger<UsersController> logger)
    {
        _userService = userService;
        _logger = logger;
    }

    [HttpGet]
    [ProducesResponseType(typeof(IEnumerable<UserResponseDto>), StatusCodes.Status200OK)]
    public async Task<IActionResult> GetAllUsers(CancellationToken cancellationToken)
    {
        var users = await _userService.GetAllUsersAsync(cancellationToken);
        return Ok(users);
    }

    [HttpGet("{id:guid}")]
    [ProducesResponseType(typeof(UserResponseDto), StatusCodes.Status200OK)]
    [ProducesResponseType(StatusCodes.Status404NotFound)]
    public async Task<IActionResult> GetUserById(Guid id, CancellationToken cancellationToken)
    {
        var user = await _userService.GetUserByIdAsync(id, cancellationToken);
        
        if (user == null)
            return NotFound(new { message = $"User with ID {id} not found" });
        
        return Ok(user);
    }

    [HttpPost]
    [ProducesResponseType(typeof(UserResponseDto), StatusCodes.Status201Created)]
    [ProducesResponseType(StatusCodes.Status400BadRequest)]
    public async Task<IActionResult> CreateUser(
        [FromBody] CreateUserRequestDto request,
        CancellationToken cancellationToken)
    {
        if (!ModelState.IsValid)
            return BadRequest(ModelState);

        var user = await _userService.CreateUserAsync(request, cancellationToken);
        
        return CreatedAtAction(
            nameof(GetUserById),
            new { id = user.Id },
            user);
    }

    [HttpPut("{id:guid}")]
    [ProducesResponseType(StatusCodes.Status204NoContent)]
    [ProducesResponseType(StatusCodes.Status404NotFound)]
    public async Task<IActionResult> UpdateUser(
        Guid id,
        [FromBody] UpdateUserRequestDto request,
        CancellationToken cancellationToken)
    {
        if (!ModelState.IsValid)
            return BadRequest(ModelState);

        var result = await _userService.UpdateUserAsync(id, request, cancellationToken);
        
        if (!result)
            return NotFound(new { message = $"User with ID {id} not found" });
        
        return NoContent();
    }

    [HttpDelete("{id:guid}")]
    [ProducesResponseType(StatusCodes.Status204NoContent)]
    [ProducesResponseType(StatusCodes.Status404NotFound)]
    public async Task<IActionResult> DeleteUser(Guid id, CancellationToken cancellationToken)
    {
        var result = await _userService.DeleteUserAsync(id, cancellationToken);
        
        if (!result)
            return NotFound(new { message = $"User with ID {id} not found" });
        
        return NoContent();
    }
}
```

#### 2. **Core Layer (Business Logic)**
**Purpose**: Business rules, validation, and orchestration

**Contains:**
- Service interfaces and implementations
- DTOs (Data Transfer Objects)
- Business logic validation
- Domain exceptions
- Mappers/Helpers

**Example Structure:**
```csharp
// Interfaces/IServices/IUserService.cs
public interface IUserService
{
    Task<IEnumerable<UserResponseDto>> GetAllUsersAsync(CancellationToken cancellationToken = default);
    Task<UserResponseDto?> GetUserByIdAsync(Guid id, CancellationToken cancellationToken = default);
    Task<UserResponseDto> CreateUserAsync(CreateUserRequestDto request, CancellationToken cancellationToken = default);
    Task<bool> UpdateUserAsync(Guid id, UpdateUserRequestDto request, CancellationToken cancellationToken = default);
    Task<bool> DeleteUserAsync(Guid id, CancellationToken cancellationToken = default);
    Task<PagedResult<UserResponseDto>> GetUsersPagedAsync(int pageNumber, int pageSize, CancellationToken cancellationToken = default);
}

// Services/UserService.cs
public class UserService : IUserService
{
    private readonly IUserRepository _userRepository;
    private readonly ILogger<UserService> _logger;

    public UserService(IUserRepository userRepository, ILogger<UserService> logger)
    {
        _userRepository = userRepository;
        _logger = logger;
    }

    public async Task<IEnumerable<UserResponseDto>> GetAllUsersAsync(
        CancellationToken cancellationToken = default)
    {
        _logger.LogInformation("Retrieving all users");

        var users = await _userRepository.GetAllAsync(cancellationToken);
        
        return users.Select(u => new UserResponseDto
        {
            Id = u.Id,
            Email = u.Email,
            FirstName = u.FirstName,
            LastName = u.LastName,
            FullName = $"{u.FirstName} {u.LastName}",
            CreatedAt = u.CreatedAt,
            IsActive = u.IsActive
        });
    }

    public async Task<UserResponseDto?> GetUserByIdAsync(
        Guid id,
        CancellationToken cancellationToken = default)
    {
        _logger.LogInformation("Retrieving user with ID: {UserId}", id);

        var user = await _userRepository.GetByIdAsync(id, cancellationToken);
        
        if (user == null)
        {
            _logger.LogWarning("User with ID: {UserId} not found", id);
            return null;
        }

        return new UserResponseDto
        {
            Id = user.Id,
            Email = user.Email,
            FirstName = user.FirstName,
            LastName = user.LastName,
            FullName = $"{user.FirstName} {user.LastName}",
            CreatedAt = user.CreatedAt,
            IsActive = user.IsActive
        };
    }

    public async Task<UserResponseDto> CreateUserAsync(
        CreateUserRequestDto request,
        CancellationToken cancellationToken = default)
    {
        _logger.LogInformation("Creating new user with email: {Email}", request.Email);

        // Business validation
        var existingUser = await _userRepository.GetByEmailAsync(request.Email, cancellationToken);
        if (existingUser != null)
        {
            throw new BusinessException($"User with email {request.Email} already exists");
        }

        // Create entity
        var user = new User
        {
            Id = Guid.NewGuid(),
            Email = request.Email,
            FirstName = request.FirstName,
            LastName = request.LastName,
            PasswordHash = HashPassword(request.Password), // Use proper password hashing
            CreatedAt = DateTime.UtcNow,
            IsActive = true
        };

        await _userRepository.AddAsync(user, cancellationToken);

        _logger.LogInformation("User created successfully with ID: {UserId}", user.Id);

        return new UserResponseDto
        {
            Id = user.Id,
            Email = user.Email,
            FirstName = user.FirstName,
            LastName = user.LastName,
            FullName = $"{user.FirstName} {user.LastName}",
            CreatedAt = user.CreatedAt,
            IsActive = user.IsActive
        };
    }

    public async Task<bool> UpdateUserAsync(
        Guid id,
        UpdateUserRequestDto request,
        CancellationToken cancellationToken = default)
    {
        _logger.LogInformation("Updating user with ID: {UserId}", id);

        var user = await _userRepository.GetByIdAsync(id, cancellationToken);
        
        if (user == null)
        {
            _logger.LogWarning("User with ID: {UserId} not found", id);
            return false;
        }

        // Update properties
        user.FirstName = request.FirstName;
        user.LastName = request.LastName;
        user.UpdatedAt = DateTime.UtcNow;

        await _userRepository.UpdateAsync(user, cancellationToken);

        _logger.LogInformation("User with ID: {UserId} updated successfully", id);

        return true;
    }

    public async Task<bool> DeleteUserAsync(
        Guid id,
        CancellationToken cancellationToken = default)
    {
        _logger.LogInformation("Deleting user with ID: {UserId}", id);

        var user = await _userRepository.GetByIdAsync(id, cancellationToken);
        
        if (user == null)
        {
            _logger.LogWarning("User with ID: {UserId} not found", id);
            return false;
        }

        // Soft delete
        user.IsActive = false;
        user.DeletedAt = DateTime.UtcNow;

        await _userRepository.UpdateAsync(user, cancellationToken);

        _logger.LogInformation("User with ID: {UserId} deleted successfully", id);

        return true;
    }

    public async Task<PagedResult<UserResponseDto>> GetUsersPagedAsync(
        int pageNumber,
        int pageSize,
        CancellationToken cancellationToken = default)
    {
        _logger.LogInformation(
            "Retrieving paged users - Page: {PageNumber}, Size: {PageSize}",
            pageNumber,
            pageSize);

        var pagedUsers = await _userRepository.GetPagedAsync(
            pageNumber,
            pageSize,
            cancellationToken);

        var userDtos = pagedUsers.Items.Select(u => new UserResponseDto
        {
            Id = u.Id,
            Email = u.Email,
            FirstName = u.FirstName,
            LastName = u.LastName,
            FullName = $"{u.FirstName} {u.LastName}",
            CreatedAt = u.CreatedAt,
            IsActive = u.IsActive
        }).ToList();

        return new PagedResult<UserResponseDto>(
            userDtos,
            pagedUsers.TotalCount,
            pagedUsers.PageNumber,
            pagedUsers.PageSize);
    }

    private string HashPassword(string password)
    {
        // Implement proper password hashing (e.g., BCrypt, PBKDF2)
        // This is just a placeholder
        return BCrypt.Net.BCrypt.HashPassword(password);
    }
}
```

#### 3. **Infrastructure Layer (Data Access)**
**Purpose**: Database operations, external services, and data persistence

**Contains:**
- DbContext
- Entity configurations
- Repository implementations
- Database entities
- Migrations

**Example Structure:**
```csharp
// Entities/User.cs
public class User
{
    public Guid Id { get; set; }
    public string Email { get; set; } = string.Empty;
    public string FirstName { get; set; } = string.Empty;
    public string LastName { get; set; } = string.Empty;
    public string PasswordHash { get; set; } = string.Empty;
    public bool IsActive { get; set; }
    public DateTime CreatedAt { get; set; }
    public DateTime? UpdatedAt { get; set; }
    public DateTime? DeletedAt { get; set; }
}

// Interfaces/IRepositories/IUserRepository.cs (in Core layer)
public interface IUserRepository
{
    Task<IEnumerable<User>> GetAllAsync(CancellationToken cancellationToken = default);
    Task<User?> GetByIdAsync(Guid id, CancellationToken cancellationToken = default);
    Task<User?> GetByEmailAsync(string email, CancellationToken cancellationToken = default);
    Task<User> AddAsync(User user, CancellationToken cancellationToken = default);
    Task UpdateAsync(User user, CancellationToken cancellationToken = default);
    Task DeleteAsync(Guid id, CancellationToken cancellationToken = default);
    Task<PagedResult<User>> GetPagedAsync(int pageNumber, int pageSize, CancellationToken cancellationToken = default);
    Task<bool> ExistsAsync(Guid id, CancellationToken cancellationToken = default);
}

// Repositories/UserRepository.cs (in Infrastructure layer)
public class UserRepository : IUserRepository
{
    private readonly ApplicationDbContext _context;

    public UserRepository(ApplicationDbContext context)
    {
        _context = context;
    }

    public async Task<IEnumerable<User>> GetAllAsync(
        CancellationToken cancellationToken = default)
    {
        return await _context.Users
            .AsNoTracking()
            .Where(u => u.IsActive)
            .OrderByDescending(u => u.CreatedAt)
            .ToListAsync(cancellationToken);
    }

    public async Task<User?> GetByIdAsync(
        Guid id,
        CancellationToken cancellationToken = default)
    {
        return await _context.Users
            .AsNoTracking()
            .FirstOrDefaultAsync(u => u.Id == id, cancellationToken);
    }

    public async Task<User?> GetByEmailAsync(
        string email,
        CancellationToken cancellationToken = default)
    {
        return await _context.Users
            .AsNoTracking()
            .FirstOrDefaultAsync(u => u.Email == email, cancellationToken);
    }

    public async Task<User> AddAsync(
        User user,
        CancellationToken cancellationToken = default)
    {
        await _context.Users.AddAsync(user, cancellationToken);
        await _context.SaveChangesAsync(cancellationToken);
        return user;
    }

    public async Task UpdateAsync(
        User user,
        CancellationToken cancellationToken = default)
    {
        _context.Users.Update(user);
        await _context.SaveChangesAsync(cancellationToken);
    }

    public async Task DeleteAsync(
        Guid id,
        CancellationToken cancellationToken = default)
    {
        var user = await GetByIdAsync(id, cancellationToken);
        if (user != null)
        {
            _context.Users.Remove(user);
            await _context.SaveChangesAsync(cancellationToken);
        }
    }

    public async Task<PagedResult<User>> GetPagedAsync(
        int pageNumber,
        int pageSize,
        CancellationToken cancellationToken = default)
    {
        var query = _context.Users
            .AsNoTracking()
            .Where(u => u.IsActive);

        var totalCount = await query.CountAsync(cancellationToken);

        var items = await query
            .OrderByDescending(u => u.CreatedAt)
            .Skip((pageNumber - 1) * pageSize)
            .Take(pageSize)
            .ToListAsync(cancellationToken);

        return new PagedResult<User>(items, totalCount, pageNumber, pageSize);
    }

    public async Task<bool> ExistsAsync(
        Guid id,
        CancellationToken cancellationToken = default)
    {
        return await _context.Users
            .AsNoTracking()
            .AnyAsync(u => u.Id == id, cancellationToken);
    }
}
```

---

## 🏷️ Naming Conventions

### General Rules
| Element | Convention | Example | Notes |
|---------|------------|---------|-------|
| **Namespaces** | PascalCase | `YourProject.Core.Services` | Match folder structure |
| **Classes** | PascalCase | `UserService`, `UserRepository` | Descriptive, singular nouns |
| **Interfaces** | IPascalCase | `IUserService`, `IUserRepository` | Prefix with 'I' |
| **Methods** | PascalCase | `GetUserByIdAsync`, `CreateUser` | Verbs or verb phrases |
| **Properties** | PascalCase | `FirstName`, `CreatedAt` | Nouns or noun phrases |
| **Private Fields** | _camelCase | `_userRepository`, `_logger` | Prefix with underscore |
| **Parameters** | camelCase | `userId`, `pageSize` | Descriptive, no abbreviations |
| **Local Variables** | camelCase | `user`, `totalCount` | Short but meaningful |
| **Constants** | PascalCase | `MaxPageSize`, `DefaultTimeout` | All words capitalized |
| **Enums** | PascalCase | `UserRole`, `OrderStatus` | Singular noun |
| **Enum Values** | PascalCase | `Active`, `Pending` | Singular adjectives |
| **Async Methods** | Suffix 'Async' | `GetUserAsync`, `SaveChangesAsync` | Always suffix with 'Async' |

### File Naming Conventions
| Type | Pattern | Example |
|------|---------|---------|
| Controller | `{Entity}Controller.cs` | `UsersController.cs` |
| Service Interface | `I{Entity}Service.cs` | `IUserService.cs` |
| Service Implementation | `{Entity}Service.cs` | `UserService.cs` |
| Repository Interface | `I{Entity}Repository.cs` | `IUserRepository.cs` |
| Repository Implementation | `{Entity}Repository.cs` | `UserRepository.cs` |
| Entity/Model | `{Entity}.cs` | `User.cs` |
| DTO (Request) | `{Action}{Entity}RequestDto.cs` | `CreateUserRequestDto.cs` |
| DTO (Response) | `{Entity}ResponseDto.cs` | `UserResponseDto.cs` |
| Exception | `{Name}Exception.cs` | `BusinessException.cs` |
| Middleware | `{Purpose}Middleware.cs` | `ExceptionHandlingMiddleware.cs` |

### DTOs Naming Pattern
```csharp
// Request DTOs - Action + Entity + Request + Dto
public class CreateUserRequestDto { }
public class UpdateUserRequestDto { }
public class LoginRequestDto { }

// Response DTOs - Entity + Response + Dto
public class UserResponseDto { }
public class OrderResponseDto { }

// List/Paged DTOs
public class PagedResult<T> { }
public class UserListResponseDto { }
```

---

## 🗄️ Entity Framework Core

### DbContext Configuration

```csharp
// Data/ApplicationDbContext.cs
public class ApplicationDbContext : DbContext
{
    public ApplicationDbContext(DbContextOptions<ApplicationDbContext> options)
        : base(options)
    {
    }

    // DbSets
    public DbSet<User> Users { get; set; }
    public DbSet<Product> Products { get; set; }
    public DbSet<Order> Orders { get; set; }

    protected override void OnModelCreating(ModelBuilder modelBuilder)
    {
        base.OnModelCreating(modelBuilder);

        // Apply configurations from separate files
        modelBuilder.ApplyConfigurationsFromAssembly(typeof(ApplicationDbContext).Assembly);

        // Or apply individually
        // modelBuilder.ApplyConfiguration(new UserConfiguration());
    }

    // Override SaveChangesAsync for audit fields
    public override async Task<int> SaveChangesAsync(
        CancellationToken cancellationToken = default)
    {
        var entries = ChangeTracker.Entries()
            .Where(e => e.State == EntityState.Added || e.State == EntityState.Modified);

        foreach (var entry in entries)
        {
            if (entry.State == EntityState.Added)
            {
                if (entry.Entity is IAuditable auditableEntity)
                {
                    auditableEntity.CreatedAt = DateTime.UtcNow;
                }
            }
            else if (entry.State == EntityState.Modified)
            {
                if (entry.Entity is IAuditable auditableEntity)
                {
                    auditableEntity.UpdatedAt = DateTime.UtcNow;
                }
            }
        }

        return await base.SaveChangesAsync(cancellationToken);
    }
}
```

### Entity Configuration (Fluent API)

```csharp
// Data/Configurations/UserConfiguration.cs
public class UserConfiguration : IEntityTypeConfiguration<User>
{
    public void Configure(EntityTypeBuilder<User> builder)
    {
        // Table name
        builder.ToTable("Users");

        // Primary key
        builder.HasKey(u => u.Id);

        // Properties
        builder.Property(u => u.Email)
            .IsRequired()
            .HasMaxLength(255);

        builder.Property(u => u.FirstName)
            .IsRequired()
            .HasMaxLength(100);

        builder.Property(u => u.LastName)
            .IsRequired()
            .HasMaxLength(100);

        builder.Property(u => u.PasswordHash)
            .IsRequired()
            .HasMaxLength(500);

        builder.Property(u => u.CreatedAt)
            .IsRequired()
            .HasDefaultValueSql("GETUTCDATE()");

        builder.Property(u => u.IsActive)
            .IsRequired()
            .HasDefaultValue(true);

        // Indexes
        builder.HasIndex(u => u.Email)
            .IsUnique();

        // Query filters (global filter for soft delete)
        builder.HasQueryFilter(u => u.DeletedAt == null);
    }
}
```

### Entity Base Classes

```csharp
// Entities/Base/IAuditable.cs
public interface IAuditable
{
    DateTime CreatedAt { get; set; }
    DateTime? UpdatedAt { get; set; }
}

// Entities/Base/BaseEntity.cs
public abstract class BaseEntity : IAuditable
{
    public Guid Id { get; set; }
    public DateTime CreatedAt { get; set; }
    public DateTime? UpdatedAt { get; set; }
    public DateTime? DeletedAt { get; set; }
    public bool IsActive { get; set; } = true;
}

// Entities/User.cs
public class User : BaseEntity
{
    public string Email { get; set; } = string.Empty;
    public string FirstName { get; set; } = string.Empty;
    public string LastName { get; set; } = string.Empty;
    public string PasswordHash { get; set; } = string.Empty;
    public string? RefreshToken { get; set; }
    public DateTime? RefreshTokenExpiryTime { get; set; }
    
    // Navigation properties
    public virtual ICollection<Order> Orders { get; set; } = new List<Order>();
}
```

### Generic Repository Pattern

```csharp
// Core/Interfaces/IRepositories/IGenericRepository.cs
public interface IGenericRepository<T> where T : class
{
    Task<IEnumerable<T>> GetAllAsync(CancellationToken cancellationToken = default);
    Task<T?> GetByIdAsync(Guid id, CancellationToken cancellationToken = default);
    Task<T> AddAsync(T entity, CancellationToken cancellationToken = default);
    Task UpdateAsync(T entity, CancellationToken cancellationToken = default);
    Task DeleteAsync(Guid id, CancellationToken cancellationToken = default);
    Task<bool> ExistsAsync(Guid id, CancellationToken cancellationToken = default);
    Task<int> CountAsync(CancellationToken cancellationToken = default);
}

// Infrastructure/Repositories/GenericRepository.cs
public class GenericRepository<T> : IGenericRepository<T> where T : BaseEntity
{
    protected readonly ApplicationDbContext _context;
    protected readonly DbSet<T> _dbSet;

    public GenericRepository(ApplicationDbContext context)
    {
        _context = context;
        _dbSet = context.Set<T>();
    }

    public virtual async Task<IEnumerable<T>> GetAllAsync(
        CancellationToken cancellationToken = default)
    {
        return await _dbSet
            .AsNoTracking()
            .Where(e => e.IsActive)
            .ToListAsync(cancellationToken);
    }

    public virtual async Task<T?> GetByIdAsync(
        Guid id,
        CancellationToken cancellationToken = default)
    {
        return await _dbSet
            .AsNoTracking()
            .FirstOrDefaultAsync(e => e.Id == id, cancellationToken);
    }

    public virtual async Task<T> AddAsync(
        T entity,
        CancellationToken cancellationToken = default)
    {
        await _dbSet.AddAsync(entity, cancellationToken);
        await _context.SaveChangesAsync(cancellationToken);
        return entity;
    }

    public virtual async Task UpdateAsync(
        T entity,
        CancellationToken cancellationToken = default)
    {
        _dbSet.Update(entity);
        await _context.SaveChangesAsync(cancellationToken);
    }

    public virtual async Task DeleteAsync(
        Guid id,
        CancellationToken cancellationToken = default)
    {
        var entity = await GetByIdAsync(id, cancellationToken);
        if (entity != null)
        {
            // Soft delete
            entity.IsActive = false;
            entity.DeletedAt = DateTime.UtcNow;
            await UpdateAsync(entity, cancellationToken);
        }
    }

    public virtual async Task<bool> ExistsAsync(
        Guid id,
        CancellationToken cancellationToken = default)
    {
        return await _dbSet
            .AsNoTracking()
            .AnyAsync(e => e.Id == id, cancellationToken);
    }

    public virtual async Task<int> CountAsync(
        CancellationToken cancellationToken = default)
    {
        return await _dbSet
            .AsNoTracking()
            .Where(e => e.IsActive)
            .CountAsync(cancellationToken);
    }
}
```

### Migration Commands
```bash
# Add migration
dotnet ef migrations add InitialCreate --project YourProject.Infrastructure --startup-project YourProject.API

# Update database
dotnet ef database update --project YourProject.Infrastructure --startup-project YourProject.API

# Remove last migration
dotnet ef migrations remove --project YourProject.Infrastructure --startup-project YourProject.API

# Generate SQL script
dotnet ef migrations script --project YourProject.Infrastructure --startup-project YourProject.API --output migration.sql
```

---

# .NET API Copilot Instructions - Part 2
## (Continue from Part 1)

## 🔐 JWT Authentication & Authorization

### JWT Settings Configuration

```csharp
// Core/Constants/JwtSettings.cs
public class JwtSettings
{
    public const string SectionName = "JwtSettings";
    
    public string Secret { get; set; } = string.Empty;
    public string Issuer { get; set; } = string.Empty;
    public string Audience { get; set; } = string.Empty;
    public int ExpiryMinutes { get; set; }
    public int RefreshTokenExpiryDays { get; set; }
}
```

### JWT Token Generator

```csharp
// Infrastructure/Identity/IJwtTokenGenerator.cs (in Core/Interfaces)
public interface IJwtTokenGenerator
{
    string GenerateToken(User user, IList<string>? roles = null);
    string GenerateRefreshToken();
    ClaimsPrincipal? GetPrincipalFromExpiredToken(string token);
}

// Infrastructure/Identity/JwtTokenGenerator.cs
public class JwtTokenGenerator : IJwtTokenGenerator
{
    private readonly JwtSettings _jwtSettings;
    private readonly ILogger<JwtTokenGenerator> _logger;

    public JwtTokenGenerator(
        IOptions<JwtSettings> jwtSettings,
        ILogger<JwtTokenGenerator> logger)
    {
        _jwtSettings = jwtSettings.Value;
        _logger = logger;
    }

    public string GenerateToken(User user, IList<string>? roles = null)
    {
        var securityKey = new SymmetricSecurityKey(
            Encoding.UTF8.GetBytes(_jwtSettings.Secret));
        
        var credentials = new SigningCredentials(
            securityKey,
            SecurityAlgorithms.HmacSha256);

        var claims = new List<Claim>
        {
            new(JwtRegisteredClaimNames.Sub, user.Id.ToString()),
            new(JwtRegisteredClaimNames.Email, user.Email),
            new(JwtRegisteredClaimNames.Jti, Guid.NewGuid().ToString()),
            new(JwtRegisteredClaimNames.Iat, DateTimeOffset.UtcNow.ToUnixTimeSeconds().ToString()),
            new(ClaimTypes.NameIdentifier, user.Id.ToString()),
            new(ClaimTypes.Email, user.Email),
            new(ClaimTypes.Name, $"{user.FirstName} {user.LastName}"),
            new("FirstName", user.FirstName),
            new("LastName", user.LastName)
        };

        // Add role claims if provided
        if (roles != null && roles.Any())
        {
            claims.AddRange(roles.Select(role => new Claim(ClaimTypes.Role, role)));
        }

        var token = new JwtSecurityToken(
            issuer: _jwtSettings.Issuer,
            audience: _jwtSettings.Audience,
            claims: claims,
            expires: DateTime.UtcNow.AddMinutes(_jwtSettings.ExpiryMinutes),
            signingCredentials: credentials
        );

        var tokenString = new JwtSecurityTokenHandler().WriteToken(token);
        
        _logger.LogInformation("JWT token generated for user: {UserId}", user.Id);
        
        return tokenString;
    }

    public string GenerateRefreshToken()
    {
        var randomNumber = new byte[64];
        using var rng = RandomNumberGenerator.Create();
        rng.GetBytes(randomNumber);
        return Convert.ToBase64String(randomNumber);
    }

    public ClaimsPrincipal? GetPrincipalFromExpiredToken(string token)
    {
        var tokenValidationParameters = new TokenValidationParameters
        {
            ValidateAudience = true,
            ValidateIssuer = true,
            ValidateIssuerSigningKey = true,
            ValidIssuer = _jwtSettings.Issuer,
            ValidAudience = _jwtSettings.Audience,
            IssuerSigningKey = new SymmetricSecurityKey(
                Encoding.UTF8.GetBytes(_jwtSettings.Secret)),
            ValidateLifetime = false // We're validating an expired token
        };

        try
        {
            var tokenHandler = new JwtSecurityTokenHandler();
            var principal = tokenHandler.ValidateToken(
                token,
                tokenValidationParameters,
                out var securityToken);

            if (securityToken is not JwtSecurityToken jwtSecurityToken ||
                !jwtSecurityToken.Header.Alg.Equals(
                    SecurityAlgorithms.HmacSha256,
                    StringComparison.InvariantCultureIgnoreCase))
            {
                _logger.LogWarning("Invalid token algorithm");
                return null;
            }

            return principal;
        }
        catch (Exception ex)
        {
            _logger.LogError(ex, "Error validating expired token");
            return null;
        }
    }
}
```

### Program.cs - JWT Configuration

```csharp
// Program.cs

var builder = WebApplication.CreateBuilder(args);

// Add Configuration
builder.Services.Configure<JwtSettings>(
    builder.Configuration.GetSection(JwtSettings.SectionName));

// Add Authentication
var jwtSettings = builder.Configuration
    .GetSection(JwtSettings.SectionName)
    .Get<JwtSettings>()
    ?? throw new InvalidOperationException("JWT settings not configured");

builder.Services.AddAuthentication(options =>
{
    options.DefaultAuthenticateScheme = JwtBearerDefaults.AuthenticationScheme;
    options.DefaultChallengeScheme = JwtBearerDefaults.AuthenticationScheme;
    options.DefaultScheme = JwtBearerDefaults.AuthenticationScheme;
})
.AddJwtBearer(options =>
{
    options.SaveToken = true;
    options.RequireHttpsMetadata = false; // Set to true in production
    options.TokenValidationParameters = new TokenValidationParameters
    {
        ValidateIssuer = true,
        ValidateAudience = true,
        ValidateLifetime = true,
        ValidateIssuerSigningKey = true,
        ValidIssuer = jwtSettings.Issuer,
        ValidAudience = jwtSettings.Audience,
        IssuerSigningKey = new SymmetricSecurityKey(
            Encoding.UTF8.GetBytes(jwtSettings.Secret)),
        ClockSkew = TimeSpan.Zero // Remove delay of token expiry
    };

    options.Events = new JwtBearerEvents
    {
        OnAuthenticationFailed = context =>
        {
            if (context.Exception.GetType() == typeof(SecurityTokenExpiredException))
            {
                context.Response.Headers.Add("Token-Expired", "true");
            }
            return Task.CompletedTask;
        }
    };
});

// Add Authorization
builder.Services.AddAuthorization(options =>
{
    options.AddPolicy("AdminOnly", policy =>
        policy.RequireRole("Admin"));
    
    options.AddPolicy("UserOrAdmin", policy =>
        policy.RequireRole("User", "Admin"));
});

// ... other services

var app = builder.Build();

// ... middleware

app.UseAuthentication();
app.UseAuthorization();

app.MapControllers();

app.Run();
```

### appsettings.json Configuration

```json
{
  "Logging": {
    "LogLevel": {
      "Default": "Information",
      "Microsoft.AspNetCore": "Warning",
      "Microsoft.EntityFrameworkCore": "Information"
    }
  },
  "AllowedHosts": "*",
  "ConnectionStrings": {
    "DefaultConnection": "Server=localhost;Database=YourProjectDb;Trusted_Connection=True;TrustServerCertificate=True;MultipleActiveResultSets=true"
  },
  "JwtSettings": {
    "Secret": "YourSuperSecretKeyThatIsAtLeast32CharactersLong!@#$%",
    "Issuer": "YourProjectAPI",
    "Audience": "YourProjectClient",
    "ExpiryMinutes": 60,
    "RefreshTokenExpiryDays": 7
  }
}
```

---

## 💉 Dependency Injection

### Service Registration Pattern

```csharp
// Infrastructure/DependencyInjection.cs
public static class DependencyInjection
{
    public static IServiceCollection AddInfrastructure(
        this IServiceCollection services,
        IConfiguration configuration)
    {
        // Database
        services.AddDbContext<ApplicationDbContext>(options =>
            options.UseSqlServer(
                configuration.GetConnectionString("DefaultConnection"),
                b => b.MigrationsAssembly(typeof(ApplicationDbContext).Assembly.FullName)));

        // Repositories
        services.AddScoped(typeof(IGenericRepository<>), typeof(GenericRepository<>));
        services.AddScoped<IUserRepository, UserRepository>();
        services.AddScoped<IProductRepository, ProductRepository>();
        services.AddScoped<IOrderRepository, OrderRepository>();

        // Identity & JWT
        services.AddScoped<IJwtTokenGenerator, JwtTokenGenerator>();

        return services;
    }
}

// Core/DependencyInjection.cs
public static class DependencyInjection
{
    public static IServiceCollection AddCore(this IServiceCollection services)
    {
        // Services
        services.AddScoped<IAuthService, AuthService>();
        services.AddScoped<IUserService, UserService>();
        services.AddScoped<IProductService, ProductService>();
        services.AddScoped<IOrderService, OrderService>();

        // Validators
        services.AddValidatorsFromAssembly(Assembly.GetExecutingAssembly());

        return services;
    }
}

// Program.cs
var builder = WebApplication.CreateBuilder(args);

// Add layers
builder.Services.AddCore();
builder.Services.AddInfrastructure(builder.Configuration);

// Add controllers
builder.Services.AddControllers();

// Add API versioning
builder.Services.AddApiVersioning(options =>
{
    options.DefaultApiVersion = new ApiVersion(1, 0);
    options.AssumeDefaultVersionWhenUnspecified = true;
    options.ReportApiVersions = true;
});

// Add Swagger
builder.Services.AddEndpointsApiExplorer();
builder.Services.AddSwaggerGen(c =>
{
    c.SwaggerDoc("v1", new OpenApiInfo
    {
        Title = "Your Project API",
        Version = "v1",
        Description = "API documentation"
    });

    // Add JWT authentication to Swagger
    c.AddSecurityDefinition("Bearer", new OpenApiSecurityScheme
    {
        Description = "JWT Authorization header using the Bearer scheme. Enter 'Bearer' [space] and then your token",
        Name = "Authorization",
        In = ParameterLocation.Header,
        Type = SecuritySchemeType.ApiKey,
        Scheme = "Bearer"
    });

    c.AddSecurityRequirement(new OpenApiSecurityRequirement
    {
        {
            new OpenApiSecurityScheme
            {
                Reference = new OpenApiReference
                {
                    Type = ReferenceType.SecurityScheme,
                    Id = "Bearer"
                }
            },
            Array.Empty<string>()
        }
    });
});

// Add CORS
builder.Services.AddCors(options =>
{
    options.AddPolicy("AllowAll", policy =>
    {
        policy.AllowAnyOrigin()
              .AllowAnyMethod()
              .AllowAnyHeader();
    });
});

// Add health checks
builder.Services.AddHealthChecks()
    .AddDbContextCheck<ApplicationDbContext>();
```

### Service Lifetime Guidelines

| Lifetime | Usage | Example |
|----------|-------|---------|
| **Singleton** | Services that are stateless and thread-safe | Configuration, Logging, Caching |
| **Scoped** | Services that should be created once per request | DbContext, Repositories, Business Services |
| **Transient** | Services that should be created every time they're requested | Lightweight, stateless services |

```csharp
// Singleton - Created once for application lifetime
services.AddSingleton<IConfiguration, Configuration>();

// Scoped - Created once per HTTP request
services.AddScoped<IUserService, UserService>();
services.AddScoped<ApplicationDbContext>();

// Transient - Created every time they're requested
services.AddTransient<IEmailService, EmailService>();
```

---

## 🔍 LINQ Best Practices

### ✅ DO - Performance Optimizations

#### 1. Use AsNoTracking for Read-Only Queries
```csharp
// ✅ GOOD - No tracking overhead for read-only data
var users = await _context.Users
    .AsNoTracking()
    .Where(u => u.IsActive)
    .ToListAsync(cancellationToken);

// ❌ BAD - Unnecessary tracking
var users = await _context.Users
    .Where(u => u.IsActive)
    .ToListAsync(cancellationToken);
```

#### 2. Project to DTOs Early (Select Projection)
```csharp
// ✅ GOOD - Only fetch required columns
var users = await _context.Users
    .AsNoTracking()
    .Where(u => u.IsActive)
    .Select(u => new UserResponseDto
    {
        Id = u.Id,
        Email = u.Email,
        FullName = $"{u.FirstName} {u.LastName}"
    })
    .ToListAsync(cancellationToken);

// ❌ BAD - Fetches all columns then maps in memory
var users = await _context.Users
    .AsNoTracking()
    .Where(u => u.IsActive)
    .ToListAsync(cancellationToken);

var dtos = users.Select(u => new UserResponseDto
{
    Id = u.Id,
    Email = u.Email,
    FullName = $"{u.FirstName} {u.LastName}"
});
```

#### 3. Use Pagination
```csharp
// ✅ GOOD - Server-side pagination
public async Task<PagedResult<UserResponseDto>> GetUsersPagedAsync(
    int pageNumber = 1,
    int pageSize = 10,
    CancellationToken cancellationToken = default)
{
    var query = _context.Users.AsNoTracking();

    var totalCount = await query.CountAsync(cancellationToken);

    var items = await query
        .OrderByDescending(u => u.CreatedAt)
        .Skip((pageNumber - 1) * pageSize)
        .Take(pageSize)
        .Select(u => new UserResponseDto
        {
            Id = u.Id,
            Email = u.Email
        })
        .ToListAsync(cancellationToken);

    return new PagedResult<UserResponseDto>(items, totalCount, pageNumber, pageSize);
}

// ❌ BAD - Loading all data
var allUsers = await _context.Users.ToListAsync();
var pagedUsers = allUsers.Skip((pageNumber - 1) * pageSize).Take(pageSize);
```

#### 4. Use Include/ThenInclude for Related Data (Avoid N+1)
```csharp
// ✅ GOOD - Single query with joins
var orders = await _context.Orders
    .AsNoTracking()
    .Include(o => o.Customer)
    .Include(o => o.OrderItems)
        .ThenInclude(oi => oi.Product)
    .Where(o => o.CustomerId == customerId)
    .ToListAsync(cancellationToken);

// ❌ BAD - N+1 problem
var orders = await _context.Orders
    .Where(o => o.CustomerId == customerId)
    .ToListAsync(cancellationToken);

foreach (var order in orders)
{
    // Separate query for each order
    order.Customer = await _context.Customers.FindAsync(order.CustomerId);
    
    // Separate query for each order's items
    order.OrderItems = await _context.OrderItems
        .Where(oi => oi.OrderId == order.Id)
        .ToListAsync();
}
```

#### 5. Use AsSplitQuery for Multiple Includes
```csharp
// ✅ GOOD - Splits into multiple queries to avoid cartesian explosion
var orders = await _context.Orders
    .AsNoTracking()
    .Include(o => o.OrderItems)
    .Include(o => o.Payments)
    .Include(o => o.Shipments)
    .AsSplitQuery()
    .ToListAsync(cancellationToken);

// ⚠️ WARNING - Can cause cartesian explosion with multiple collections
var orders = await _context.Orders
    .Include(o => o.OrderItems)
    .Include(o => o.Payments)
    .Include(o => o.Shipments)
    .ToListAsync(cancellationToken);
```

#### 6. Use Any() instead of Count() > 0
```csharp
// ✅ GOOD - Stops at first match
if (await _context.Users.AnyAsync(u => u.Email == email, cancellationToken))
{
    // User exists
}

// ❌ BAD - Counts all matching records
if (await _context.Users.CountAsync(u => u.Email == email, cancellationToken) > 0)
{
    // User exists
}
```

#### 7. Use Compiled Queries for Frequently Executed Queries
```csharp
// Define compiled query
private static readonly Func<ApplicationDbContext, string, Task<User?>> _getUserByEmail =
    EF.CompileAsyncQuery((ApplicationDbContext context, string email) =>
        context.Users
            .AsNoTracking()
            .FirstOrDefault(u => u.Email == email));

// Use compiled query
public async Task<User?> GetByEmailAsync(string email)
{
    return await _getUserByEmail(_context, email);
}
```

#### 8. Batch Operations
```csharp
// ✅ GOOD - Batch insert
public async Task AddRangeAsync(IEnumerable<User> users, CancellationToken cancellationToken)
{
    await _context.Users.AddRangeAsync(users, cancellationToken);
    await _context.SaveChangesAsync(cancellationToken);
}

// ❌ BAD - Individual inserts
public async Task AddUsersAsync(IEnumerable<User> users, CancellationToken cancellationToken)
{
    foreach (var user in users)
    {
        await _context.Users.AddAsync(user, cancellationToken);
        await _context.SaveChangesAsync(cancellationToken); // SaveChanges in loop!
    }
}
```

### ❌ DON'T - Common Mistakes

#### 1. Don't Load Entire Tables
```csharp
// ❌ BAD
var allUsers = await _context.Users.ToListAsync();
var activeUsers = allUsers.Where(u => u.IsActive);
```

#### 2. Don't Mix Client and Server Evaluation
```csharp
// ❌ BAD - CustomMethod can't be translated to SQL
var users = await _context.Users
    .Where(u => CustomMethod(u.Email))
    .ToListAsync();

// ✅ GOOD - Filter after query or use SQL-translatable expression
var users = await _context.Users.ToListAsync();
var filtered = users.Where(u => CustomMethod(u.Email));
```

#### 3. Don't Use Select().ToList().FirstOrDefault()
```csharp
// ❌ BAD - Fetches all, then takes first
var user = await _context.Users
    .Select(u => new { u.Id, u.Email })
    .ToListAsync()
    .FirstOrDefault();

// ✅ GOOD - Takes first on database side
var user = await _context.Users
    .Select(u => new { u.Id, u.Email })
    .FirstOrDefaultAsync();
```

#### 4. Don't Forget CancellationToken
```csharp
// ❌ BAD - No cancellation support
public async Task<List<User>> GetAllUsersAsync()
{
    return await _context.Users.ToListAsync();
}

// ✅ GOOD - Supports cancellation
public async Task<List<User>> GetAllUsersAsync(CancellationToken cancellationToken = default)
{
    return await _context.Users.ToListAsync(cancellationToken);
}
```

---

## 🎮 API Controllers Best Practices

### RESTful Endpoint Patterns

```csharp
[ApiController]
[Route("api/v{version:apiVersion}/[controller]")]
[ApiVersion("1.0")]
[Produces("application/json")]
public class UsersController : ControllerBase
{
    private readonly IUserService _userService;
    private readonly ILogger<UsersController> _logger;

    public UsersController(
        IUserService userService,
        ILogger<UsersController> logger)
    {
        _userService = userService;
        _logger = logger;
    }

    /// <summary>
    /// Get all users with pagination
    /// </summary>
    [HttpGet]
    [ProducesResponseType(typeof(PagedResult<UserResponseDto>), StatusCodes.Status200OK)]
    public async Task<IActionResult> GetUsers(
        [FromQuery] int pageNumber = 1,
        [FromQuery] int pageSize = 10,
        CancellationToken cancellationToken = default)
    {
        var result = await _userService.GetUsersPagedAsync(
            pageNumber,
            pageSize,
            cancellationToken);
        
        return Ok(result);
    }

    /// <summary>
    /// Get user by ID
    /// </summary>
    [HttpGet("{id:guid}")]
    [ProducesResponseType(typeof(UserResponseDto), StatusCodes.Status200OK)]
    [ProducesResponseType(StatusCodes.Status404NotFound)]
    public async Task<IActionResult> GetUser(
        Guid id,
        CancellationToken cancellationToken)
    {
        var user = await _userService.GetUserByIdAsync(id, cancellationToken);
        
        if (user == null)
        {
            return NotFound(new { message = $"User with ID {id} not found" });
        }
        
        return Ok(user);
    }

    /// <summary>
    /// Create new user
    /// </summary>
    [HttpPost]
    [ProducesResponseType(typeof(UserResponseDto), StatusCodes.Status201Created)]
    [ProducesResponseType(typeof(ValidationProblemDetails), StatusCodes.Status400BadRequest)]
    public async Task<IActionResult> CreateUser(
        [FromBody] CreateUserRequestDto request,
        CancellationToken cancellationToken)
    {
        var user = await _userService.CreateUserAsync(request, cancellationToken);
        
        return CreatedAtAction(
            nameof(GetUser),
            new { id = user.Id },
            user);
    }

    /// <summary>
    /// Update existing user
    /// </summary>
    [HttpPut("{id:guid}")]
    [ProducesResponseType(StatusCodes.Status204NoContent)]
    [ProducesResponseType(StatusCodes.Status404NotFound)]
    [ProducesResponseType(typeof(ValidationProblemDetails), StatusCodes.Status400BadRequest)]
    public async Task<IActionResult> UpdateUser(
        Guid id,
        [FromBody] UpdateUserRequestDto request,
        CancellationToken cancellationToken)
    {
        var result = await _userService.UpdateUserAsync(id, request, cancellationToken);
        
        if (!result)
        {
            return NotFound(new { message = $"User with ID {id} not found" });
        }
        
        return NoContent();
    }

    /// <summary>
    /// Delete user (soft delete)
    /// </summary>
    [HttpDelete("{id:guid}")]
    [ProducesResponseType(StatusCodes.Status204NoContent)]
    [ProducesResponseType(StatusCodes.Status404NotFound)]
    public async Task<IActionResult> DeleteUser(
        Guid id,
        CancellationToken cancellationToken)
    {
        var result = await _userService.DeleteUserAsync(id, cancellationToken);
        
        if (!result)
        {
            return NotFound(new { message = $"User with ID {id} not found" });
        }
        
        return NoContent();
    }

    /// <summary>
    /// Search users
    /// </summary>
    [HttpGet("search")]
    [ProducesResponseType(typeof(PagedResult<UserResponseDto>), StatusCodes.Status200OK)]
    public async Task<IActionResult> SearchUsers(
        [FromQuery] string searchTerm,
        [FromQuery] int pageNumber = 1,
        [FromQuery] int pageSize = 10,
        CancellationToken cancellationToken = default)
    {
        var result = await _userService.SearchUsersAsync(
            searchTerm,
            pageNumber,
            pageSize,
            cancellationToken);
        
        return Ok(result);
    }
}
```

### HTTP Status Codes Usage

| Status Code | Usage | Example |
|-------------|-------|---------|
| **200 OK** | Successful GET, PUT, PATCH | Returning data |
| **201 Created** | Successful POST | Resource created |
| **204 No Content** | Successful DELETE, PUT | No response body needed |
| **400 Bad Request** | Validation errors | Invalid input |
| **401 Unauthorized** | Authentication required | Missing/invalid token |
| **403 Forbidden** | Authorization failed | Insufficient permissions |
| **404 Not Found** | Resource not found | Invalid ID |
| **409 Conflict** | Business rule violation | Duplicate email |
| **500 Internal Server Error** | Unhandled server error | Caught by middleware |

---

# .NET API Copilot Instructions - Part 3
## (Continue from Part 2)

## ⚠️ Error Handling

### Custom Exception Classes

```csharp
// Core/Exceptions/BaseException.cs
public abstract class BaseException : Exception
{
    public int StatusCode { get; set; }

    protected BaseException(string message, int statusCode = 500)
        : base(message)
    {
        StatusCode = statusCode;
    }
}

// Core/Exceptions/NotFoundException.cs
public class NotFoundException : BaseException
{
    public NotFoundException(string message)
        : base(message, StatusCodes.Status404NotFound)
    {
    }

    public NotFoundException(string name, object key)
        : base($"{name} with ID '{key}' was not found", StatusCodes.Status404NotFound)
    {
    }
}

// Core/Exceptions/BadRequestException.cs
public class BadRequestException : BaseException
{
    public BadRequestException(string message)
        : base(message, StatusCodes.Status400BadRequest)
    {
    }
}

// Core/Exceptions/UnauthorizedException.cs
public class UnauthorizedException : BaseException
{
    public UnauthorizedException(string message = "Unauthorized access")
        : base(message, StatusCodes.Status401Unauthorized)
    {
    }
}

// Core/Exceptions/BusinessException.cs
public class BusinessException : BaseException
{
    public BusinessException(string message)
        : base(message, StatusCodes.Status422UnprocessableEntity)
    {
    }
}

// Core/Exceptions/ConflictException.cs
public class ConflictException : BaseException
{
    public ConflictException(string message)
        : base(message, StatusCodes.Status409Conflict)
    {
    }
}
```

### Global Exception Handling Middleware

```csharp
// API/Middleware/ExceptionHandlingMiddleware.cs
public class ExceptionHandlingMiddleware
{
    private readonly RequestDelegate _next;
    private readonly ILogger<ExceptionHandlingMiddleware> _logger;
    private readonly IHostEnvironment _environment;

    public ExceptionHandlingMiddleware(
        RequestDelegate next,
        ILogger<ExceptionHandlingMiddleware> logger,
        IHostEnvironment environment)
    {
        _next = next;
        _logger = logger;
        _environment = environment;
    }

    public async Task InvokeAsync(HttpContext context)
    {
        try
        {
            await _next(context);
        }
        catch (Exception ex)
        {
            await HandleExceptionAsync(context, ex);
        }
    }

    private async Task HandleExceptionAsync(HttpContext context, Exception exception)
    {
        _logger.LogError(
            exception,
            "An error occurred: {Message} | Path: {Path}",
            exception.Message,
            context.Request.Path);

        var response = context.Response;
        response.ContentType = "application/json";

        var errorResponse = new ErrorResponse
        {
            TraceId = Activity.Current?.Id ?? context.TraceIdentifier,
            Instance = context.Request.Path
        };

        switch (exception)
        {
            case NotFoundException notFoundException:
                response.StatusCode = StatusCodes.Status404NotFound;
                errorResponse.Title = "Resource Not Found";
                errorResponse.Status = response.StatusCode;
                errorResponse.Detail = notFoundException.Message;
                break;

            case BadRequestException badRequestException:
                response.StatusCode = StatusCodes.Status400BadRequest;
                errorResponse.Title = "Bad Request";
                errorResponse.Status = response.StatusCode;
                errorResponse.Detail = badRequestException.Message;
                break;

            case UnauthorizedException unauthorizedException:
                response.StatusCode = StatusCodes.Status401Unauthorized;
                errorResponse.Title = "Unauthorized";
                errorResponse.Status = response.StatusCode;
                errorResponse.Detail = unauthorizedException.Message;
                break;

            case ConflictException conflictException:
                response.StatusCode = StatusCodes.Status409Conflict;
                errorResponse.Title = "Conflict";
                errorResponse.Status = response.StatusCode;
                errorResponse.Detail = conflictException.Message;
                break;

            case BusinessException businessException:
                response.StatusCode = StatusCodes.Status422UnprocessableEntity;
                errorResponse.Title = "Business Rule Violation";
                errorResponse.Status = response.StatusCode;
                errorResponse.Detail = businessException.Message;
                break;

            case ValidationException validationException:
                response.StatusCode = StatusCodes.Status400BadRequest;
                errorResponse.Title = "Validation Error";
                errorResponse.Status = response.StatusCode;
                errorResponse.Detail = "One or more validation errors occurred";
                errorResponse.Errors = validationException.Errors
                    .GroupBy(e => e.PropertyName)
                    .ToDictionary(
                        g => g.Key,
                        g => g.Select(e => e.ErrorMessage).ToArray());
                break;

            default:
                response.StatusCode = StatusCodes.Status500InternalServerError;
                errorResponse.Title = "Internal Server Error";
                errorResponse.Status = response.StatusCode;
                errorResponse.Detail = _environment.IsDevelopment()
                    ? exception.Message
                    : "An unexpected error occurred";
                
                if (_environment.IsDevelopment())
                {
                    errorResponse.StackTrace = exception.StackTrace;
                }
                break;
        }

        await response.WriteAsJsonAsync(errorResponse);
    }
}

// API/Models/ErrorResponse.cs
public class ErrorResponse
{
    public string Title { get; set; } = string.Empty;
    public int Status { get; set; }
    public string Detail { get; set; } = string.Empty;
    public string? Instance { get; set; }
    public string? TraceId { get; set; }
    public Dictionary<string, string[]>? Errors { get; set; }
    public string? StackTrace { get; set; }
}

// Register in Program.cs
app.UseMiddleware<ExceptionHandlingMiddleware>();
```

### Validation with FluentValidation

```csharp
// Core/DTOs/Requests/CreateUserRequestDto.cs
public class CreateUserRequestDto
{
    public string Email { get; set; } = string.Empty;
    public string Password { get; set; } = string.Empty;
    public string FirstName { get; set; } = string.Empty;
    public string LastName { get; set; } = string.Empty;
}

// Core/Validators/CreateUserRequestValidator.cs
public class CreateUserRequestValidator : AbstractValidator<CreateUserRequestDto>
{
    public CreateUserRequestValidator()
    {
        RuleFor(x => x.Email)
            .NotEmpty().WithMessage("Email is required")
            .EmailAddress().WithMessage("Invalid email format")
            .MaximumLength(255).WithMessage("Email cannot exceed 255 characters");

        RuleFor(x => x.Password)
            .NotEmpty().WithMessage("Password is required")
            .MinimumLength(8).WithMessage("Password must be at least 8 characters")
            .Matches(@"^(?=.*[a-z])(?=.*[A-Z])(?=.*\d)(?=.*[@$!%*?&])")
            .WithMessage("Password must contain uppercase, lowercase, number and special character");

        RuleFor(x => x.FirstName)
            .NotEmpty().WithMessage("First name is required")
            .MaximumLength(100).WithMessage("First name cannot exceed 100 characters");

        RuleFor(x => x.LastName)
            .NotEmpty().WithMessage("Last name is required")
            .MaximumLength(100).WithMessage("Last name cannot exceed 100 characters");
    }
}

// API/Filters/ValidationFilter.cs
public class ValidationFilter : IAsyncActionFilter
{
    public async Task OnActionExecutionAsync(
        ActionExecutingContext context,
        ActionExecutionDelegate next)
    {
        if (!context.ModelState.IsValid)
        {
            var errors = context.ModelState
                .Where(x => x.Value?.Errors.Count > 0)
                .ToDictionary(
                    kvp => kvp.Key,
                    kvp => kvp.Value?.Errors.Select(e => e.ErrorMessage).ToArray() ?? Array.Empty<string>()
                );

            var errorResponse = new ErrorResponse
            {
                Title = "Validation Error",
                Status = StatusCodes.Status400BadRequest,
                Detail = "One or more validation errors occurred",
                Errors = errors,
                TraceId = Activity.Current?.Id ?? context.HttpContext.TraceIdentifier
            };

            context.Result = new BadRequestObjectResult(errorResponse);
            return;
        }

        await next();
    }
}

// Register in Program.cs
builder.Services.AddControllers(options =>
{
    options.Filters.Add<ValidationFilter>();
});

builder.Services.AddFluentValidationAutoValidation();
builder.Services.AddValidatorsFromAssembly(typeof(CreateUserRequestValidator).Assembly);
```

---

## 📝 Logging Standards

### Structured Logging with Serilog

```csharp
// Install packages:
// - Serilog.AspNetCore
// - Serilog.Sinks.Console
// - Serilog.Sinks.File
// - Serilog.Sinks.Seq (optional)

// Program.cs
using Serilog;

var builder = WebApplication.CreateBuilder(args);

// Configure Serilog
Log.Logger = new LoggerConfiguration()
    .ReadFrom.Configuration(builder.Configuration)
    .Enrich.FromLogContext()
    .Enrich.WithMachineName()
    .Enrich.WithEnvironmentName()
    .WriteTo.Console(
        outputTemplate: "[{Timestamp:HH:mm:ss} {Level:u3}] {Message:lj} {Properties:j}{NewLine}{Exception}")
    .WriteTo.File(
        path: "logs/log-.txt",
        rollingInterval: RollingInterval.Day,
        outputTemplate: "{Timestamp:yyyy-MM-dd HH:mm:ss.fff zzz} [{Level:u3}] {Message:lj} {Properties:j}{NewLine}{Exception}")
    .CreateLogger();

builder.Host.UseSerilog();

var app = builder.Build();

// Use Serilog request logging
app.UseSerilogRequestLogging(options =>
{
    options.MessageTemplate = "HTTP {RequestMethod} {RequestPath} responded {StatusCode} in {Elapsed:0.0000} ms";
    options.EnrichDiagnosticContext = (diagnosticContext, httpContext) =>
    {
        diagnosticContext.Set("RequestHost", httpContext.Request.Host.Value);
        diagnosticContext.Set("RequestScheme", httpContext.Request.Scheme);
        diagnosticContext.Set("UserAgent", httpContext.Request.Headers["User-Agent"].ToString());
    };
});

app.Run();

// Ensure logs are flushed on shutdown
Log.CloseAndFlush();
```

### appsettings.json - Serilog Configuration

```json
{
  "Serilog": {
    "MinimumLevel": {
      "Default": "Information",
      "Override": {
        "Microsoft": "Warning",
        "Microsoft.AspNetCore": "Warning",
        "Microsoft.EntityFrameworkCore": "Information",
        "System": "Warning"
      }
    },
    "WriteTo": [
      {
        "Name": "Console",
        "Args": {
          "outputTemplate": "[{Timestamp:HH:mm:ss} {Level:u3}] {Message:lj} {Properties:j}{NewLine}{Exception}"
        }
      },
      {
        "Name": "File",
        "Args": {
          "path": "logs/log-.txt",
          "rollingInterval": "Day",
          "retainedFileCountLimit": 30,
          "outputTemplate": "{Timestamp:yyyy-MM-dd HH:mm:ss.fff zzz} [{Level:u3}] {Message:lj} {Properties:j}{NewLine}{Exception}"
        }
      }
    ]
  }
}
```

### Logging Best Practices

```csharp
public class UserService : IUserService
{
    private readonly IUserRepository _userRepository;
    private readonly ILogger<UserService> _logger;

    public UserService(IUserRepository userRepository, ILogger<UserService> logger)
    {
        _userRepository = userRepository;
        _logger = logger;
    }

    public async Task<UserResponseDto> CreateUserAsync(
        CreateUserRequestDto request,
        CancellationToken cancellationToken)
    {
        // ✅ GOOD - Structured logging with parameters
        _logger.LogInformation(
            "Creating user with email: {Email}",
            request.Email);

        try
        {
            var existingUser = await _userRepository.GetByEmailAsync(
                request.Email,
                cancellationToken);

            if (existingUser != null)
            {
                // ✅ GOOD - Log with context
                _logger.LogWarning(
                    "User creation failed - email already exists: {Email}",
                    request.Email);
                
                throw new ConflictException($"User with email {request.Email} already exists");
            }

            var user = new User
            {
                Id = Guid.NewGuid(),
                Email = request.Email,
                FirstName = request.FirstName,
                LastName = request.LastName,
                CreatedAt = DateTime.UtcNow
            };

            await _userRepository.AddAsync(user, cancellationToken);

            // ✅ GOOD - Log success with key information
            _logger.LogInformation(
                "User created successfully - ID: {UserId}, Email: {Email}",
                user.Id,
                user.Email);

            return MapToDto(user);
        }
        catch (Exception ex) when (ex is not ConflictException)
        {
            // ✅ GOOD - Log errors with exception details
            _logger.LogError(
                ex,
                "Error creating user with email: {Email}",
                request.Email);
            throw;
        }
    }

    // ❌ BAD - String concatenation
    // _logger.LogInformation("Creating user with email: " + request.Email);

    // ❌ BAD - No context
    // _logger.LogInformation("Creating user");

    // ❌ BAD - Logging sensitive data
    // _logger.LogInformation("User password: {Password}", request.Password);
}
```

### Log Levels Guide

| Level | When to Use | Example |
|-------|-------------|---------|
| **Trace** | Detailed diagnostic information | Entry/exit of methods, loop iterations |
| **Debug** | Developer debugging information | Variable values, execution flow |
| **Information** | General flow of application | User actions, business events |
| **Warning** | Abnormal or unexpected events | Deprecated API usage, soft failures |
| **Error** | Failures/exceptions that need attention | Caught exceptions, failed operations |
| **Critical** | Critical failures requiring immediate action | Application crashes, data loss |

---

## 🧪 Testing with XUnit

### Test Project Structure

```
YourProject.UnitTests/
├── Services/
│   ├── UserServiceTests.cs
│   └── AuthServiceTests.cs
├── Repositories/
│   └── UserRepositoryTests.cs
├── Controllers/
│   └── UsersControllerTests.cs
├── Helpers/
│   └── TestDataHelper.cs
└── Fixtures/
    └── DatabaseFixture.cs
```

### Unit Test Example - Service Layer

```csharp
// YourProject.UnitTests/Services/UserServiceTests.cs
public class UserServiceTests
{
    private readonly Mock<IUserRepository> _userRepositoryMock;
    private readonly Mock<ILogger<UserService>> _loggerMock;
    private readonly UserService _userService;

    public UserServiceTests()
    {
        _userRepositoryMock = new Mock<IUserRepository>();
        _loggerMock = new Mock<ILogger<UserService>>();
        _userService = new UserService(_userRepositoryMock.Object, _loggerMock.Object);
    }

    [Fact]
    public async Task GetUserByIdAsync_WhenUserExists_ReturnsUserDto()
    {
        // Arrange
        var userId = Guid.NewGuid();
        var user = new User
        {
            Id = userId,
            Email = "test@example.com",
            FirstName = "John",
            LastName = "Doe",
            IsActive = true,
            CreatedAt = DateTime.UtcNow
        };

        _userRepositoryMock
            .Setup(x => x.GetByIdAsync(userId, It.IsAny<CancellationToken>()))
            .ReturnsAsync(user);

        // Act
        var result = await _userService.GetUserByIdAsync(userId, CancellationToken.None);

        // Assert
        Assert.NotNull(result);
        Assert.Equal(userId, result.Id);
        Assert.Equal("test@example.com", result.Email);
        Assert.Equal("John Doe", result.FullName);
        
        _userRepositoryMock.Verify(
            x => x.GetByIdAsync(userId, It.IsAny<CancellationToken>()),
            Times.Once);
    }

    [Fact]
    public async Task GetUserByIdAsync_WhenUserDoesNotExist_ReturnsNull()
    {
        // Arrange
        var userId = Guid.NewGuid();
        
        _userRepositoryMock
            .Setup(x => x.GetByIdAsync(userId, It.IsAny<CancellationToken>()))
            .ReturnsAsync((User?)null);

        // Act
        var result = await _userService.GetUserByIdAsync(userId, CancellationToken.None);

        // Assert
        Assert.Null(result);
    }

    [Fact]
    public async Task CreateUserAsync_WhenEmailExists_ThrowsConflictException()
    {
        // Arrange
        var request = new CreateUserRequestDto
        {
            Email = "existing@example.com",
            FirstName = "John",
            LastName = "Doe",
            Password = "Password123!"
        };

        var existingUser = new User { Email = request.Email };

        _userRepositoryMock
            .Setup(x => x.GetByEmailAsync(request.Email, It.IsAny<CancellationToken>()))
            .ReturnsAsync(existingUser);

        // Act & Assert
        await Assert.ThrowsAsync<ConflictException>(
            () => _userService.CreateUserAsync(request, CancellationToken.None));
        
        _userRepositoryMock.Verify(
            x => x.AddAsync(It.IsAny<User>(), It.IsAny<CancellationToken>()),
            Times.Never);
    }

    [Fact]
    public async Task CreateUserAsync_WhenEmailIsUnique_CreatesUser()
    {
        // Arrange
        var request = new CreateUserRequestDto
        {
            Email = "new@example.com",
            FirstName = "Jane",
            LastName = "Doe",
            Password = "Password123!"
        };

        _userRepositoryMock
            .Setup(x => x.GetByEmailAsync(request.Email, It.IsAny<CancellationToken>()))
            .ReturnsAsync((User?)null);

        _userRepositoryMock
            .Setup(x => x.AddAsync(It.IsAny<User>(), It.IsAny<CancellationToken>()))
            .ReturnsAsync((User user, CancellationToken ct) => user);

        // Act
        var result = await _userService.CreateUserAsync(request, CancellationToken.None);

        // Assert
        Assert.NotNull(result);
        Assert.Equal(request.Email, result.Email);
        Assert.Equal("Jane Doe", result.FullName);
        
        _userRepositoryMock.Verify(
            x => x.AddAsync(It.Is<User>(u =>
                u.Email == request.Email &&
                u.FirstName == request.FirstName &&
                u.LastName == request.LastName),
                It.IsAny<CancellationToken>()),
            Times.Once);
    }

    [Theory]
    [InlineData("", "John", "Doe", "Password123!")]
    [InlineData("invalid-email", "John", "Doe", "Password123!")]
    [InlineData("test@example.com", "", "Doe", "Password123!")]
    [InlineData("test@example.com", "John", "", "Password123!")]
    public async Task CreateUserAsync_WithInvalidData_ThrowsValidationException(
        string email,
        string firstName,
        string lastName,
        string password)
    {
        // Arrange
        var request = new CreateUserRequestDto
        {
            Email = email,
            FirstName = firstName,
            LastName = lastName,
            Password = password
        };

        var validator = new CreateUserRequestValidator();

        // Act
        var validationResult = await validator.ValidateAsync(request);

        // Assert
        Assert.False(validationResult.IsValid);
        Assert.NotEmpty(validationResult.Errors);
    }
}
```

### Repository Integration Test

```csharp
// YourProject.IntegrationTests/Repositories/UserRepositoryTests.cs
public class UserRepositoryTests : IClassFixture<DatabaseFixture>
{
    private readonly ApplicationDbContext _context;
    private readonly UserRepository _repository;

    public UserRepositoryTests(DatabaseFixture fixture)
    {
        _context = fixture.CreateContext();
        _repository = new UserRepository(_context);
    }

    [Fact]
    public async Task GetByIdAsync_WhenUserExists_ReturnsUser()
    {
        // Arrange
        var user = new User
        {
            Id = Guid.NewGuid(),
            Email = "test@example.com",
            FirstName = "John",
            LastName = "Doe",
            PasswordHash = "hashedpassword",
            IsActive = true,
            CreatedAt = DateTime.UtcNow
        };

        _context.Users.Add(user);
        await _context.SaveChangesAsync();

        // Act
        var result = await _repository.GetByIdAsync(user.Id);

        // Assert
        Assert.NotNull(result);
        Assert.Equal(user.Id, result.Id);
        Assert.Equal(user.Email, result.Email);
    }

    [Fact]
    public async Task AddAsync_CreatesNewUser()
    {
        // Arrange
        var user = new User
        {
            Id = Guid.NewGuid(),
            Email = "newuser@example.com",
            FirstName = "New",
            LastName = "User",
            PasswordHash = "hashedpassword",
            IsActive = true,
            CreatedAt = DateTime.UtcNow
        };

        // Act
        var result = await _repository.AddAsync(user);

        // Assert
        Assert.NotNull(result);
        var savedUser = await _context.Users.FindAsync(user.Id);
        Assert.NotNull(savedUser);
        Assert.Equal(user.Email, savedUser.Email);
    }

    public void Dispose()
    {
        _context.Database.EnsureDeleted();
        _context.Dispose();
    }
}

// YourProject.IntegrationTests/Fixtures/DatabaseFixture.cs
public class DatabaseFixture : IDisposable
{
    private readonly string _connectionString;

    public DatabaseFixture()
    {
        _connectionString = $"Server=(localdb)\\mssqllocaldb;Database=TestDb_{Guid.NewGuid()};Trusted_Connection=True;";
    }

    public ApplicationDbContext CreateContext()
    {
        var options = new DbContextOptionsBuilder<ApplicationDbContext>()
            .UseSqlServer(_connectionString)
            .Options;

        var context = new ApplicationDbContext(options);
        context.Database.EnsureCreated();
        
        return context;
    }

    public void Dispose()
    {
        var options = new DbContextOptionsBuilder<ApplicationDbContext>()
            .UseSqlServer(_connectionString)
            .Options;

        using var context = new ApplicationDbContext(options);
        context.Database.EnsureDeleted();
    }
}
```

### Controller Test Example

```csharp
// YourProject.UnitTests/Controllers/UsersControllerTests.cs
public class UsersControllerTests
{
    private readonly Mock<IUserService> _userServiceMock;
    private readonly Mock<ILogger<UsersController>> _loggerMock;
    private readonly UsersController _controller;

    public UsersControllerTests()
    {
        _userServiceMock = new Mock<IUserService>();
        _loggerMock = new Mock<ILogger<UsersController>>();
        _controller = new UsersController(_userServiceMock.Object, _loggerMock.Object);
    }

    [Fact]
    public async Task GetUserById_WhenUserExists_ReturnsOkResult()
    {
        // Arrange
        var userId = Guid.NewGuid();
        var userDto = new UserResponseDto
        {
            Id = userId,
            Email = "test@example.com",
            FirstName = "John",
            LastName = "Doe"
        };

        _userServiceMock
            .Setup(x => x.GetUserByIdAsync(userId, It.IsAny<CancellationToken>()))
            .ReturnsAsync(userDto);

        // Act
        var result = await _controller.GetUserById(userId, CancellationToken.None);

        // Assert
        var okResult = Assert.IsType<OkObjectResult>(result);
        var returnedUser = Assert.IsType<UserResponseDto>(okResult.Value);
        Assert.Equal(userId, returnedUser.Id);
    }

    [Fact]
    public async Task GetUserById_WhenUserDoesNotExist_ReturnsNotFound()
    {
        // Arrange
        var userId = Guid.NewGuid();
        
        _userServiceMock
            .Setup(x => x.GetUserByIdAsync(userId, It.IsAny<CancellationToken>()))
            .ReturnsAsync((UserResponseDto?)null);

        // Act
        var result = await _controller.GetUserById(userId, CancellationToken.None);

        // Assert
        Assert.IsType<NotFoundObjectResult>(result);
    }

    [Fact]
    public async Task CreateUser_WithValidData_ReturnsCreatedResult()
    {
        // Arrange
        var request = new CreateUserRequestDto
        {
            Email = "new@example.com",
            FirstName = "John",
            LastName = "Doe",
            Password = "Password123!"
        };

        var userDto = new UserResponseDto
        {
            Id = Guid.NewGuid(),
            Email = request.Email,
            FirstName = request.FirstName,
            LastName = request.LastName
        };

        _userServiceMock
            .Setup(x => x.CreateUserAsync(request, It.IsAny<CancellationToken>()))
            .ReturnsAsync(userDto);

        // Act
        var result = await _controller.CreateUser(request, CancellationToken.None);

        // Assert
        var createdResult = Assert.IsType<CreatedAtActionResult>(result);
        var returnedUser = Assert.IsType<UserResponseDto>(createdResult.Value);
        Assert.Equal(userDto.Id, returnedUser.Id);
    }
}
```

---

## ✅ DO's and ❌ DON'Ts

### ✅ DO - Best Practices

#### 1. Always Use Async/Await Properly
```csharp
// ✅ GOOD
public async Task<User?> GetUserAsync(Guid id, CancellationToken cancellationToken)
{
    return await _context.Users
        .AsNoTracking()
        .FirstOrDefaultAsync(u => u.Id == id, cancellationToken);
}

// ❌ BAD - Blocking call
public User? GetUser(Guid id)
{
    return _context.Users.FirstOrDefault(u => u.Id == id); // Synchronous
}

// ❌ BAD - Async without await
public Task<User?> GetUserAsync(Guid id)
{
    return Task.Run(() => _context.Users.FirstOrDefault(u => u.Id == id));
}
```

#### 2. Always Pass CancellationToken
```csharp
// ✅ GOOD
public async Task<List<User>> GetAllUsersAsync(CancellationToken cancellationToken = default)
{
    return await _context.Users.ToListAsync(cancellationToken);
}

// ❌ BAD
public async Task<List<User>> GetAllUsersAsync()
{
    return await _context.Users.ToListAsync(); // No cancellation support
}
```

#### 3. Use Dependency Injection
```csharp
// ✅ GOOD
public class UserService : IUserService
{
    private readonly IUserRepository _userRepository;
    private readonly ILogger<UserService> _logger;

    public UserService(IUserRepository userRepository, ILogger<UserService> logger)
    {
        _userRepository = userRepository;
        _logger = logger;
    }
}

// ❌ BAD - Direct instantiation
public class UserService
{
    private readonly UserRepository _userRepository = new UserRepository();
}
```

#### 4. Implement Repository Pattern Correctly
```csharp
// ✅ GOOD - Repository handles data access
public class UserRepository : IUserRepository
{
    private readonly ApplicationDbContext _context;

    public async Task<User?> GetByIdAsync(Guid id, CancellationToken cancellationToken)
    {
        return await _context.Users
            .AsNoTracking()
            .FirstOrDefaultAsync(u => u.Id == id, cancellationToken);
    }
}

// ❌ BAD - Controller accessing DbContext directly
public class UsersController : ControllerBase
{
    private readonly ApplicationDbContext _context;
    
    public async Task<IActionResult> GetUser(Guid id)
    {
        var user = await _context.Users.FindAsync(id); // Violates separation
        return Ok(user);
    }
}
```

#### 5. Use DTOs for API Contracts
```csharp
// ✅ GOOD - Separate DTO from Entity
public class UserResponseDto
{
    public Guid Id { get; set; }
    public string Email { get; set; }
    public string FullName { get; set; }
}

public async Task<IActionResult> GetUser(Guid id)
{
    var user = await _userService.GetUserByIdAsync(id);
    return Ok(user); // Returns DTO
}

// ❌ BAD - Returning entities directly
public async Task<IActionResult> GetUser(Guid id)
{
    var user = await _context.Users.FindAsync(id);
    return Ok(user); // Returns Entity with all properties including PasswordHash!
}
```

#### 6. Validate Input Data
```csharp
// ✅ GOOD - FluentValidation
public class CreateUserRequestValidator : AbstractValidator<CreateUserRequestDto>
{
    public CreateUserRequestValidator()
    {
        RuleFor(x => x.Email).NotEmpty().EmailAddress();
        RuleFor(x => x.Password).MinimumLength(8);
    }
}

// ❌ BAD - No validation
public async Task<IActionResult> CreateUser(CreateUserRequestDto request)
{
    var user = new User { Email = request.Email }; // No validation!
    await _userRepository.AddAsync(user);
    return Ok(user);
}
```

#### 7. Implement Proper Exception Handling
```csharp
// ✅ GOOD - Custom exceptions with middleware
public async Task<UserResponseDto> GetUserByIdAsync(Guid id)
{
    var user = await _userRepository.GetByIdAsync(id);
    
    if (user == null)
    {
        throw new NotFoundException("User", id);
    }
    
    return MapToDto(user);
}

// ❌ BAD - Generic exceptions
public async Task<UserResponseDto> GetUserByIdAsync(Guid id)
{
    var user = await _userRepository.GetByIdAsync(id);
    
    if (user == null)
    {
        throw new Exception("User not found"); // Generic, no status code
    }
    
    return MapToDto(user);
}
```

#### 8. Use Structured Logging
```csharp
// ✅ GOOD
_logger.LogInformation(
    "User {UserId} updated by {UpdatedBy}",
    userId,
    updatedBy);

// ❌ BAD
_logger.LogInformation($"User {userId} updated by {updatedBy}");
_logger.LogInformation("User " + userId + " updated");
```

#### 9. Write Unit Tests
```csharp
// ✅ GOOD - AAA Pattern (Arrange, Act, Assert)
[Fact]
public async Task GetUserByIdAsync_WhenUserExists_ReturnsUser()
{
    // Arrange
    var userId = Guid.NewGuid();
    _userRepositoryMock
        .Setup(x => x.GetByIdAsync(userId, It.IsAny<CancellationToken>()))
        .ReturnsAsync(new User { Id = userId });

    // Act
    var result = await _userService.GetUserByIdAsync(userId);

    // Assert
    Assert.NotNull(result);
    Assert.Equal(userId, result.Id);
}
```

#### 10. Use Configuration Files
```csharp
// ✅ GOOD - From appsettings.json
builder.Services.Configure<JwtSettings>(
    builder.Configuration.GetSection(JwtSettings.SectionName));

// ❌ BAD - Hardcoded values
var secret = "MyHardcodedSecret123!";
```

### ❌ DON'T - Common Mistakes

#### 1. Don't Use Blocking Calls
```csharp
// ❌ BAD
public User GetUser(Guid id)
{
    return _context.Users.Find(id); // Synchronous, blocks thread
}

public async Task ProcessAsync()
{
    var user = GetUser(userId); // Blocking in async method
}
```

#### 2. Don't Ignore CancellationToken
```csharp
// ❌ BAD
public async Task<List<User>> GetUsersAsync(CancellationToken cancellationToken)
{
    await Task.Delay(5000); // Not passing cancellationToken
    return await _context.Users.ToListAsync(); // Not passing cancellationToken
}
```

#### 3. Don't Create Tight Coupling
```csharp
// ❌ BAD - Direct dependency on concrete class
public class UserService
{
    private readonly UserRepository _userRepository;
    
    public UserService()
    {
        _userRepository = new UserRepository(); // Tight coupling!
    }
}
```

#### 4. Don't Return Null for Collections
```csharp
// ❌ BAD
public async Task<List<User>?> GetUsersAsync()
{
    var users = await _context.Users.ToListAsync();
    return users.Count == 0 ? null : users; // Don't return null!
}

// ✅ GOOD
public async Task<List<User>> GetUsersAsync()
{
    return await _context.Users.ToListAsync(); // Returns empty list if no users
}
```

#### 5. Don't Catch and Swallow Exceptions
```csharp
// ❌ BAD
try
{
    await _userRepository.AddAsync(user);
}
catch (Exception)
{
    // Swallowed exception, no logging or rethrowing
}

// ✅ GOOD
try
{
    await _userRepository.AddAsync(user);
}
catch (Exception ex)
{
    _logger.LogError(ex, "Error adding user");
    throw; // Rethrow or handle appropriately
}
```

#### 6. Don't Use Magic Strings/Numbers
```csharp
// ❌ BAD
if (user.Role == "Admin") // Magic string
{
    // Do something
}

// ✅ GOOD
public static class UserRoles
{
    public const string Admin = "Admin";
    public const string User = "User";
}

if (user.Role == UserRoles.Admin)
{
    // Do something
}
```

#### 7. Don't Mix Business Logic in Controllers
```csharp
// ❌ BAD
[HttpPost]
public async Task<IActionResult> CreateUser(CreateUserRequestDto request)
{
    // Business logic in controller!
    var existingUser = await _context.Users.FirstOrDefaultAsync(u => u.Email == request.Email);
    if (existingUser != null)
        return BadRequest("Email exists");
    
    var user = new User { Email = request.Email };
    _context.Users.Add(user);
    await _context.SaveChangesAsync();
    
    return Ok(user);
}

// ✅ GOOD
[HttpPost]
public async Task<IActionResult> CreateUser(CreateUserRequestDto request)
{
    var user = await _userService.CreateUserAsync(request); // Business logic in service
    return CreatedAtAction(nameof(GetUser), new { id = user.Id }, user);
}
```

#### 8. Don't Forget to Dispose Resources
```csharp
// ❌ BAD
public void ProcessFile(string path)
{
    var stream = new FileStream(path, FileMode.Open);
    // Process file
    // Stream not disposed!
}

// ✅ GOOD
public void ProcessFile(string path)
{
    using var stream = new FileStream(path, FileMode.Open);
    // Process file
} // Automatically disposed
```

#### 9. Don't Store Passwords in Plain Text
```csharp
// ❌ BAD
var user = new User
{
    Password = request.Password // Plain text password!
};

// ✅ GOOD
var user = new User
{
    PasswordHash = BCrypt.Net.BCrypt.HashPassword(request.Password)
};
```

#### 10. Don't Return Sensitive Data
```csharp
// ❌ BAD
public async Task<IActionResult> GetUser(Guid id)
{
    var user = await _context.Users.FindAsync(id);
    return Ok(user); // Returns PasswordHash and other sensitive fields!
}

// ✅ GOOD
public async Task<IActionResult> GetUser(Guid id)
{
    var user = await _userService.GetUserByIdAsync(id);
    return Ok(user); // Returns DTO with only safe fields
}
```

---

## 📦 Required NuGet Packages

```xml
<!-- YourProject.API -->
<ItemGroup>
  <PackageReference Include="Microsoft.AspNetCore.Authentication.JwtBearer" Version="8.0.*" />
  <PackageReference Include="Microsoft.AspNetCore.OpenApi" Version="8.0.*" />
  <PackageReference Include="Swashbuckle.AspNetCore" Version="6.5.*" />
  <PackageReference Include="Serilog.AspNetCore" Version="8.0.*" />
  <PackageReference Include="Serilog.Sinks.Console" Version="5.0.*" />
  <PackageReference Include="Serilog.Sinks.File" Version="5.0.*" />
  <PackageReference Include="FluentValidation.AspNetCore" Version="11.3.*" />
</ItemGroup>

<!-- YourProject.Infrastructure -->
<ItemGroup>
  <PackageReference Include="Microsoft.EntityFrameworkCore" Version="8.0.*" />
  <PackageReference Include="Microsoft.EntityFrameworkCore.SqlServer" Version="8.0.*" />
  <PackageReference Include="Microsoft.EntityFrameworkCore.Tools" Version="8.0.*" />
  <PackageReference Include="BCrypt.Net-Next" Version="4.0.*" />
</ItemGroup>

<!-- YourProject.Core -->
<ItemGroup>
  <PackageReference Include="FluentValidation" Version="11.9.*" />
</ItemGroup>

<!-- YourProject.UnitTests -->
<ItemGroup>
  <PackageReference Include="Microsoft.NET.Test.Sdk" Version="17.9.*" />
  <PackageReference Include="xunit" Version="2.6.*" />
  <PackageReference Include="xunit.runner.visualstudio" Version="2.5.*" />
  <PackageReference Include="Moq" Version="4.20.*" />
  <PackageReference Include="FluentAssertions" Version="6.12.*" />
  <PackageReference Include="Microsoft.EntityFrameworkCore.InMemory" Version="8.0.*" />
</ItemGroup>
```

---

## 🎯 Summary Checklist

Use this checklist when developing:

- [ ] Project structure follows N-tier architecture
- [ ] All dependencies flow in the correct direction (API → Core → Infrastructure)
- [ ] Naming conventions are consistent (PascalCase, camelCase, IPascalCase)
- [ ] All async methods have `Async` suffix and accept `CancellationToken`
- [ ] Repository pattern is implemented correctly
- [ ] DTOs are used for API contracts (not entities)
- [ ] FluentValidation is configured for input validation
- [ ] JWT authentication is properly configured
- [ ] Dependency injection is used throughout
- [ ] LINQ queries use `AsNoTracking()` for read-only operations
- [ ] LINQ queries project to DTOs early
- [ ] Pagination is implemented for list endpoints
- [ ] Structured logging with Serilog is configured
- [ ] Global exception handling middleware is in place
- [ ] Custom exceptions are used for business errors
- [ ] Unit tests cover services and repositories
- [ ] Integration tests cover database operations
- [ ] No hardcoded connection strings or secrets
- [ ] Configuration comes from appsettings.json
- [ ] Swagger/OpenAPI documentation is configured
- [ ] Health checks are implemented

---

**END OF COPILOT INSTRUCTIONS**
