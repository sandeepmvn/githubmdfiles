---
name: LINQ Performance Rules
description: Critical LINQ and EF Core performance optimization rules
applyTo: "**/*.cs"
---

# 🔥 CRITICAL LINQ PERFORMANCE RULES

## NON-NEGOTIABLE RULES

### ⚡ RULE #1: Project to DTOs in Query (Never After)

```csharp
// ❌ NEVER - Loads all columns, maps in memory (10-100x slower)
var users = await _context.Users.ToListAsync();
var dtos = users.Select(u => new UserDto { Id = u.Id, Email = u.Email });

// ✅ ALWAYS - Database-side projection
var users = await _context.Users
    .Select(u => new UserDto { Id = u.Id, Email = u.Email })
    .ToListAsync(cancellationToken);
```

**Impact**: 10-100x faster, 90% less memory

---

### ⚡ RULE #2: Paginate Always (Never Load All)

```csharp
// ❌ NEVER - Memory explosion
var allUsers = await _context.Users.ToListAsync();
var page = allUsers.Skip((pageNumber - 1) * pageSize).Take(pageSize);

// ✅ ALWAYS - Database pagination
var users = await _context.Users
    .AsNoTracking()
    .OrderByDescending(u => u.CreatedAt)
    .Skip((pageNumber - 1) * pageSize)
    .Take(pageSize)
    .Select(u => new UserDto { ... })
    .ToListAsync(cancellationToken);
```

**Impact**: 1000-10000x less memory

---

### ⚡ RULE #3: AsNoTracking for Read-Only

```csharp
// ❌ DON'T - Tracking overhead
var users = await _context.Users.Where(u => u.IsActive).ToListAsync();

// ✅ DO - No tracking
var users = await _context.Users
    .AsNoTracking()
    .Where(u => u.IsActive)
    .Select(u => new UserDto { ... })
    .ToListAsync(cancellationToken);
```

---

## 📋 MANDATORY Query Pattern

**Every database query MUST follow this exact pattern:**

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

    // 6. SELECT to DTO (CRITICAL - before ToListAsync)
    var items = await query
        .Select(e => new YourDto
        {
            Id = e.Id,
            Name = e.Name,
            RelatedName = e.RelatedEntity.Name  // JOIN handled by EF
        })
        .ToListAsync(cancellationToken);

    // 7. Return paged result
    return new PagedResult<YourDto>(items, totalCount, pageNumber, pageSize);
}
```

**Order Matters:**
1. AsNoTracking
2. Where
3. CountAsync
4. OrderBy
5. Skip/Take
6. **Select ← BEFORE ToListAsync**
7. ToListAsync

---

## 🚫 NEVER DO

```csharp
// ❌ Loading all then filtering
var all = await _context.Users.ToListAsync();
var filtered = all.Where(u => u.IsActive);

// ❌ Mapping after query
var users = await _context.Users.ToListAsync();
var dtos = users.Select(u => new UserDto { ... });

// ❌ No pagination
public async Task<List<OrderDto>> GetAllOrders()
{
    return await _context.Orders.Select(o => new OrderDto { ... }).ToListAsync();
}

// ❌ N+1 queries
foreach (var order in orders)
{
    var customer = await _context.Customers.FindAsync(order.CustomerId);
}

// ❌ Synchronous calls
var user = _context.Users.Find(id);  // Use FindAsync!
var users = _context.Users.ToList();  // Use ToListAsync!
```

---

## ✅ ALWAYS DO

```csharp
// ✅ Single optimized query
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

// ✅ Include navigation properties in projection
.Select(p => new ProductDto
{
    Id = p.Id,
    Name = p.Name,
    CategoryName = p.Category.Name,  // EF creates JOIN
    SupplierName = p.Supplier.CompanyName  // Another JOIN
})
```

---

## 📊 Performance Comparison

| Method | Records | Memory | Time | Status |
|--------|---------|--------|------|--------|
| Load all, filter, map | 10,000 | 500MB | 2000ms | ❌ TERRIBLE |
| Load all, filter in query, map after | 10,000 | 500MB | 1800ms | ❌ BAD |
| Project in query, no pagination | 10,000 | 50MB | 800ms | ⚠️ OK |
| **Project + Paginate** | **10** | **5KB** | **50ms** | ✅ **EXCELLENT** |

**Result: 400x faster, 100,000x less memory!**

---

## ✅ Validation Checklist

Before committing LINQ code:

- [ ] Using `AsNoTracking()` for read-only queries?
- [ ] Projecting with `Select()` BEFORE `ToListAsync()`?
- [ ] Paginating with `Skip()` and `Take()`?
- [ ] Getting `CountAsync()` before pagination?
- [ ] Filtering with `Where()` on database (not after `ToList()`)?
- [ ] Passing `CancellationToken` to all async operations?
- [ ] Maximum page size enforced (e.g., 100)?
- [ ] Only fetching columns actually needed?
- [ ] No N+1 queries (using projections for joins)?
- [ ] Ordering before pagination?

**If you can't check ALL boxes, fix the query!**

---

## 🎯 Controller Pagination Enforcement

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

## 🔥 GOLDEN RULES (Memorize!)

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

## 📝 PagedResult Helper

```csharp
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

**REMEMBER: Let the database do the work, not your C# code!**
