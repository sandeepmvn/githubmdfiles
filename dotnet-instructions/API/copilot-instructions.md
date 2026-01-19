# .NET API Project - GitHub Copilot Instructions

> **Purpose**: These are workspace-wide coding standards for .NET 8+ Web API projects using N-tier architecture. All AI-assisted code generation must follow these guidelines.

---

## 🎯 Core Principles

### Architecture
- **N-tier Architecture**: API → Core → Infrastructure
- **Dependency Flow**: Always flows inward (API depends on Core, Core depends on Infrastructure)
- **Separation of Concerns**: Controllers (HTTP), Services (Business Logic), Repositories (Data Access)

### Naming Conventions
- **PascalCase**: Classes, Interfaces, Properties, Methods, Enums
- **camelCase**: Parameters, local variables, private fields
- **Interfaces**: Prefix with `I` (e.g., `IUserService`)
- **Async Methods**: Suffix with `Async` (e.g., `GetUserAsync`)
- **DTOs**: Suffix with purpose (e.g., `CreateUserRequestDto`, `UserResponseDto`)

### Code Quality Standards
- **All methods must be async** with `Task<T>` return type
- **All async methods must accept `CancellationToken`** (last parameter with default value)
- **Use dependency injection** for all dependencies (never `new` keyword for services)
- **DTOs for all API contracts** (never return entities directly)
- **Repository pattern** for all data access
- **AsNoTracking** for all read-only queries
- **Pagination required** for all list endpoints

---

## 📁 Project Structure

```
YourProject.sln
│
├── src/
│   ├── YourProject.API/              # Controllers, Middleware, Filters
│   ├── YourProject.Core/             # DTOs, Services, Interfaces
│   └── YourProject.Infrastructure/   # Entities, Repositories, DbContext
│
└── tests/
    ├── YourProject.UnitTests/
    └── YourProject.IntegrationTests/
```

### Layer Responsibilities

**API Layer**:
- Handle HTTP requests/responses
- Route mapping and validation
- Authentication/Authorization
- Exception handling middleware
- **Never** contain business logic or data access

**Core Layer**:
- Business logic and validation
- Service interfaces and implementations
- DTOs (Request/Response)
- Custom exceptions
- Domain logic

**Infrastructure Layer**:
- EF Core DbContext
- Entity definitions
- Repository implementations
- Database migrations
- External service integrations

---

## 🔥 CRITICAL PERFORMANCE RULES

### LINQ Rule #1: ALWAYS Project in the Query
```csharp
// ❌ NEVER - Loads all columns, maps in memory
var users = await _context.Users.ToListAsync();
var dtos = users.Select(u => new UserDto { ... });

// ✅ ALWAYS - Database-side projection
var users = await _context.Users
    .Select(u => new UserDto { Id = u.Id, Email = u.Email })
    .ToListAsync(cancellationToken);
```

### LINQ Rule #2: ALWAYS Paginate
```csharp
// ❌ NEVER - Loads entire table
var all = await _context.Users.ToListAsync();

// ✅ ALWAYS - Database-side pagination
var users = await _context.Users
    .AsNoTracking()
    .OrderByDescending(u => u.CreatedAt)
    .Skip((pageNumber - 1) * pageSize)
    .Take(pageSize)
    .Select(u => new UserDto { ... })
    .ToListAsync(cancellationToken);
```

### LINQ Rule #3: ALWAYS Use AsNoTracking for Reads
```csharp
// ✅ Every read-only query
var users = await _context.Users
    .AsNoTracking()
    .Where(u => u.IsActive)
    .Select(u => new UserDto { ... })
    .ToListAsync(cancellationToken);
```

### Query Order of Operations (MANDATORY)
1. `AsNoTracking()`
2. `Where()` (filters)
3. `CountAsync()` (for pagination total)
4. `OrderBy()` (required before pagination)
5. `Skip()` / `Take()` (pagination)
6. **`Select()` (BEFORE ToListAsync)**
7. `ToListAsync(cancellationToken)`

---

## 📋 Standard Patterns

### Repository Method Pattern
```csharp
public async Task<PagedResult<TDto>> GetPagedAsync(
    int pageNumber,
    int pageSize,
    string? searchTerm,
    CancellationToken cancellationToken = default)
{
    var query = _context.YourEntity.AsNoTracking();

    if (!string.IsNullOrWhiteSpace(searchTerm))
        query = query.Where(e => e.Name.Contains(searchTerm));

    var totalCount = await query.CountAsync(cancellationToken);

    var items = await query
        .OrderByDescending(e => e.CreatedAt)
        .Skip((pageNumber - 1) * pageSize)
        .Take(pageSize)
        .Select(e => new YourDto
        {
            Id = e.Id,
            Name = e.Name,
            RelatedName = e.RelatedEntity.Name  // EF handles JOIN
        })
        .ToListAsync(cancellationToken);

    return new PagedResult<YourDto>(items, totalCount, pageNumber, pageSize);
}
```

### Service Method Pattern
```csharp
public async Task<UserResponseDto?> GetUserByIdAsync(
    Guid id, 
    CancellationToken cancellationToken = default)
{
    _logger.LogInformation("Retrieving user {UserId}", id);

    var user = await _userRepository.GetByIdAsync(id, cancellationToken);
    
    if (user == null)
    {
        _logger.LogWarning("User {UserId} not found", id);
        return null;
    }

    return user;  // Repository already returns DTO
}
```

### Controller Method Pattern
```csharp
[HttpGet]
public async Task<IActionResult> GetProducts(
    [FromQuery] int pageNumber = 1,
    [FromQuery] int pageSize = 10,
    [FromQuery] string? search = null,
    CancellationToken cancellationToken = default)
{
    const int MaxPageSize = 100;
    pageSize = pageSize > MaxPageSize ? MaxPageSize : pageSize;
    pageNumber = pageNumber < 1 ? 1 : pageNumber;

    var result = await _productService.GetProductsPagedAsync(
        pageNumber,
        pageSize,
        search,
        cancellationToken);

    return Ok(result);
}
```

---

## 🚫 Never Do These

1. ❌ **Never** load entities then map to DTOs (`ToListAsync()` then `Select()`)
2. ❌ **Never** return unbounded lists (always paginate)
3. ❌ **Never** use tracking for read queries (always `AsNoTracking()`)
4. ❌ **Never** filter after `ToList()` (filter before with `Where()`)
5. ❌ **Never** return entities from repositories (return DTOs)
6. ❌ **Never** put business logic in controllers
7. ❌ **Never** use synchronous database calls (`Find()`, `.Result`, `.Wait()`)
8. ❌ **Never** ignore `CancellationToken` in async methods
9. ❌ **Never** catch exceptions without logging
10. ❌ **Never** return null for collections (return empty list)

---

## ✅ Always Do These

1. ✅ **Always** use async/await for database operations
2. ✅ **Always** pass `CancellationToken` to all async methods
3. ✅ **Always** use `AsNoTracking()` for read queries
4. ✅ **Always** project to DTOs in the database query
5. ✅ **Always** implement pagination for list endpoints
6. ✅ **Always** validate input with FluentValidation
7. ✅ **Always** log with structured logging (Serilog)
8. ✅ **Always** use dependency injection
9. ✅ **Always** handle exceptions with proper error responses
10. ✅ **Always** enforce maximum page size (50-100)

---

## 🔒 Security & Authentication

### JWT Configuration
```csharp
// Program.cs
builder.Services.AddAuthentication(JwtBearerDefaults.AuthenticationScheme)
    .AddJwtBearer(options =>
    {
        options.TokenValidationParameters = new TokenValidationParameters
        {
            ValidateIssuerSigningKey = true,
            IssuerSigningKey = new SymmetricSecurityKey(key),
            ValidateIssuer = true,
            ValidIssuer = jwtSettings.Issuer,
            ValidateAudience = true,
            ValidAudience = jwtSettings.Audience,
            ValidateLifetime = true,
            ClockSkew = TimeSpan.Zero
        };
    });
```

### Password Hashing
```csharp
// ✅ Always hash passwords
var user = new User
{
    Email = request.Email,
    PasswordHash = BCrypt.Net.BCrypt.HashPassword(request.Password)
};
```

---

## 📊 Error Handling

### Global Exception Middleware
```csharp
public class GlobalExceptionMiddleware
{
    public async Task InvokeAsync(HttpContext context, RequestDelegate next)
    {
        try
        {
            await next(context);
        }
        catch (NotFoundException ex)
        {
            await HandleExceptionAsync(context, ex, StatusCodes.Status404NotFound);
        }
        catch (ConflictException ex)
        {
            await HandleExceptionAsync(context, ex, StatusCodes.Status409Conflict);
        }
        catch (Exception ex)
        {
            _logger.LogError(ex, "Unhandled exception");
            await HandleExceptionAsync(context, ex, StatusCodes.Status500InternalServerError);
        }
    }
}
```

### Custom Exceptions
```csharp
public class NotFoundException : Exception
{
    public NotFoundException(string message) : base(message) { }
}

public class ConflictException : Exception
{
    public ConflictException(string message) : base(message) { }
}

public class BusinessException : Exception
{
    public BusinessException(string message) : base(message) { }
}
```

---

## 📝 Validation

### FluentValidation Example
```csharp
public class CreateUserRequestValidator : AbstractValidator<CreateUserRequestDto>
{
    public CreateUserRequestValidator()
    {
        RuleFor(x => x.Email)
            .NotEmpty().WithMessage("Email is required")
            .EmailAddress().WithMessage("Invalid email format")
            .MaximumLength(100);

        RuleFor(x => x.Password)
            .NotEmpty()
            .MinimumLength(8)
            .Matches(@"[A-Z]").WithMessage("Password must contain uppercase")
            .Matches(@"[a-z]").WithMessage("Password must contain lowercase")
            .Matches(@"[0-9]").WithMessage("Password must contain number");

        RuleFor(x => x.FirstName)
            .NotEmpty()
            .MaximumLength(50);
    }
}
```

---

## 🧪 Testing Standards

### Unit Test Pattern (XUnit + Moq)
```csharp
[Fact]
public async Task GetUserByIdAsync_WhenUserExists_ReturnsUser()
{
    // Arrange
    var userId = Guid.NewGuid();
    var expectedUser = new UserResponseDto { Id = userId };
    
    _mockRepository
        .Setup(r => r.GetByIdAsync(userId, It.IsAny<CancellationToken>()))
        .ReturnsAsync(expectedUser);

    // Act
    var result = await _userService.GetUserByIdAsync(userId);

    // Assert
    Assert.NotNull(result);
    Assert.Equal(userId, result.Id);
    _mockRepository.Verify(r => r.GetByIdAsync(userId, It.IsAny<CancellationToken>()), Times.Once);
}
```

---

## 📦 Required NuGet Packages

### API Project
- Microsoft.AspNetCore.Authentication.JwtBearer
- Swashbuckle.AspNetCore
- Serilog.AspNetCore
- FluentValidation.AspNetCore

### Infrastructure Project
- Microsoft.EntityFrameworkCore
- Microsoft.EntityFrameworkCore.SqlServer
- Microsoft.EntityFrameworkCore.Tools
- BCrypt.Net-Next

### Core Project
- FluentValidation

### Test Project
- xunit
- Moq
- FluentAssertions
- Microsoft.EntityFrameworkCore.InMemory

---

## ✅ Code Review Checklist

Before committing code, verify:

- [ ] All queries use `AsNoTracking()` for reads
- [ ] All queries project with `Select()` BEFORE `ToListAsync()`
- [ ] All list endpoints are paginated
- [ ] Maximum page size enforced (≤100)
- [ ] `CancellationToken` passed to all async methods
- [ ] No business logic in controllers
- [ ] DTOs used for all API responses
- [ ] Proper error handling with custom exceptions
- [ ] Structured logging in place
- [ ] Input validation with FluentValidation
- [ ] Unit tests for business logic
- [ ] No hardcoded secrets or connection strings

---

## 🎯 Summary

**Golden Rules**:
1. Project Early (Select before ToList)
2. Paginate Always
3. Track Never (AsNoTracking)
4. Filter on Database
5. DTO Not Entity
6. Cancel Always (CancellationToken)
7. Async All The Way
8. Inject, Don't New
9. Log Everything
10. Test Thoroughly

**Remember**: The database is 100x faster at filtering and projecting than C# code!

---

*For detailed examples and advanced patterns, refer to the specialized instruction files.*
