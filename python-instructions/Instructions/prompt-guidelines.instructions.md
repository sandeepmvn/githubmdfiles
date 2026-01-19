---
name: Prompt Guidelines for FastAPI
description: How to structure effective prompts for FastAPI development
applyTo: ""
---

# Prompt Guidelines for FastAPI Development

## The CARD Framework

Every prompt should include:

**C** - **Context**: What are you building?  
**A** - **Action**: What do you want created?  
**R** - **Requirements**: What specific features/constraints?  
**D** - **Details**: Any additional specifications?

---

## Quick Templates

### Creating a Model (SQLAlchemy)

```
Create a [ModelName] model for [purpose].

Fields:
- [field_name]: [type] - [description]
- [field_name]: [type] - [description]

Requirements:
- Inherit from Base
- Include timestamps (created_at, updated_at)
- Add relationships to [RelatedModel]
- Add indexes on [field names]

Constraints:
- [field] must be unique
- [field] max length: [number]
- [field] default value: [value]
```

**Example:**
```
Create a Product model for e-commerce inventory.

Fields:
- name: string - Product name
- sku: string - Stock keeping unit
- price: decimal - Unit price (2 decimal places)
- stock: integer - Available quantity
- category_id: integer - Foreign key to Category
- is_active: boolean - Product availability status

Requirements:
- Inherit from Base
- Include created_at and updated_at timestamps
- Add relationship to Category
- Add index on sku and category_id

Constraints:
- sku must be unique
- name max length 200
- price must be positive
- stock cannot be negative
```

---

### Creating a Pydantic Schema

```
Create Pydantic schemas for [EntityName]:

Base Schema:
- Shared fields: [list fields]

Create Schema (for POST):
- Additional fields: [list fields]
- Validation: [list rules]

Update Schema (for PUT/PATCH):
- All fields optional except: [list required]

Response Schema:
- Include: [list all fields]
- Exclude: [sensitive fields]
- Config: from_attributes = True

List Response Schema:
- Paginated with total count
```

**Example:**
```
Create Pydantic schemas for User:

Base Schema:
- email: EmailStr
- full_name: str (1-100 chars)
- is_active: bool (default True)

Create Schema:
- password: str (8-100 chars, strong validation)

Update Schema:
- All fields optional

Response Schema:
- Include all base fields plus id, created_at, updated_at
- Exclude hashed_password
- Config: from_attributes = True

List Response Schema:
- Paginated with items, total, page, page_size, total_pages
```

---

### Creating a Repository

```
Create a [EntityName]Repository inheriting from BaseRepository.

Custom Methods:
- [method_name]: [description]
- [method_name]: [description]

Query Features:
- Support search by [fields]
- Support filtering by [conditions]
- Return [DTOName] not entities
- Use async/await
- Include pagination

Performance:
- Use select() statements
- Add proper indexes
- Eager load [relationships]
```

**Example:**
```
Create a ProductRepository inheriting from BaseRepository.

Custom Methods:
- get_by_sku: Find product by SKU
- get_active_products: Get all active products
- get_by_category: Filter by category with pagination
- search_products: Search by name or description

Query Features:
- Support search by name (case-insensitive)
- Support filtering by category_id, is_active
- Use select() statements (not query())
- Include eager loading for category relationship
- All queries async

Performance:
- Add index hints where beneficial
- Use proper pagination (offset/limit)
- Return count for paginated queries
```

---

### Creating a Service

```
Create a [EntityName]Service for business logic.

Dependencies:
- [Repository1], [Repository2]
- Database session

Methods:
- [method_name]: [description and business rules]
- [method_name]: [description and business rules]

Validation Rules:
- [rule 1]
- [rule 2]

Error Handling:
- Raise [ExceptionType] when [condition]
- Log all errors

Additional Features:
- [feature description]
```

**Example:**
```
Create a ProductService for product management.

Dependencies:
- ProductRepository
- CategoryRepository
- Database session

Methods:
- get_product_by_id: Get single product, raise NotFoundException if not found
- get_products: Get paginated list with optional search and category filter
- create_product: Create new product
  - Verify category exists
  - Verify SKU is unique
  - Validate price is positive
- update_product: Update existing product
  - Verify product exists
  - Verify new SKU is unique if changed
- delete_product: Delete product
  - Check if product has active orders
  - Raise BadRequestException if has orders

Validation Rules:
- Price must be positive
- Stock cannot be negative
- SKU must be unique
- Category must exist

Error Handling:
- Raise NotFoundException when product not found
- Raise ConflictException for duplicate SKU
- Raise BadRequestException for invalid data
- Log all operations with user context
```

---

### Creating API Endpoints

```
Create API endpoints for [EntityName] at [route].

Endpoints:
- GET [route]: List all with pagination
- GET [route]/{id}: Get single item
- POST [route]: Create new item
- PUT [route]/{id}: Update item
- DELETE [route]/{id}: Delete item

Authentication:
- All endpoints require authentication
- [Specific roles for specific endpoints]

Request/Response:
- Use [Schema] for requests
- Return [Schema] for responses
- Include proper status codes

Validation:
- [Validation requirements]

Documentation:
- Add endpoint descriptions
- Document all parameters
- Show example responses
```

**Example:**
```
Create API endpoints for Product at /api/v1/products.

Endpoints:
- GET /api/v1/products: List products with pagination, search, category filter
- GET /api/v1/products/{id}: Get single product by ID
- POST /api/v1/products: Create new product (admin only)
- PUT /api/v1/products/{id}: Update product (admin only)
- DELETE /api/v1/products/{id}: Delete product (admin only)

Authentication:
- All GET endpoints: any authenticated user
- POST/PUT/DELETE: require superuser role

Request/Response:
- POST: ProductCreate schema
- PUT: ProductUpdate schema
- GET: ProductResponse schema
- List: ProductListResponse with pagination

Query Parameters (for list):
- page: int (default 1)
- page_size: int (default 10, max 100)
- search: optional string
- category_id: optional int
- is_active: optional bool

Validation:
- Enforce max page size of 100
- Validate category_id exists

Documentation:
- Add docstrings to all endpoints
- Document query parameters
- Show 200, 404, 422, 401, 403 responses
```

---

## Complexity Levels

### Simple (One-liner)
```
Create a Category model with id, name (max 100, unique), description (max 500), is_active (default True), and timestamps.
```

### Medium (Structured)
```
Create a ProductRepository with:
- get_by_sku: async method returning Product or None
- get_active_products: paginated query with search
- Use SQLAlchemy select() statements
- Include proper type hints
- Add error handling
```

### Complex (Full Feature)
```
Create complete product management feature:

1. Product model (SQLAlchemy):
   - name, sku, price, stock, category_id, description, is_active
   - Relationships to Category, OrderItem
   - Timestamps and indexes

2. Pydantic schemas:
   - ProductBase, ProductCreate, ProductUpdate, ProductResponse
   - ProductListResponse with pagination
   - Validation for positive price, unique SKU

3. ProductRepository:
   - CRUD operations
   - get_by_sku, search_products, get_by_category
   - Pagination support

4. ProductService:
   - Business logic and validation
   - Check category exists
   - Verify SKU uniqueness
   - Prevent deletion if in orders

5. API endpoints (/api/v1/products):
   - Full CRUD with proper auth
   - Search and filtering
   - Admin-only for modifications

6. Tests:
   - Unit tests for service methods
   - Integration tests for API endpoints
   - Test factories for product creation
```

---

## Key Phrases for Best Results

Use these phrases to ensure compliance with coding standards:

**Architecture:**
- "Following layered architecture (API → Service → Repository → Model)"
- "Use dependency injection with Depends()"
- "Separate business logic in service layer"

**Database:**
- "Use async SQLAlchemy with AsyncSession"
- "Use select() statements (not query())"
- "Include pagination with offset/limit"
- "Eager load relationships with selectinload()"
- "Add proper database indexes"

**Validation:**
- "Use Pydantic schemas for all request/response"
- "Add field validators for [specific rules]"
- "Validate email format with EmailStr"
- "Ensure password strength validation"

**Error Handling:**
- "Raise custom exceptions (NotFoundException, ConflictException, etc.)"
- "Include proper error logging with context"
- "Return appropriate HTTP status codes"

**Security:**
- "Require authentication with Depends(get_current_user)"
- "Check user permissions for admin operations"
- "Hash passwords with bcrypt"
- "Validate JWT tokens"

**Performance:**
- "Limit maximum page size to 100"
- "Use database-level pagination"
- "Add caching where appropriate"

**Testing:**
- "Write pytest async tests"
- "Use test factories for data"
- "Mock external dependencies"
- "Aim for >80% coverage"

---

## Prompt Quality Checklist

- [ ] Clear action (what to create)
- [ ] Context provided (purpose)
- [ ] Requirements listed
- [ ] Data types specified
- [ ] Validation rules stated
- [ ] Dependencies identified
- [ ] Error handling addressed
- [ ] Authentication requirements noted
- [ ] Performance considerations mentioned

---

## Example Prompts

### Good Prompt (High Success) âœ…

```
Create a UserService for user management following our layered architecture.

Dependencies:
- UserRepository (injected)
- AsyncSession database connection

Methods:
1. get_user_by_id(user_id: int) -> User
   - Fetch user from repository
   - Raise NotFoundException if not found
   - Log the operation

2. create_user(user_data: UserCreate) -> User
   - Validate email uniqueness via repository
   - Raise ConflictException if email exists
   - Hash password using get_password_hash()
   - Create user via repository
   - Log successful creation

3. update_user(user_id: int, user_data: UserUpdate) -> User
   - Verify user exists
   - Check email uniqueness if email being changed
   - Update only provided fields
   - Raise appropriate exceptions

4. delete_user(user_id: int) -> None
   - Verify user exists
   - Check business rules (e.g., can't delete user with active orders)
   - Delete via repository

Error Handling:
- Use custom exceptions (NotFoundException, ConflictException, BadRequestException)
- Log all errors with user_id context
- Rollback transactions on errors

Type Hints:
- Add type hints to all parameters and return types
- Use type unions (User | None) where appropriate

All methods should be async with proper await statements.
```

### Bad Prompt (Low Success) âŒ

```
Create a user service
```

---

## Domain-Specific Patterns

### For E-commerce

```
Include:
- Inventory management (stock tracking)
- Order validation (stock availability)
- Price calculations (tax, discounts)
- Payment processing integration points
```

### For SaaS Applications

```
Include:
- Multi-tenancy support (tenant_id filtering)
- Subscription management
- Feature flags per tier
- Usage tracking/quotas
```

### For Content Management

```
Include:
- Versioning support
- Draft/publish workflows
- Media file handling
- SEO metadata fields
```

---

## Advanced Prompt Techniques

### Referencing Existing Code

```
Create a ProductService similar to UserService but with these differences:
- Add inventory management methods
- Include category validation
- Support SKU-based lookups

Follow the same error handling patterns and logging approach.
```

### Requesting Specific Patterns

```
Create a ProductRepository using the generic BaseRepository pattern.

Override these methods:
- get_by_sku: Custom query for SKU lookup
- search: Full-text search on name and description

Keep all other CRUD operations from base class.
Use async/await throughout.
```

### Including Test Requirements

```
Create a UserService with complete pytest test coverage:

Service Methods:
[... method descriptions ...]

Tests Required:
- Happy path tests for each method
- Error case tests (not found, conflict, validation)
- Mock the repository layer
- Use AAA pattern (Arrange, Act, Assert)
- Aim for >90% coverage
```

---

## Common Mistakes to Avoid

### âŒ Vague Requirements
```
"Create an API for products"
```

### âœ… Specific Requirements
```
"Create RESTful API endpoints for product CRUD at /api/v1/products with:
- Pagination on list endpoint
- Authentication required
- Admin role for modifications
- Pydantic schema validation
- Proper HTTP status codes"
```

---

### âŒ Missing Context
```
"Add validation"
```

### âœ… Clear Context
```
"Add Pydantic field validators to ProductCreate schema:
- price must be positive decimal
- stock must be non-negative integer
- sku must be alphanumeric, 8-20 characters"
```

---

### âŒ No Error Handling
```
"Create a method to get user"
```

### âœ… With Error Handling
```
"Create async method get_user_by_id(user_id: int) -> User that:
- Queries database using repository
- Raises NotFoundException if user not found
- Logs the operation with user_id
- Returns User model"
```

---

## Effectiveness Matrix

| Prompt Detail Level | Success Rate | Time to Correct Result |
|-------------------|--------------|----------------------|
| Minimal ("create user endpoint") | 30% | High |
| Basic ("create user CRUD endpoint") | 60% | Medium |
| Detailed ("create user endpoint with auth, validation, pagination") | 90% | Low |
| Comprehensive (full CARD framework) | 95%+ | Very Low |

**Goal: Aim for Detailed or Comprehensive prompts**

---

## Quick Reference Card

### When Creating Models
- Specify all fields with types
- Mention relationships
- Request indexes
- Include timestamps
- State constraints

### When Creating Schemas
- List all fields
- Specify validation rules
- Separate Create/Update/Response
- Include config options

### When Creating Repositories
- List custom query methods
- Specify return types
- Request pagination support
- Mention eager loading

### When Creating Services
- Define all methods
- List business rules
- Specify error handling
- Mention logging requirements

### When Creating Endpoints
- List all HTTP methods
- Specify authentication
- Define request/response schemas
- Include documentation

---

## Tips for Success

1. **Be Specific**: More details = better results
2. **Include Types**: Always specify data types
3. **State Patterns**: Reference architectural patterns
4. **Request Tests**: Ask for tests alongside code
5. **Mention Standards**: Reference coding standards explicitly
6. **Use Examples**: "Similar to X but with Y changes"
7. **Iterate**: Refine based on initial results

---

**Remember**: 
- Clear prompt = Clean code
- Context matters
- Specificity wins
- Standards compliance starts with the prompt
