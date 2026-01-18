# 🔥 CRITICAL ADDITION TO COPILOT INSTRUCTIONS 🔥

## ⚡ MANDATORY LINQ PERFORMANCE RULES

**Add this section to your copilot-instructions.md file. These rules are NON-NEGOTIABLE for performance.**

---

### 🚨 RULE #1: ALWAYS Project to DTOs in the Query (Never After)

```csharp
// ❌ NEVER DO THIS - Loads all columns, maps in memory
var users = await _context.Users.ToListAsync();
var dtos = users.Select(u => new UserDto { Id = u.Id, Email = u.Email });

// ✅ ALWAYS DO THIS - Only fetches needed columns
var users = await _context.Users
    .Select(u => new UserDto { Id = u.Id, Email = u.Email })
    .ToListAsync(cancellationToken);
```

**Performance Impact: 10-100x faster, 90% less memory**

---

### 🚨 RULE #2: ALWAYS Paginate (Never Load All Records)

```csharp
// ❌ NEVER DO THIS - Loads entire table
var allUsers = await _context.Users.ToListAsync();
var page = allUsers.Skip((pageNumber - 1) * pageSize).Take(pageSize);

// ✅ ALWAYS DO THIS - Database-side pagination
var users = await _context.Users
    .AsNoTracking()
    .OrderByDescending(u => u.CreatedAt)
    .Skip((pageNumber - 1) * pageSize)
    .Take(pageSize)
    .Select(u => new UserDto { ... })
    .ToListAsync(cancellationToken);
```

**Performance Impact: 1000-10000x less memory usage**

---

### 🚨 RULE #3: ALWAYS Use AsNoTracking for Read-Only Queries

```csharp
// ❌ DON'T - Tracking overhead for read-only data
var users = await _context.Users.Where(u => u.IsActive).ToListAsync();

// ✅ DO - No tracking overhead
var users = await _context.Users
    .AsNoTracking()
    .Where(u => u.IsActive)
    .Select(u => new UserDto { ... })
    .ToListAsync(cancellationToken);
```

---

### 📋 MANDATORY Query Pattern for ALL Database Queries

**Every repository method MUST follow this exact pattern:**

```csharp
public async Task<PagedResult<TDto>> GetPagedAsync(
    int pageNumber,
    int pageSize,
    string? searchTerm,
    CancellationToken cancellationToken = default)
{
    // 1. Start with AsNoTracking
    var query = _context.YourEntity.AsNoTracking();

    // 2. Apply WHERE filters
    if (!string.IsNullOrWhiteSpace(searchTerm))
    {
        query = query.Where(e => e.Name.Contains(searchTerm));
    }

    // 3. Get count BEFORE pagination
    var totalCount = await query.CountAsync(cancellationToken);

    // 4. ORDER BY (required for pagination)
    query = query.OrderByDescending(e => e.CreatedAt);

    // 5. SKIP and TAKE (pagination)
    query = query.Skip((pageNumber - 1) * pageSize).Take(pageSize);

    // 6. SELECT to DTO (CRITICAL - must be before ToListAsync)
    var items = await query
        .Select(e => new YourDto
        {
            Id = e.Id,
            Name = e.Name,
            // Only include fields you need!
            RelatedName = e.RelatedEntity.Name  // Handles JOIN
        })
        .ToListAsync(cancellationToken);

    // 7. Return paged result
    return new PagedResult<YourDto>(items, totalCount, pageNumber, pageSize);
}
```

**Order of operations matters! Always:**
1. AsNoTracking
2. Where
3. CountAsync
4. OrderBy
5. Skip/Take
6. **Select** ← BEFORE ToListAsync
7. ToListAsync

---

### 🎯 Repository Pattern with Projection

```csharp
// ✅ CORRECT - Repository returns DTOs with database projection
public async Task<PagedResult<ProductDto>> GetProductsAsync(
    int pageNumber,
    int pageSize,
    CancellationToken cancellationToken = default)
{
    var query = _context.Products.AsNoTracking();
    
    var totalCount = await query.CountAsync(cancellationToken);
    
    var items = await query
        .OrderBy(p => p.Name)
        .Skip((pageNumber - 1) * pageSize)
        .Take(pageSize)
        .Select(p => new ProductDto  // ← Projection in database
        {
            Id = p.Id,
            Name = p.Name,
            Price = p.Price,
            CategoryName = p.Category.Name  // JOIN handled by EF
        })
        .ToListAsync(cancellationToken);
    
    return new PagedResult<ProductDto>(items, totalCount, pageNumber, pageSize);
}

// ❌ WRONG - Repository returns entities, service maps
public async Task<IEnumerable<Product>> GetProductsAsync()
{
    return await _context.Products.ToListAsync();  // Loads all columns!
}
```

---

### 🚫 NEVER DO These

```csharp
// ❌ NEVER - Loading all then filtering in memory
var all = await _context.Users.ToListAsync();
var filtered = all.Where(u => u.IsActive);

// ❌ NEVER - Mapping after database query
var users = await _context.Users.ToListAsync();
var dtos = users.Select(u => new UserDto { ... });

// ❌ NEVER - No pagination on lists
public async Task<List<OrderDto>> GetAllOrders()
{
    return await _context.Orders.Select(o => new OrderDto { ... }).ToListAsync();
}

// ❌ NEVER - N+1 queries in loop
foreach (var order in orders)
{
    var customer = await _context.Customers.FindAsync(order.CustomerId);
}
```

---

### ✅ ALWAYS Do These

```csharp
// ✅ ALWAYS - Single query with projection and pagination
var result = await _context.Orders
    .AsNoTracking()
    .Where(o => o.Status == OrderStatus.Pending)
    .OrderByDescending(o => o.OrderDate)
    .Skip((pageNumber - 1) * pageSize)
    .Take(pageSize)
    .Select(o => new OrderDto
    {
        Id = o.Id,
        OrderNumber = o.OrderNumber,
        CustomerName = o.Customer.Name,  // Automatic JOIN
        TotalAmount = o.OrderItems.Sum(oi => oi.Total)  // Calculated in DB
    })
    .ToListAsync(cancellationToken);

// ✅ ALWAYS - Include PagedResult helper
public class PagedResult<T>
{
    public List<T> Items { get; set; }
    public int TotalCount { get; set; }
    public int PageNumber { get; set; }
    public int PageSize { get; set; }
    public int TotalPages => (int)Math.Ceiling(TotalCount / (double)PageSize);
    public bool HasNext => PageNumber < TotalPages;
    public bool HasPrevious => PageNumber > 1;

    public PagedResult(List<T> items, int totalCount, int pageNumber, int pageSize)
    {
        Items = items;
        TotalCount = totalCount;
        PageNumber = pageNumber;
        PageSize = pageSize;
    }
}
```

---

### 📊 Performance Comparison

| Method | Records | Memory | Time | Status |
|--------|---------|--------|------|--------|
| Load all, filter, map | 10,000 | 500MB | 2000ms | ❌ TERRIBLE |
| Load all, filter in query, map after | 10,000 | 500MB | 1800ms | ❌ BAD |
| Project in query, no pagination | 10,000 | 50MB | 800ms | ⚠️ OK |
| **Project + Paginate** | **10** | **5KB** | **50ms** | ✅ **EXCELLENT** |

**Result: 400x faster, 100,000x less memory!**

---

### 🎓 Validation Checklist

Before committing ANY code with database queries, verify:

- [ ] Using `AsNoTracking()` for read-only queries?
- [ ] Projecting with `Select()` BEFORE `ToListAsync()`?
- [ ] Paginating with `Skip()` and `Take()`?
- [ ] Getting `CountAsync()` before pagination?
- [ ] Filtering with `Where()` on database (not after `ToList()`)?
- [ ] Passing `CancellationToken` to all async operations?
- [ ] Maximum page size enforced (e.g., 100)?
- [ ] Only fetching columns actually needed?
- [ ] No N+1 queries (using proper projections for joins)?
- [ ] Ordering before pagination?

**If you can't check ALL boxes, your query needs fixing!**

---

### 🎯 Controller Validation Example

```csharp
[HttpGet]
public async Task<IActionResult> GetProducts(
    [FromQuery] int pageNumber = 1,
    [FromQuery] int pageSize = 10,
    [FromQuery] string? search = null,
    CancellationToken cancellationToken = default)
{
    // Enforce limits
    const int MaxPageSize = 100;
    if (pageSize > MaxPageSize) pageSize = MaxPageSize;
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

## 🔥 GOLDEN RULES (Memorize These!)

1. **Project Early** - `Select()` before `ToListAsync()`, never after
2. **Paginate Always** - Never return unbounded lists
3. **Track Never** - Use `AsNoTracking()` for all read queries
4. **Filter on Database** - `Where()` before `ToList()`, never after
5. **Count Before Paginate** - Get `TotalCount` before `Skip/Take`
6. **Order Before Paginate** - Always `OrderBy()` before pagination
7. **DTO Not Entity** - Repository returns DTOs, not entities
8. **Cancel Always** - Pass `CancellationToken` to all async methods
9. **Limit Page Size** - Enforce maximum (50-100 records)
10. **Join in Projection** - Use navigation properties in `Select()`

---

**REMEMBER: The database is 100x faster at filtering and projecting than your application code!**

**Let the database do the work - not your C# code!**

---

## 📝 Add This to Your Code Reviews

```markdown
### LINQ Performance Check
- [ ] All queries use projection (.Select() before .ToListAsync())
- [ ] All list endpoints are paginated
- [ ] AsNoTracking() used for read-only queries
- [ ] No .ToList() followed by .Where() or .Select()
- [ ] Maximum page size enforced (≤100)
- [ ] CancellationToken passed to all async methods
```

---

**END OF CRITICAL LINQ GUIDELINES**

*Add these rules to your main copilot-instructions.md file and ensure all team members follow them!*
