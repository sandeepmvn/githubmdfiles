# LINQ Performance Guidelines - Enhanced Section

## ⚡ CRITICAL PERFORMANCE RULES

### Rule #1: ALWAYS Use Projection (Select) - Never Load Full Entities
**This is the #1 performance killer in .NET applications!**

```csharp
// ❌ TERRIBLE - Loads ALL columns from database, then filters in memory
var users = await _context.Users.ToListAsync();
var userDtos = users.Select(u => new UserDto 
{
    Id = u.Id,
    Email = u.Email,
    FullName = $"{u.FirstName} {u.LastName}"
}).ToList();

// ✅ EXCELLENT - Only fetches required columns from database
var users = await _context.Users
    .AsNoTracking()
    .Select(u => new UserDto 
    {
        Id = u.Id,
        Email = u.Email,
        FullName = $"{u.FirstName} {u.LastName}"
    })
    .ToListAsync(cancellationToken);
```

**Performance Impact:**
- ❌ Bad way: Fetches 20+ columns, 500KB of data
- ✅ Good way: Fetches 3 columns, 50KB of data
- **Result: 10x faster, 90% less memory**

---

### Rule #2: ALWAYS Paginate - Never Load All Records

```csharp
// ❌ TERRIBLE - Loads entire table into memory
var allOrders = await _context.Orders.ToListAsync();
var pagedOrders = allOrders.Skip((pageNumber - 1) * pageSize).Take(pageSize);

// ✅ EXCELLENT - Server-side pagination
var orders = await _context.Orders
    .AsNoTracking()
    .OrderByDescending(o => o.CreatedAt)
    .Skip((pageNumber - 1) * pageSize)
    .Take(pageSize)
    .Select(o => new OrderDto
    {
        Id = o.Id,
        OrderNumber = o.OrderNumber,
        TotalAmount = o.TotalAmount,
        Status = o.Status
    })
    .ToListAsync(cancellationToken);
```

**Performance Impact:**
- ❌ Bad way: 10,000 records × 50KB = 500MB loaded into memory!
- ✅ Good way: 10 records × 5KB = 50KB loaded
- **Result: 10,000x less memory usage**

---

## 🚀 Complete LINQ Query Pattern (MANDATORY)

**Every query MUST follow this pattern:**

```csharp
public async Task<PagedResult<ProductDto>> GetProductsAsync(
    int pageNumber,
    int pageSize,
    string? searchTerm,
    CancellationToken cancellationToken = default)
{
    // 1. Start with AsNoTracking (for read-only queries)
    var query = _context.Products.AsNoTracking();

    // 2. Apply filters (WHERE clauses)
    if (!string.IsNullOrWhiteSpace(searchTerm))
    {
        query = query.Where(p => p.Name.Contains(searchTerm) || 
                                 p.Description.Contains(searchTerm));
    }

    // 3. Get total count (before pagination)
    var totalCount = await query.CountAsync(cancellationToken);

    // 4. Apply sorting
    query = query.OrderByDescending(p => p.CreatedAt);

    // 5. Apply pagination
    query = query.Skip((pageNumber - 1) * pageSize).Take(pageSize);

    // 6. PROJECT TO DTO (Select) - CRITICAL FOR PERFORMANCE
    var items = await query
        .Select(p => new ProductDto
        {
            Id = p.Id,
            Name = p.Name,
            Price = p.Price,
            CategoryName = p.Category.Name,  // Navigation property
            IsAvailable = p.Stock > 0
        })
        .ToListAsync(cancellationToken);

    // 7. Return paged result
    return new PagedResult<ProductDto>(items, totalCount, pageNumber, pageSize);
}
```

**Order matters:**
1. AsNoTracking
2. Where (filters)
3. CountAsync (total count)
4. OrderBy (sorting)
5. Skip/Take (pagination)
6. **Select (projection) - MUST be before ToListAsync**
7. ToListAsync

---

## 📊 Real-World Performance Comparisons

### Example 1: User List with 10,000 Records

```csharp
// ❌ WORST WAY - Memory: ~500MB, Time: ~2000ms
var allUsers = await _context.Users.ToListAsync();
var result = allUsers
    .Where(u => u.IsActive)
    .Skip((page - 1) * 10)
    .Take(10)
    .Select(u => new UserDto { Id = u.Id, Email = u.Email })
    .ToList();

// ⚠️ BETTER BUT STILL BAD - Memory: ~500MB, Time: ~1800ms
var users = await _context.Users
    .Where(u => u.IsActive)
    .Skip((page - 1) * 10)
    .Take(10)
    .ToListAsync();
var result = users.Select(u => new UserDto { Id = u.Id, Email = u.Email });

// ✅ BEST WAY - Memory: ~5KB, Time: ~50ms
var result = await _context.Users
    .AsNoTracking()
    .Where(u => u.IsActive)
    .Skip((page - 1) * 10)
    .Take(10)
    .Select(u => new UserDto 
    { 
        Id = u.Id, 
        Email = u.Email 
    })
    .ToListAsync(cancellationToken);
```

**Performance Difference: 40x faster, 100x less memory!**

---

### Example 2: Complex Query with Joins

```csharp
// ❌ BAD - Loads everything, filters in memory
var orders = await _context.Orders
    .Include(o => o.Customer)
    .Include(o => o.OrderItems)
        .ThenInclude(oi => oi.Product)
    .ToListAsync();

var result = orders
    .Where(o => o.Status == OrderStatus.Completed)
    .Select(o => new OrderDto
    {
        OrderNumber = o.OrderNumber,
        CustomerName = o.Customer.Name,
        TotalItems = o.OrderItems.Count,
        TotalAmount = o.OrderItems.Sum(oi => oi.Quantity * oi.UnitPrice)
    });

// ✅ GOOD - Filters and projects on database
var result = await _context.Orders
    .AsNoTracking()
    .Where(o => o.Status == OrderStatus.Completed)
    .Skip((pageNumber - 1) * pageSize)
    .Take(pageSize)
    .Select(o => new OrderDto
    {
        OrderNumber = o.OrderNumber,
        CustomerName = o.Customer.Name,
        TotalItems = o.OrderItems.Count,
        TotalAmount = o.OrderItems.Sum(oi => oi.Quantity * oi.UnitPrice)
    })
    .ToListAsync(cancellationToken);
```

**SQL Generated:**
- ❌ Bad: Fetches ALL orders with ALL related data (MBs of data)
- ✅ Good: Single query with JOIN, calculated fields, filters (KBs of data)

---

## 🎯 Repository Pattern with Projection

### ❌ WRONG WAY - Repository Returns Entities

```csharp
// DON'T DO THIS
public async Task<IEnumerable<User>> GetAllUsersAsync()
{
    return await _context.Users.ToListAsync();  // Returns full entities
}

// Service layer
public async Task<IEnumerable<UserDto>> GetUsersAsync()
{
    var users = await _userRepository.GetAllUsersAsync();  // Loads everything
    return users.Select(u => new UserDto { ... });  // Maps in memory
}
```

### ✅ RIGHT WAY - Repository Returns DTOs with Projection

```csharp
// DO THIS
public async Task<PagedResult<UserDto>> GetUsersPagedAsync(
    int pageNumber,
    int pageSize,
    string? searchTerm,
    CancellationToken cancellationToken = default)
{
    var query = _context.Users.AsNoTracking();

    // Apply search filter
    if (!string.IsNullOrWhiteSpace(searchTerm))
    {
        query = query.Where(u => 
            u.Email.Contains(searchTerm) || 
            u.FirstName.Contains(searchTerm) ||
            u.LastName.Contains(searchTerm));
    }

    var totalCount = await query.CountAsync(cancellationToken);

    // PROJECT TO DTO in database query
    var users = await query
        .OrderByDescending(u => u.CreatedAt)
        .Skip((pageNumber - 1) * pageSize)
        .Take(pageSize)
        .Select(u => new UserDto
        {
            Id = u.Id,
            Email = u.Email,
            FirstName = u.FirstName,
            LastName = u.LastName,
            FullName = $"{u.FirstName} {u.LastName}",
            IsActive = u.IsActive
        })
        .ToListAsync(cancellationToken);

    return new PagedResult<UserDto>(users, totalCount, pageNumber, pageSize);
}
```

---

## 📋 Mandatory Patterns for All Queries

### Pattern 1: Simple List Query

```csharp
public async Task<IEnumerable<ProductDto>> GetProductsAsync(
    CancellationToken cancellationToken = default)
{
    return await _context.Products
        .AsNoTracking()
        .Where(p => p.IsActive)
        .OrderBy(p => p.Name)
        .Select(p => new ProductDto  // ← ALWAYS SELECT TO DTO
        {
            Id = p.Id,
            Name = p.Name,
            Price = p.Price
        })
        .ToListAsync(cancellationToken);
}
```

### Pattern 2: Paged Query with Search

```csharp
public async Task<PagedResult<ProductDto>> GetProductsPagedAsync(
    int pageNumber,
    int pageSize,
    string? searchTerm,
    CancellationToken cancellationToken = default)
{
    var query = _context.Products
        .AsNoTracking()
        .Where(p => p.IsActive);

    // Search filter
    if (!string.IsNullOrWhiteSpace(searchTerm))
    {
        query = query.Where(p => p.Name.Contains(searchTerm));
    }

    // Count before pagination
    var totalCount = await query.CountAsync(cancellationToken);

    // Paginate and project
    var items = await query
        .OrderBy(p => p.Name)
        .Skip((pageNumber - 1) * pageSize)
        .Take(pageSize)
        .Select(p => new ProductDto  // ← PROJECTION BEFORE ToListAsync
        {
            Id = p.Id,
            Name = p.Name,
            Price = p.Price,
            CategoryName = p.Category.Name
        })
        .ToListAsync(cancellationToken);

    return new PagedResult<ProductDto>(items, totalCount, pageNumber, pageSize);
}
```

### Pattern 3: Query with Related Data

```csharp
public async Task<OrderDto?> GetOrderDetailsAsync(
    Guid orderId,
    CancellationToken cancellationToken = default)
{
    return await _context.Orders
        .AsNoTracking()
        .Where(o => o.Id == orderId)
        .Select(o => new OrderDto  // ← PROJECT including navigation properties
        {
            Id = o.Id,
            OrderNumber = o.OrderNumber,
            CustomerName = o.Customer.Name,  // Navigation property
            CustomerEmail = o.Customer.Email,
            OrderDate = o.OrderDate,
            TotalAmount = o.TotalAmount,
            Items = o.OrderItems.Select(oi => new OrderItemDto  // Nested projection
            {
                ProductId = oi.ProductId,
                ProductName = oi.Product.Name,
                Quantity = oi.Quantity,
                UnitPrice = oi.UnitPrice,
                Subtotal = oi.Quantity * oi.UnitPrice
            }).ToList()
        })
        .FirstOrDefaultAsync(cancellationToken);
}
```

---

## 🚫 NEVER DO THESE

### ❌ Loading Full Entity Then Mapping
```csharp
// NEVER DO THIS
var user = await _context.Users.FirstOrDefaultAsync(u => u.Id == id);
if (user == null) return null;
return new UserDto { Id = user.Id, Email = user.Email };  // Maps in memory
```

### ❌ ToList() Before Where
```csharp
// NEVER DO THIS
var allUsers = await _context.Users.ToListAsync();  // Loads everything
var activeUsers = allUsers.Where(u => u.IsActive);  // Filters in memory
```

### ❌ No Pagination on Large Tables
```csharp
// NEVER DO THIS
public async Task<List<OrderDto>> GetAllOrdersAsync()
{
    return await _context.Orders
        .Select(o => new OrderDto { ... })
        .ToListAsync();  // Could load 100,000+ records!
}
```

### ❌ Multiple Database Hits in Loop
```csharp
// NEVER DO THIS
var orders = await _context.Orders.ToListAsync();
foreach (var order in orders)
{
    var customer = await _context.Customers.FindAsync(order.CustomerId);  // N+1 problem
    order.CustomerName = customer.Name;
}
```

---

## ✅ Always Do These

### ✅ Single Query with Projection
```csharp
var result = await _context.Orders
    .AsNoTracking()
    .Where(o => o.Status == OrderStatus.Pending)
    .Skip((pageNumber - 1) * pageSize)
    .Take(pageSize)
    .Select(o => new OrderDto
    {
        Id = o.Id,
        CustomerName = o.Customer.Name,  // JOIN handled by EF Core
        OrderDate = o.OrderDate
    })
    .ToListAsync(cancellationToken);
```

### ✅ Check Count Before Loading
```csharp
// First check if any records exist
var hasRecords = await _context.Orders.AnyAsync(o => o.Status == status, cancellationToken);
if (!hasRecords)
{
    return new PagedResult<OrderDto>(new List<OrderDto>(), 0, pageNumber, pageSize);
}

// Then load with projection
var result = await _context.Orders
    .AsNoTracking()
    .Where(o => o.Status == status)
    .Skip((pageNumber - 1) * pageSize)
    .Take(pageSize)
    .Select(o => new OrderDto { ... })
    .ToListAsync(cancellationToken);
```

---

## 🎓 PagedResult Helper Class

```csharp
public class PagedResult<T>
{
    public List<T> Items { get; set; }
    public int PageNumber { get; set; }
    public int PageSize { get; set; }
    public int TotalCount { get; set; }
    public int TotalPages => (int)Math.Ceiling(TotalCount / (double)PageSize);
    public bool HasPreviousPage => PageNumber > 1;
    public bool HasNextPage => PageNumber < TotalPages;

    public PagedResult(List<T> items, int totalCount, int pageNumber, int pageSize)
    {
        Items = items;
        TotalCount = totalCount;
        PageNumber = pageNumber;
        PageSize = pageSize;
    }
}

// Usage in controller
[HttpGet]
public async Task<IActionResult> GetProducts(
    [FromQuery] int pageNumber = 1,
    [FromQuery] int pageSize = 10,
    [FromQuery] string? search = null,
    CancellationToken cancellationToken = default)
{
    // Validate page size
    if (pageSize > 100) pageSize = 100;  // Max 100 items per page
    if (pageSize < 1) pageSize = 10;
    if (pageNumber < 1) pageNumber = 1;

    var result = await _productService.GetProductsPagedAsync(
        pageNumber,
        pageSize,
        search,
        cancellationToken);

    return Ok(result);
}
```

---

## 📊 Performance Monitoring

### Log Query Performance
```csharp
public async Task<PagedResult<UserDto>> GetUsersPagedAsync(
    int pageNumber,
    int pageSize,
    CancellationToken cancellationToken = default)
{
    var stopwatch = System.Diagnostics.Stopwatch.StartNew();

    var result = await _context.Users
        .AsNoTracking()
        .Skip((pageNumber - 1) * pageSize)
        .Take(pageSize)
        .Select(u => new UserDto { ... })
        .ToListAsync(cancellationToken);

    stopwatch.Stop();
    
    _logger.LogInformation(
        "Query executed in {ElapsedMs}ms - Page: {PageNumber}, Size: {PageSize}",
        stopwatch.ElapsedMilliseconds,
        pageNumber,
        pageSize);

    return result;
}
```

### Enable Query Logging (Development Only)
```csharp
// Program.cs - Development environment only
if (builder.Environment.IsDevelopment())
{
    builder.Services.AddDbContext<ApplicationDbContext>(options =>
        options.UseSqlServer(connectionString)
               .EnableSensitiveDataLogging()
               .LogTo(Console.WriteLine, LogLevel.Information));
}
```

---

## 🎯 Summary Checklist

Before writing ANY LINQ query, ask yourself:

- [ ] Am I using `AsNoTracking()` for read-only queries?
- [ ] Am I projecting to DTO with `Select()` BEFORE `ToListAsync()`?
- [ ] Am I paginating with `Skip()` and `Take()`?
- [ ] Am I getting `CountAsync()` before pagination?
- [ ] Am I filtering with `Where()` on the database, not in memory?
- [ ] Am I avoiding N+1 queries by using proper projections?
- [ ] Am I passing `CancellationToken` to all async methods?
- [ ] Is my query fetching only the columns I need?
- [ ] Have I set a maximum page size limit (e.g., 100)?
- [ ] Am I logging slow queries for monitoring?

**If you answered NO to any question, fix it!**

---

## ⚡ Performance Targets

| Metric | Target | Red Flag |
|--------|--------|----------|
| Query execution time | < 100ms | > 1000ms |
| Records per page | 10-50 | > 100 |
| Memory per query | < 1MB | > 10MB |
| Database columns fetched | Only needed | All columns |
| N+1 queries | 0 | > 0 |

---

**REMEMBER: The database is ALWAYS faster at filtering and projecting than your application code!**

**GOLDEN RULE: Project early, paginate always, track never (for reads)!**
