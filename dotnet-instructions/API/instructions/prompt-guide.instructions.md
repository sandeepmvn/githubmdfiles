---
name: Copilot Interaction Guide
description: How to write effective prompts for code generation
applyTo: ""
---

# 📝 How to Request Code from GitHub Copilot

## 🎯 The CARD Framework

Every prompt should include:

**C** - **Context**: What are you building?  
**A** - **Action**: What do you want created?  
**R** - **Requirements**: What specific features/constraints?  
**D** - **Details**: Any additional specifications?

---

## 📋 Quick Templates

### Creating an Entity
```
Create a [EntityName] entity for [purpose].

Properties:
- [Property1]: [Type] - [Description]
- [Property2]: [Type] - [Description]

Requirements:
- Inherit from BaseEntity
- Include [specific features]
- Navigation properties to [RelatedEntity]

Constraints:
- [Field] must be unique
- [Field] max length: [number]
```

**Example:**
```
Create a Product entity for managing inventory.

Properties:
- Name: string - Product name
- SKU: string - Stock keeping unit
- Price: decimal - Unit price
- CategoryId: Guid - Foreign key

Requirements:
- Inherit from BaseEntity
- Navigation to Category
- Collection of OrderItems

Constraints:
- SKU must be unique
- Name max length 200
- Price must be positive
```

---

### Creating a Repository
```
Create a [EntityName]Repository for [purpose].

Required Methods:
- GetPagedAsync: Paged list with search
- GetByIdAsync: Single item by ID
- AddAsync, UpdateAsync, DeleteAsync

Query Requirements:
- AsNoTracking for reads
- Project to [DtoName]
- Support search by [fields]

Performance:
- Max page size: [number]
```

---

### Creating a Service
```
Create a [EntityName]Service implementing I[EntityName]Service.

Business Operations:
- [Operation1]: [Logic description]
- [Operation2]: [Logic description]

Validation Rules:
- [Rule 1]
- [Rule 2]

Dependencies:
- I[Repository1], I[Repository2]

Error Handling:
- Throw [ExceptionType] when [condition]
```

---

### Creating a Controller
```
Create a [EntityName]Controller for [purpose].

Endpoints:
- GET api/[route] - [Description]
- POST api/[route] - [Description]
- PUT api/[route]/{id} - [Description]

Authorization:
- GET: [Authorize]
- POST/PUT/DELETE: [Authorize(Roles = "Admin")]

Validation:
- Max page size: [number]
```

---

### Creating DTOs
```
Create DTOs for [EntityName]:

Request DTOs:
- Create[Entity]RequestDto: [fields for creation]
- Update[Entity]RequestDto: [fields for update]

Response DTOs:
- [Entity]ResponseDto: [all fields]
- [Entity]ListItemDto: [minimal fields for lists]

Validation:
- [Field]: [rules]
```

---

## 💡 Complexity Levels

### Simple (One-liner)
```
Create a Category entity with Id, Name (max 100), Description (max 500), inheriting from BaseEntity.
```

### Medium (Structured)
```
Create a ProductRepository with:
- GetPagedAsync: Paged ProductDto with search
- GetByCategoryAsync: Filter by category
- Use AsNoTracking and projection
- Max page size: 50
```

### Complex (Full Feature)
```
Create order management feature with:
1. Order entity (OrderNumber, CustomerId, Status, TotalAmount)
2. OrderRepository (paged queries, projection to OrderDto)
3. OrderService (validation: customer active, products in stock)
4. OrdersController (CRUD with role-based auth)
5. DTOs with FluentValidation
6. Unit tests for service methods
```

---

## ✅ Prompt Quality Checklist

- [ ] Clear action (what to create)
- [ ] Context provided (what it's for)
- [ ] Requirements listed
- [ ] Data types defined
- [ ] Validation rules stated
- [ ] Dependencies identified
- [ ] Error handling addressed

---

## 🔥 Key Phrases for Best Results

**Use these phrases to ensure compliance:**

- "Following our N-tier architecture"
- "Use AsNoTracking for reads"
- "Project to DTOs in the query"
- "Include pagination with PagedResult"
- "Pass CancellationToken to all async methods"
- "Use FluentValidation for input validation"
- "Follow the pattern from [existing class]"
- "Maximum page size of [number]"
- "Include proper error handling with custom exceptions"
- "Add structured logging with ILogger"

---

## 📊 Effectiveness Matrix

| Prompt Type | Success Rate |
|-------------|--------------|
| "Create repository" | 20% ❌ |
| "Create ProductRepository with CRUD" | 60% ⚠️ |
| "Create ProductRepository with paged queries, DTOs, AsNoTracking, search by name" | 95% ✅ |

**Aim for High Detail = High Success**

---

## 🎓 Tips

1. **Be Specific**: "ProductRepository with pagination" vs "repository"
2. **Include Types**: Specify data types for properties/parameters
3. **State Rules**: Business rules, validation, constraints
4. **Reference Standards**: "Following N-tier pattern"
5. **Use Examples**: Reference existing code patterns
6. **Iterate**: Refine based on results

---

## ⚡ Quick Reference

**When asking for:**
- **Entity**: List properties, types, constraints, relationships
- **Repository**: Specify methods, queries, DTOs, performance requirements
- **Service**: Define business logic, validation rules, dependencies
- **Controller**: List endpoints, auth requirements, validation
- **DTOs**: Specify fields, validation rules, relationships

**Always mention:**
- Async/await required
- CancellationToken support
- Pagination requirements
- Performance optimizations (AsNoTracking, projection)
- Error handling approach
- Logging requirements

---

**Remember**: Clear prompt = Clean code!
