# 📝 Prompt Guidelines for .NET API Development
## How to Request Tasks Effectively

> **Purpose**: This guide helps you structure your requests to get the best possible code generation results. Follow these templates to ensure clear, actionable prompts.

---

## 🎯 The CARD Framework for Prompts

Every prompt should include:

**C** - **Context**: What are you building?  
**A** - **Action**: What do you want created?  
**R** - **Requirements**: What specific features/constraints?  
**D** - **Details**: Any additional specifications?

---

## 📋 Universal Prompt Template

```
I need to [ACTION] for [CONTEXT].

Requirements:
- [Requirement 1]
- [Requirement 2]
- [Requirement 3]

Additional Details:
- [Detail 1]
- [Detail 2]

Optional Reference:
- Similar to: [existing file/pattern]
- Based on: [documentation/example]
```

---

## ✅ Task-Specific Templates

### 1. Creating an Entity

**Template:**
```
Create a [EntityName] entity for [purpose].

Properties:
- [Property1]: [Type] - [Description]
- [Property2]: [Type] - [Description]
- [Property3]: [Type] - [Description]

Requirements:
- Inherit from BaseEntity (Id, CreatedAt, UpdatedAt, IsActive)
- Include [specific features, e.g., soft delete, audit fields]
- Add navigation properties to [RelatedEntity]

Constraints:
- [Field] must be unique
- [Field] is required/optional
- [Field] max length: [number]
```

**Example:**
```
Create a Product entity for managing inventory items.

Properties:
- Name: string - Product name
- SKU: string - Stock keeping unit
- Price: decimal - Unit price
- CategoryId: Guid - Foreign key to Category
- Stock: int - Current stock quantity
- Description: string - Product description

Requirements:
- Inherit from BaseEntity
- Include soft delete functionality
- Add navigation property to Category
- Add collection of OrderItems

Constraints:
- SKU must be unique
- Name is required, max length 200
- Price must be positive
- Stock cannot be negative
```

---

### 2. Creating a Repository

**Template:**
```
Create a [EntityName]Repository for [purpose].

Required Methods:
- [Method1]: [Description and signature]
- [Method2]: [Description and signature]

Query Requirements:
- Use AsNoTracking for read operations
- Project to [DtoName] where applicable
- Include pagination for list methods
- Support search/filter by [fields]

Performance Requirements:
- Maximum page size: [number]
- Include proper indexing hints
- Optimize for [specific scenario]

Optional Reference:
- Follow pattern from: [existing repository]
```

**Example:**
```
Create a ProductRepository for managing product data access.

Required Methods:
- GetPagedAsync: Returns paged list of ProductDto with search
- GetByIdAsync: Returns single ProductDto by ID
- GetBySKUAsync: Returns ProductDto by SKU
- GetByCategoryAsync: Returns products filtered by category with pagination
- AddAsync: Adds new product
- UpdateAsync: Updates existing product
- DeleteAsync: Soft deletes product

Query Requirements:
- Use AsNoTracking for all read operations
- Project to ProductDto in all queries
- Support search by Name, SKU, and Description
- Include Category name in projections
- Paginate all list results

Performance Requirements:
- Maximum page size: 50
- Default page size: 10
- Include stock availability calculations in queries

Optional Reference:
- Follow pattern from UserRepository in the codebase
```

---

### 3. Creating a Service

**Template:**
```
Create a [EntityName]Service implementing I[EntityName]Service for [business purpose].

Business Operations:
- [Operation1]: [Business logic description]
- [Operation2]: [Business logic description]

Validation Rules:
- [Rule 1]
- [Rule 2]
- [Rule 3]

Dependencies:
- I[Repository1]
- I[Repository2]
- I[Service]

Error Handling:
- Throw [ExceptionType] when [condition]
- Return null when [condition]
- Log [level] when [event occurs]

Optional Reference:
- Similar to: [existing service]
```

**Example:**
```
Create a ProductService implementing IProductService for managing product business logic.

Business Operations:
- CreateProductAsync: Validate SKU uniqueness, ensure positive price, check category exists
- UpdateProductAsync: Validate product exists, ensure price not changed if orders exist
- DeleteProductAsync: Soft delete only if no pending orders
- GetProductsPagedAsync: Return products with category names, filter by availability
- AdjustStockAsync: Update stock quantity, log changes, prevent negative stock

Validation Rules:
- SKU must be unique across all products
- Price must be greater than zero
- Name cannot be empty
- Category must exist
- Stock cannot be negative

Dependencies:
- IProductRepository
- ICategoryRepository
- ILogger<ProductService>

Error Handling:
- Throw ConflictException when SKU already exists
- Throw NotFoundException when product not found
- Throw BusinessException when trying to delete product with pending orders
- Log Information level for all CRUD operations
- Log Warning when validation fails

Optional Reference:
- Similar to UserService pattern
```

---

### 4. Creating a Controller

**Template:**
```
Create a [EntityName]Controller for [API purpose].

Endpoints Required:
- GET api/[route] - [Description]
- GET api/[route]/{id} - [Description]
- POST api/[route] - [Description]
- PUT api/[route]/{id} - [Description]
- DELETE api/[route]/{id} - [Description]

Request/Response Models:
- Use [RequestDto] for create
- Use [RequestDto] for update
- Return [ResponseDto] for all responses

Authorization:
- [Endpoint]: Requires [role/policy]
- [Endpoint]: Anonymous allowed

Validation:
- Maximum page size: [number]
- Required parameters: [list]

Optional Reference:
- Follow pattern from: [existing controller]
```

**Example:**
```
Create a ProductsController for product management API.

Endpoints Required:
- GET api/products - Get paged list with search and category filter
- GET api/products/{id} - Get single product by ID
- GET api/products/sku/{sku} - Get product by SKU
- GET api/products/category/{categoryId} - Get products by category (paged)
- POST api/products - Create new product
- PUT api/products/{id} - Update existing product
- DELETE api/products/{id} - Soft delete product
- PATCH api/products/{id}/stock - Adjust stock quantity

Request/Response Models:
- Use CreateProductRequestDto for POST
- Use UpdateProductRequestDto for PUT
- Use AdjustStockRequestDto for PATCH
- Return ProductResponseDto for all GET operations
- Return PagedResult<ProductResponseDto> for lists

Authorization:
- GET endpoints: [Authorize] - any authenticated user
- POST/PUT/DELETE: [Authorize(Roles = "Admin")]
- PATCH stock: [Authorize(Roles = "Admin, InventoryManager")]

Validation:
- Maximum page size: 50
- Page number minimum: 1
- Search term max length: 100

Optional Reference:
- Follow pattern from UsersController
```

---

### 5. Creating DTOs

**Template:**
```
Create DTOs for [EntityName] with the following:

Request DTOs:
- Create[EntityName]RequestDto: [fields needed for creation]
- Update[EntityName]RequestDto: [fields allowed for update]

Response DTOs:
- [EntityName]ResponseDto: [fields to return]
- [EntityName]ListItemDto: [minimal fields for lists]

Validation Requirements:
- [Field]: [validation rules]
- [Field]: [validation rules]

Computed Fields:
- [Field]: [how it's calculated]

Optional Reference:
- Similar to: [existing DTOs]
```

**Example:**
```
Create DTOs for Product management.

Request DTOs:
- CreateProductRequestDto: Name, SKU, Price, Description, CategoryId, InitialStock
- UpdateProductRequestDto: Name, Price, Description, CategoryId (SKU not updateable)
- AdjustStockRequestDto: ProductId, Quantity, Reason

Response DTOs:
- ProductResponseDto: Id, Name, SKU, Price, Description, CategoryName, Stock, IsAvailable, CreatedAt
- ProductListItemDto: Id, Name, SKU, Price, CategoryName, IsAvailable (for list views)
- ProductDetailDto: All fields plus OrderCount, LastOrderDate, LowStockWarning

Validation Requirements:
- Name: Required, MaxLength 200
- SKU: Required, MaxLength 50, Alphanumeric only
- Price: Required, Range 0.01 to 999999.99
- Description: MaxLength 1000
- CategoryId: Required, Must exist
- InitialStock/Quantity: Range 0 to 999999

Computed Fields:
- IsAvailable: Stock > 0
- LowStockWarning: Stock < ReorderLevel

Optional Reference:
- Similar to UserResponseDto structure
```

---

### 6. Creating Unit Tests

**Template:**
```
Create unit tests for [ClassName].[MethodName] with the following scenarios:

Test Scenarios:
1. [Scenario]: Expected result: [outcome]
2. [Scenario]: Expected result: [outcome]
3. [Scenario]: Expected result: [outcome]

Mocking Requirements:
- Mock [Dependency1]: Setup [behavior]
- Mock [Dependency2]: Setup [behavior]

Assertions:
- Verify [what]
- Assert [condition]

Optional Reference:
- Follow pattern from: [existing test file]
```

**Example:**
```
Create unit tests for ProductService.CreateProductAsync with the following scenarios:

Test Scenarios:
1. Valid product data: Should create product and return ProductResponseDto
2. Duplicate SKU: Should throw ConflictException
3. Invalid category: Should throw NotFoundException
4. Negative price: Should throw ValidationException
5. Empty name: Should throw ValidationException
6. Null request: Should throw ArgumentNullException

Mocking Requirements:
- Mock IProductRepository: 
  - GetBySKUAsync returns null for scenario 1
  - GetBySKUAsync returns existing product for scenario 2
  - AddAsync returns created product
- Mock ICategoryRepository:
  - ExistsAsync returns true for valid scenarios
  - ExistsAsync returns false for scenario 3
- Mock ILogger: Verify logging calls

Assertions:
- Verify repository methods called with correct parameters
- Verify repository AddAsync called exactly once on success
- Assert returned DTO has correct values
- Assert exceptions have correct messages
- Verify logger called with appropriate log levels

Optional Reference:
- Follow pattern from UserServiceTests
```

---

### 7. Creating Validation Rules

**Template:**
```
Create FluentValidation validator for [DtoName].

Validation Rules:
- [Field]: [Rules]
- [Field]: [Rules]

Custom Validations:
- [Custom rule description]

Error Messages:
- [Field]: "[Custom error message]"

Optional Reference:
- Similar to: [existing validator]
```

**Example:**
```
Create FluentValidation validator for CreateProductRequestDto.

Validation Rules:
- Name: NotEmpty, MaxLength 200
- SKU: NotEmpty, MaxLength 50, Must be alphanumeric
- Price: GreaterThan 0, LessThanOrEqual 999999.99, Must have max 2 decimal places
- Description: MaxLength 1000
- CategoryId: NotEmpty, Must be valid Guid
- InitialStock: GreaterThanOrEqual 0, LessThanOrEqual 999999

Custom Validations:
- SKU must match pattern: ^[A-Z0-9-]+$
- Category must exist in database (use async validation)
- Price must be in increments of 0.01

Error Messages:
- Name: "Product name is required and cannot exceed 200 characters"
- SKU: "SKU must be alphanumeric and contain only uppercase letters, numbers, and hyphens"
- Price: "Price must be between $0.01 and $999,999.99"
- CategoryId: "Invalid category selected"

Optional Reference:
- Similar to CreateUserRequestValidator
```

---

### 8. Creating Database Migration

**Template:**
```
Create a migration for [purpose].

Tables to Create/Modify:
- [TableName]: [Columns and constraints]

Indexes:
- [Index description]

Relationships:
- [Foreign key description]

Default Data:
- [Seed data if needed]

Optional Reference:
- Based on: [existing migration]
```

**Example:**
```
Create a migration for adding Products and Categories tables.

Tables to Create:
- Categories:
  - Id (Guid, PK)
  - Name (nvarchar(100), NOT NULL)
  - Description (nvarchar(500), NULL)
  - CreatedAt (datetime2, NOT NULL)
  - IsActive (bit, NOT NULL, default 1)
  
- Products:
  - Id (Guid, PK)
  - Name (nvarchar(200), NOT NULL)
  - SKU (nvarchar(50), NOT NULL, UNIQUE)
  - Price (decimal(18,2), NOT NULL)
  - Description (nvarchar(1000), NULL)
  - Stock (int, NOT NULL, default 0)
  - CategoryId (Guid, FK to Categories.Id)
  - CreatedAt (datetime2, NOT NULL)
  - UpdatedAt (datetime2, NULL)
  - DeletedAt (datetime2, NULL)
  - IsActive (bit, NOT NULL, default 1)

Indexes:
- IX_Products_SKU (unique index on SKU)
- IX_Products_CategoryId (non-clustered index)
- IX_Products_IsActive_DeletedAt (filtered index for active records)

Relationships:
- Products.CategoryId → Categories.Id (ON DELETE RESTRICT)

Default Data:
- Insert default categories: Electronics, Clothing, Books, Home & Garden

Optional Reference:
- Based on Users table migration pattern
```

---

## ❌ Bad Prompts vs ✅ Good Prompts

### Example 1: Creating a Repository

❌ **Bad Prompt:**
```
Create a product repository
```

✅ **Good Prompt:**
```
Create a ProductRepository implementing IProductRepository for managing product data access.

Required Methods:
- GetPagedAsync: Returns paged ProductDto with search by name/SKU
- GetByIdAsync: Returns ProductDto by ID with category name
- GetBySKUAsync: Returns ProductDto by SKU
- AddAsync, UpdateAsync, DeleteAsync (soft delete)

Requirements:
- Use AsNoTracking for all reads
- Project to ProductDto in all queries
- Include Category.Name in projections
- Paginate with max size 50
- Support search filtering

Follow the pattern from UserRepository.
```

---

### Example 2: Creating a Service

❌ **Bad Prompt:**
```
Make a service for orders
```

✅ **Good Prompt:**
```
Create an OrderService implementing IOrderService for order processing.

Business Operations:
- CreateOrderAsync: Validate customer, check product stock, calculate total, reserve stock
- UpdateOrderStatusAsync: Update status with validation, log status changes
- CancelOrderAsync: Only allow if status is Pending or Processing, restore stock

Validation:
- Customer must exist and be active
- All products must exist and have sufficient stock
- Order total must match calculated total
- Cannot cancel shipped orders

Dependencies:
- IOrderRepository
- IProductRepository
- ICustomerRepository
- ILogger<OrderService>

Error Handling:
- Throw NotFoundException when customer/product not found
- Throw BusinessException when insufficient stock
- Throw ConflictException when trying to cancel shipped order
```

---

### Example 3: Creating a Controller

❌ **Bad Prompt:**
```
Add API endpoints for categories
```

✅ **Good Prompt:**
```
Create a CategoriesController for category management.

Endpoints:
- GET api/categories - Paged list with search
- GET api/categories/{id} - Single category with product count
- POST api/categories - Create (Admin only)
- PUT api/categories/{id} - Update (Admin only)
- DELETE api/categories/{id} - Delete (Admin only, only if no products)

Request/Response:
- CreateCategoryRequestDto: Name, Description
- UpdateCategoryRequestDto: Name, Description
- CategoryResponseDto: Id, Name, Description, ProductCount, CreatedAt

Authorization:
- GET: Authenticated users
- POST/PUT/DELETE: Admin role required

Validation:
- Max page size: 100
- Name required, max 100 characters
```

---

## 🔍 Clarification Questions Checklist

When a prompt is unclear, ask these questions:

### For Entity Creation:
- [ ] What properties does this entity need?
- [ ] What are the data types and constraints?
- [ ] Should it inherit from BaseEntity?
- [ ] What relationships does it have with other entities?
- [ ] Are there any computed properties?
- [ ] What validation rules apply?

### For Repository Creation:
- [ ] What query methods are needed?
- [ ] Should results be projected to DTOs?
- [ ] Is pagination required?
- [ ] What search/filter capabilities are needed?
- [ ] Are there any complex queries with joins?
- [ ] What's the maximum page size?

### For Service Creation:
- [ ] What business operations are required?
- [ ] What validation rules need to be enforced?
- [ ] What dependencies are needed?
- [ ] How should errors be handled?
- [ ] What should be logged?
- [ ] Are there any transaction requirements?

### For Controller Creation:
- [ ] What HTTP endpoints are needed (GET, POST, PUT, DELETE)?
- [ ] What are the route patterns?
- [ ] What request/response DTOs should be used?
- [ ] What authorization is required?
- [ ] Are there any rate limiting requirements?
- [ ] What status codes should be returned?

### For Testing:
- [ ] What method/class needs to be tested?
- [ ] What scenarios should be covered?
- [ ] What dependencies need to be mocked?
- [ ] What assertions are required?
- [ ] Are there any edge cases?

---

## 📚 Reference Guidelines

### When to Provide References:

✅ **DO provide references when:**
- Following an existing pattern in the codebase
- Implementing similar functionality to existing code
- Using a specific architecture or design pattern
- Wanting consistency with team standards

❌ **DON'T provide references when:**
- It's a standard CRUD operation
- Following documented guidelines already
- Creating something completely new
- References would be confusing

### How to Provide References:

**Good Reference Examples:**
```
Optional Reference:
- Follow the pattern from UserRepository
- Similar to the authentication flow in AuthService
- Use the same error handling as OrderController
- Based on the Customer entity structure
```

**Bad Reference Examples:**
```
❌ "Like that other thing"
❌ "You know what I mean"
❌ "Similar to before"
```

---

## 🎯 Prompt Examples by Complexity

### Simple Prompts (Quick Tasks)

```
Create a Category entity with Id, Name (max 100), Description (max 500), 
inheriting from BaseEntity.
```

```
Add GetByCategoryIdAsync method to ProductRepository that returns 
paged ProductDto list filtered by categoryId.
```

```
Create unit test for UserService.DeleteUserAsync when user doesn't exist, 
should return false.
```

### Medium Prompts (Standard Tasks)

```
Create an OrderItemRepository with the following methods:

Methods:
- GetByOrderIdAsync: Returns list of OrderItemDto for a specific order
- GetByProductIdAsync: Returns paged orders containing specific product
- AddRangeAsync: Adds multiple order items in single transaction
- UpdateQuantityAsync: Updates quantity and recalculates total

Requirements:
- Use AsNoTracking for reads
- Project to OrderItemDto including Product.Name and Product.Price
- Include pagination for list methods (max 50 items)
- Calculate Subtotal as Quantity * UnitPrice in projection

Follow the pattern from ProductRepository.
```

### Complex Prompts (Feature Development)

```
Create complete order processing functionality:

1. Create Order Entity:
   - Properties: OrderNumber (auto-generated), CustomerId, OrderDate, 
     Status (enum), TotalAmount, ShippingAddress, Notes
   - Relationships: Customer (1:1), OrderItems (1:many)
   - Inherit from BaseEntity

2. Create OrderRepository:
   - GetPagedAsync: Filter by status, customer, date range with OrderDto projection
   - GetByOrderNumberAsync: Return detailed OrderDetailDto
   - GetCustomerOrdersAsync: Customer's orders paged and sorted by date
   - CreateWithItemsAsync: Create order with items in transaction
   - UpdateStatusAsync: Update status and log change

3. Create OrderService:
   - CreateOrderAsync: Validate customer, check stock, calculate total, 
     reserve inventory, generate order number
   - UpdateOrderStatusAsync: Validate status transition, update stock if cancelled,
     send notification
   - GetOrderDetailsAsync: Return complete order with items and customer info
   
   Validation Rules:
   - Customer must be active
   - All products must be in stock
   - Minimum order amount: $10
   - Cannot change status from Shipped to Cancelled
   
   Dependencies: IOrderRepository, IProductRepository, ICustomerRepository,
   IEmailService, ILogger

4. Create OrdersController:
   - GET api/orders - Admin sees all, customers see their own
   - GET api/orders/{id} - Details with authorization check
   - POST api/orders - Create order (authenticated customers)
   - PUT api/orders/{id}/status - Update status (admin only)
   - DELETE api/orders/{id} - Cancel order (customer if pending, admin always)

5. Create DTOs:
   - CreateOrderRequestDto with validation
   - OrderResponseDto
   - OrderDetailDto with items
   - OrderListItemDto for lists

6. Create Unit Tests:
   - Test all service methods with success and failure scenarios
   - Mock all dependencies
   - Cover edge cases

Follow N-tier architecture pattern, use PagedResult for lists, 
implement proper error handling with custom exceptions.
```

---

## 🤝 Interactive Prompt Refinement

### Initial Request Template:
```
What I want to build:
[Brief description]

Current understanding:
- [Aspect 1]
- [Aspect 2]

Questions I have:
- [Question 1]
- [Question 2]

Can you help me structure a proper prompt for this?
```

### Refinement Response Template:
Based on your description, here's what I need:

**Missing Information:**
- [ ] [What's missing]
- [ ] [What needs clarification]

**Suggested Prompt Structure:**
```
[Structured prompt based on templates above]
```

**Questions:**
1. [Clarifying question]
2. [Clarifying question]

---

## 📝 Prompt Quality Checklist

Before submitting your prompt, verify:

- [ ] **Action is clear**: What exactly should be created?
- [ ] **Context is provided**: What is this for?
- [ ] **Requirements are listed**: What must it include?
- [ ] **Constraints are specified**: What are the limitations?
- [ ] **Data types are defined**: What types for properties/parameters?
- [ ] **Validation rules are stated**: What validations are needed?
- [ ] **Dependencies are identified**: What does it depend on?
- [ ] **Error handling is addressed**: How to handle failures?
- [ ] **Performance considerations**: Any specific optimizations?
- [ ] **References are helpful**: Are examples provided if needed?

---

## 💡 Tips for Better Prompts

1. **Be Specific**: "Create ProductRepository with pagination" vs "Create repository"
2. **Include Details**: Specify data types, constraints, validation rules
3. **State Requirements**: Performance needs, business rules, dependencies
4. **Provide Context**: What this fits into, how it will be used
5. **Reference Standards**: "Following our N-tier pattern" or "Use PagedResult"
6. **Think Through**: List all methods, properties, scenarios before asking
7. **Use Examples**: Show existing code or patterns to follow
8. **Iterate**: Start with clarification if unsure, then refine

---

## 🎓 Learning Path

### Beginner: Start with Simple Prompts
```
Create a [Entity] with properties [list].
Add [Method] to [Repository/Service].
```

### Intermediate: Add Requirements
```
Create [Entity] with [properties] and [relationships].
Requirements: [specific needs]
Validation: [rules]
```

### Advanced: Complete Feature Prompts
```
Implement [feature] with:
- Entity: [details]
- Repository: [methods and requirements]
- Service: [business logic and validation]
- Controller: [endpoints and authorization]
- Tests: [scenarios]
Following [architecture/patterns].
```

---

## 🔄 Feedback Loop

After receiving generated code:

1. **Review**: Does it match requirements?
2. **Test**: Run and verify functionality
3. **Refine**: If not correct, provide specific feedback:
   ```
   The generated code doesn't [specific issue].
   Please update to [specific requirement].
   ```
4. **Learn**: Note what worked in the prompt for future use

---

## 📊 Prompt Effectiveness Matrix

| Prompt Type | Clarity | Detail | Success Rate |
|-------------|---------|--------|--------------|
| "Create repository" | ❌ Low | ❌ Low | 20% |
| "Create ProductRepository with CRUD" | ⚠️ Medium | ⚠️ Medium | 60% |
| "Create ProductRepository with paged queries, DTOs, search" | ✅ High | ✅ High | 95% |

**Aim for High Clarity + High Detail = High Success Rate**

---

## 🎯 Summary

### The Golden Rules of Prompts:

1. **Be Specific**: Say exactly what you want
2. **Provide Context**: Explain what it's for
3. **List Requirements**: Detail all needs
4. **Include Constraints**: Specify limitations
5. **Reference Standards**: Follow team patterns
6. **Verify Completeness**: Use the checklist
7. **Iterate**: Refine based on results

### Remember:
- ✅ Clear prompt = Clean code
- ✅ Detailed requirements = Correct implementation
- ✅ Proper structure = Faster results
- ✅ Good references = Consistent patterns

---

**Use these templates every time you request code generation, and you'll get better results consistently!**

---

*Last Updated: January 2025*
*Version: 1.0 - Comprehensive Prompt Guidelines*
