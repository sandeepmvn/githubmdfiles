# Python FastAPI Project - GitHub Copilot Instructions

> **Purpose**: These are workspace-wide coding standards for Python 3.11+ FastAPI projects using layered architecture. All AI-assisted code generation must follow these guidelines.

---

## 🎯 Core Principles

### Architecture
- **Layered Architecture**: API → Services → Repositories → Models
- **Dependency Flow**: Always flows inward (API depends on Services, Services depend on Repositories)
- **Separation of Concerns**: Routes (HTTP), Services (Business Logic), Repositories (Data Access)

### Naming Conventions
- **PascalCase**: Classes, Pydantic Models, Enums
- **snake_case**: Functions, methods, variables, file names, modules
- **SCREAMING_SNAKE_CASE**: Constants, environment variables
- **Private methods**: Prefix with single underscore `_method_name`
- **Async methods**: No special suffix needed (Python convention)

### Code Quality Standards
- **All I/O operations must be async** with `async def`
- **Type hints required** for all function parameters and return values
- **Pydantic models for validation** (never accept raw dicts for API input)
- **Repository pattern** for all database access
- **Dependency injection** using FastAPI's Depends()
- **Pagination required** for all list endpoints
- **Proper exception handling** with custom exceptions

---

## 📁 Project Structure

```
your-project/
│
├── app/
│   ├── __init__.py
│   ├── main.py                    # FastAPI application entry point
│   ├── config.py                  # Configuration settings
│   ├── dependencies.py            # Shared dependencies
│   │
│   ├── api/                       # API layer
│   │   ├── __init__.py
│   │   ├── v1/
│   │   │   ├── __init__.py
│   │   │   ├── endpoints/
│   │   │   │   ├── __init__.py
│   │   │   │   ├── users.py
│   │   │   │   ├── auth.py
│   │   │   │   └── products.py
│   │   │   └── router.py          # API v1 router
│   │   └── deps.py                # API-specific dependencies
│   │
│   ├── core/                      # Core functionality
│   │   ├── __init__.py
│   │   ├── config.py              # Settings management
│   │   ├── security.py            # Security utilities (JWT, password hashing)
│   │   └── exceptions.py          # Custom exceptions
│   │
│   ├── services/                  # Business logic layer
│   │   ├── __init__.py
│   │   ├── user_service.py
│   │   ├── auth_service.py
│   │   └── product_service.py
│   │
│   ├── repositories/              # Data access layer
│   │   ├── __init__.py
│   │   ├── base.py                # Base repository
│   │   ├── user_repository.py
│   │   └── product_repository.py
│   │
│   ├── models/                    # Database models (SQLAlchemy)
│   │   ├── __init__.py
│   │   ├── base.py                # Base model
│   │   ├── user.py
│   │   └── product.py
│   │
│   ├── schemas/                   # Pydantic schemas
│   │   ├── __init__.py
│   │   ├── user.py
│   │   ├── auth.py
│   │   ├── product.py
│   │   └── common.py              # Shared schemas (pagination, etc.)
│   │
│   ├── db/                        # Database configuration
│   │   ├── __init__.py
│   │   ├── session.py             # Database session
│   │   └── base.py                # Import all models for Alembic
│   │
│   └── utils/                     # Utility functions
│       ├── __init__.py
│       ├── logger.py
│       └── helpers.py
│
├── alembic/                       # Database migrations
│   ├── versions/
│   └── env.py
│
├── tests/
│   ├── __init__.py
│   ├── conftest.py                # Pytest fixtures
│   ├── unit/
│   │   ├── test_services/
│   │   └── test_repositories/
│   └── integration/
│       └── test_api/
│
├── .env.example                   # Environment variables template
├── .gitignore
├── alembic.ini                    # Alembic configuration
├── pyproject.toml                 # Project dependencies (Poetry/pip)
├── requirements.txt               # Or use this for pip
└── README.md
```

---

## 🔥 CRITICAL PERFORMANCE RULES

### SQLAlchemy Rule #1: ALWAYS Use Async Sessions

```python
# ❌ NEVER - Synchronous session
from sqlalchemy.orm import Session

def get_user(db: Session, user_id: int):
    return db.query(User).filter(User.id == user_id).first()

# ✅ ALWAYS - Async session
from sqlalchemy.ext.asyncio import AsyncSession
from sqlalchemy import select

async def get_user(db: AsyncSession, user_id: int) -> User | None:
    result = await db.execute(select(User).where(User.id == user_id))
    return result.scalar_one_or_none()
```

### SQLAlchemy Rule #2: ALWAYS Paginate

```python
# ❌ NEVER - Load entire table
async def get_all_users(db: AsyncSession) -> list[User]:
    result = await db.execute(select(User))
    return result.scalars().all()

# ✅ ALWAYS - Paginated query
async def get_users_paginated(
    db: AsyncSession,
    skip: int = 0,
    limit: int = 10
) -> list[User]:
    result = await db.execute(
        select(User)
        .offset(skip)
        .limit(limit)
        .order_by(User.created_at.desc())
    )
    return result.scalars().all()
```

### SQLAlchemy Rule #3: Use Select Statements

```python
# ❌ AVOID - Legacy query API (deprecated in SQLAlchemy 2.0)
users = db.query(User).filter(User.is_active == True).all()

# ✅ ALWAYS - Modern select statement
from sqlalchemy import select

result = await db.execute(
    select(User).where(User.is_active == True)
)
users = result.scalars().all()
```

### Query Best Practices

```python
from sqlalchemy import select, func
from sqlalchemy.orm import selectinload

async def get_users_with_posts_paginated(
    db: AsyncSession,
    skip: int = 0,
    limit: int = 10,
    search: str | None = None
) -> tuple[list[User], int]:
    # Base query
    query = select(User)
    
    # Add search filter
    if search:
        query = query.where(User.name.ilike(f"%{search}%"))
    
    # Get total count
    count_query = select(func.count()).select_from(query.subquery())
    total = await db.scalar(count_query)
    
    # Apply pagination and eager loading
    query = (
        query
        .options(selectinload(User.posts))  # Avoid N+1 queries
        .offset(skip)
        .limit(limit)
        .order_by(User.created_at.desc())
    )
    
    result = await db.execute(query)
    users = result.scalars().all()
    
    return users, total
```

---

## 📋 Standard Patterns

### Pydantic Schema Pattern

```python
from pydantic import BaseModel, EmailStr, Field, ConfigDict
from datetime import datetime

# Base schema (shared fields)
class UserBase(BaseModel):
    email: EmailStr
    full_name: str = Field(..., min_length=1, max_length=100)
    is_active: bool = True

# Create schema (for POST requests)
class UserCreate(UserBase):
    password: str = Field(..., min_length=8, max_length=100)

# Update schema (for PUT/PATCH requests)
class UserUpdate(BaseModel):
    email: EmailStr | None = None
    full_name: str | None = Field(None, min_length=1, max_length=100)
    is_active: bool | None = None

# Response schema (for API responses)
class UserResponse(UserBase):
    id: int
    created_at: datetime
    updated_at: datetime
    
    model_config = ConfigDict(from_attributes=True)

# List response with pagination
class UserListResponse(BaseModel):
    items: list[UserResponse]
    total: int
    page: int
    page_size: int
    total_pages: int
```

### Repository Pattern

```python
from typing import Generic, TypeVar, Type, Any
from sqlalchemy.ext.asyncio import AsyncSession
from sqlalchemy import select, update, delete, func
from app.models.base import Base

ModelType = TypeVar("ModelType", bound=Base)

class BaseRepository(Generic[ModelType]):
    def __init__(self, model: Type[ModelType], db: AsyncSession):
        self.model = model
        self.db = db
    
    async def get_by_id(self, id: Any) -> ModelType | None:
        """Get a single record by ID"""
        result = await self.db.execute(
            select(self.model).where(self.model.id == id)
        )
        return result.scalar_one_or_none()
    
    async def get_multi(
        self,
        skip: int = 0,
        limit: int = 100,
        order_by: Any | None = None
    ) -> tuple[list[ModelType], int]:
        """Get multiple records with pagination"""
        # Count total
        count_query = select(func.count()).select_from(self.model)
        total = await self.db.scalar(count_query) or 0
        
        # Get items
        query = select(self.model).offset(skip).limit(limit)
        
        if order_by is not None:
            query = query.order_by(order_by)
        
        result = await self.db.execute(query)
        items = result.scalars().all()
        
        return items, total
    
    async def create(self, obj_in: dict[str, Any]) -> ModelType:
        """Create a new record"""
        db_obj = self.model(**obj_in)
        self.db.add(db_obj)
        await self.db.commit()
        await self.db.refresh(db_obj)
        return db_obj
    
    async def update(
        self,
        id: Any,
        obj_in: dict[str, Any]
    ) -> ModelType | None:
        """Update an existing record"""
        db_obj = await self.get_by_id(id)
        if not db_obj:
            return None
        
        for field, value in obj_in.items():
            setattr(db_obj, field, value)
        
        await self.db.commit()
        await self.db.refresh(db_obj)
        return db_obj
    
    async def delete(self, id: Any) -> bool:
        """Delete a record"""
        result = await self.db.execute(
            delete(self.model).where(self.model.id == id)
        )
        await self.db.commit()
        return result.rowcount > 0

# Specific repository
class UserRepository(BaseRepository[User]):
    async def get_by_email(self, email: str) -> User | None:
        """Get user by email"""
        result = await self.db.execute(
            select(User).where(User.email == email)
        )
        return result.scalar_one_or_none()
    
    async def get_active_users(
        self,
        skip: int = 0,
        limit: int = 100
    ) -> tuple[list[User], int]:
        """Get active users with pagination"""
        # Count
        count_query = select(func.count()).select_from(User).where(User.is_active == True)
        total = await self.db.scalar(count_query) or 0
        
        # Query
        query = (
            select(User)
            .where(User.is_active == True)
            .offset(skip)
            .limit(limit)
            .order_by(User.created_at.desc())
        )
        
        result = await self.db.execute(query)
        users = result.scalars().all()
        
        return users, total
```

### Service Pattern

```python
from app.repositories.user_repository import UserRepository
from app.schemas.user import UserCreate, UserUpdate
from app.models.user import User
from app.core.security import get_password_hash, verify_password
from app.core.exceptions import NotFoundException, ConflictException
from sqlalchemy.ext.asyncio import AsyncSession

class UserService:
    def __init__(self, db: AsyncSession):
        self.repository = UserRepository(User, db)
    
    async def get_user_by_id(self, user_id: int) -> User:
        """Get user by ID"""
        user = await self.repository.get_by_id(user_id)
        if not user:
            raise NotFoundException(f"User with id {user_id} not found")
        return user
    
    async def get_users(
        self,
        skip: int = 0,
        limit: int = 10
    ) -> tuple[list[User], int]:
        """Get paginated list of users"""
        return await self.repository.get_active_users(skip, limit)
    
    async def create_user(self, user_data: UserCreate) -> User:
        """Create a new user"""
        # Check if user exists
        existing_user = await self.repository.get_by_email(user_data.email)
        if existing_user:
            raise ConflictException(f"User with email {user_data.email} already exists")
        
        # Hash password
        hashed_password = get_password_hash(user_data.password)
        
        # Create user
        user_dict = user_data.model_dump(exclude={"password"})
        user_dict["hashed_password"] = hashed_password
        
        return await self.repository.create(user_dict)
    
    async def update_user(
        self,
        user_id: int,
        user_data: UserUpdate
    ) -> User:
        """Update user"""
        # Verify user exists
        user = await self.get_user_by_id(user_id)
        
        # Check email uniqueness if being changed
        if user_data.email and user_data.email != user.email:
            existing = await self.repository.get_by_email(user_data.email)
            if existing:
                raise ConflictException(f"Email {user_data.email} already in use")
        
        # Update only provided fields
        update_data = user_data.model_dump(exclude_unset=True)
        updated_user = await self.repository.update(user_id, update_data)
        
        if not updated_user:
            raise NotFoundException(f"User with id {user_id} not found")
        
        return updated_user
    
    async def delete_user(self, user_id: int) -> None:
        """Delete user"""
        deleted = await self.repository.delete(user_id)
        if not deleted:
            raise NotFoundException(f"User with id {user_id} not found")
```

### API Endpoint Pattern

```python
from fastapi import APIRouter, Depends, Query, status
from sqlalchemy.ext.asyncio import AsyncSession
from app.db.session import get_db
from app.services.user_service import UserService
from app.schemas.user import UserCreate, UserUpdate, UserResponse, UserListResponse
from app.schemas.common import PaginationParams
from app.api.deps import get_current_user
from app.models.user import User

router = APIRouter(prefix="/users", tags=["users"])

@router.get(
    "",
    response_model=UserListResponse,
    summary="Get all users"
)
async def get_users(
    pagination: PaginationParams = Depends(),
    db: AsyncSession = Depends(get_db),
    current_user: User = Depends(get_current_user)
):
    """
    Retrieve users with pagination.
    
    - **page**: Page number (default: 1)
    - **page_size**: Items per page (default: 10, max: 100)
    """
    service = UserService(db)
    
    skip = (pagination.page - 1) * pagination.page_size
    users, total = await service.get_users(skip=skip, limit=pagination.page_size)
    
    total_pages = (total + pagination.page_size - 1) // pagination.page_size
    
    return UserListResponse(
        items=[UserResponse.model_validate(user) for user in users],
        total=total,
        page=pagination.page,
        page_size=pagination.page_size,
        total_pages=total_pages
    )

@router.get(
    "/{user_id}",
    response_model=UserResponse,
    summary="Get user by ID"
)
async def get_user(
    user_id: int,
    db: AsyncSession = Depends(get_db),
    current_user: User = Depends(get_current_user)
):
    """Get a specific user by ID"""
    service = UserService(db)
    user = await service.get_user_by_id(user_id)
    return UserResponse.model_validate(user)

@router.post(
    "",
    response_model=UserResponse,
    status_code=status.HTTP_201_CREATED,
    summary="Create new user"
)
async def create_user(
    user_data: UserCreate,
    db: AsyncSession = Depends(get_db)
):
    """
    Create a new user.
    
    Required fields:
    - **email**: Valid email address
    - **password**: Minimum 8 characters
    - **full_name**: User's full name
    """
    service = UserService(db)
    user = await service.create_user(user_data)
    return UserResponse.model_validate(user)

@router.put(
    "/{user_id}",
    response_model=UserResponse,
    summary="Update user"
)
async def update_user(
    user_id: int,
    user_data: UserUpdate,
    db: AsyncSession = Depends(get_db),
    current_user: User = Depends(get_current_user)
):
    """Update user information"""
    service = UserService(db)
    user = await service.update_user(user_id, user_data)
    return UserResponse.model_validate(user)

@router.delete(
    "/{user_id}",
    status_code=status.HTTP_204_NO_CONTENT,
    summary="Delete user"
)
async def delete_user(
    user_id: int,
    db: AsyncSession = Depends(get_db),
    current_user: User = Depends(get_current_user)
):
    """Delete a user"""
    service = UserService(db)
    await service.delete_user(user_id)
```

---

## 🚫 Never Do These

1. ❌ **Never** use synchronous database operations (always async)
2. ❌ **Never** return unbounded lists (always paginate)
3. ❌ **Never** accept raw dicts for request bodies (use Pydantic models)
4. ❌ **Never** put business logic in route handlers
5. ❌ **Never** expose database models directly (use schemas)
6. ❌ **Never** hardcode secrets or API keys
7. ❌ **Never** ignore type hints
8. ❌ **Never** use `except Exception` without logging
9. ❌ **Never** use mutable default arguments
10. ❌ **Never** return `None` for lists (return empty list `[]`)

---

## ✅ Always Do These

1. ✅ **Always** use async/await for I/O operations
2. ✅ **Always** add type hints to all functions
3. ✅ **Always** use Pydantic for validation
4. ✅ **Always** implement pagination for list endpoints
5. ✅ **Always** use dependency injection (FastAPI Depends)
6. ✅ **Always** handle exceptions with custom exception classes
7. ✅ **Always** log errors with structured logging
8. ✅ **Always** use environment variables for configuration
9. ✅ **Always** validate input data
10. ✅ **Always** enforce maximum page size (50-100)

---

## 🔒 Security & Authentication

### JWT Configuration

```python
# app/core/security.py
from datetime import datetime, timedelta
from typing import Any
from jose import jwt
from passlib.context import CryptContext
from app.core.config import settings

pwd_context = CryptContext(schemes=["bcrypt"], deprecated="auto")

def create_access_token(data: dict[str, Any], expires_delta: timedelta | None = None) -> str:
    """Create JWT access token"""
    to_encode = data.copy()
    
    if expires_delta:
        expire = datetime.utcnow() + expires_delta
    else:
        expire = datetime.utcnow() + timedelta(minutes=settings.ACCESS_TOKEN_EXPIRE_MINUTES)
    
    to_encode.update({"exp": expire})
    encoded_jwt = jwt.encode(
        to_encode,
        settings.SECRET_KEY,
        algorithm=settings.ALGORITHM
    )
    return encoded_jwt

def verify_password(plain_password: str, hashed_password: str) -> bool:
    """Verify password against hash"""
    return pwd_context.verify(plain_password, hashed_password)

def get_password_hash(password: str) -> str:
    """Hash password"""
    return pwd_context.hash(password)
```

### Authentication Dependency

```python
# app/api/deps.py
from fastapi import Depends, HTTPException, status
from fastapi.security import OAuth2PasswordBearer
from jose import JWTError, jwt
from sqlalchemy.ext.asyncio import AsyncSession
from app.core.config import settings
from app.db.session import get_db
from app.models.user import User
from app.repositories.user_repository import UserRepository

oauth2_scheme = OAuth2PasswordBearer(tokenUrl="/api/v1/auth/login")

async def get_current_user(
    token: str = Depends(oauth2_scheme),
    db: AsyncSession = Depends(get_db)
) -> User:
    """Get current authenticated user"""
    credentials_exception = HTTPException(
        status_code=status.HTTP_401_UNAUTHORIZED,
        detail="Could not validate credentials",
        headers={"WWW-Authenticate": "Bearer"},
    )
    
    try:
        payload = jwt.decode(
            token,
            settings.SECRET_KEY,
            algorithms=[settings.ALGORITHM]
        )
        user_id: int = payload.get("sub")
        if user_id is None:
            raise credentials_exception
    except JWTError:
        raise credentials_exception
    
    repository = UserRepository(User, db)
    user = await repository.get_by_id(user_id)
    
    if user is None:
        raise credentials_exception
    
    if not user.is_active:
        raise HTTPException(
            status_code=status.HTTP_403_FORBIDDEN,
            detail="Inactive user"
        )
    
    return user

async def get_current_active_superuser(
    current_user: User = Depends(get_current_user)
) -> User:
    """Get current active superuser"""
    if not current_user.is_superuser:
        raise HTTPException(
            status_code=status.HTTP_403_FORBIDDEN,
            detail="Not enough permissions"
        )
    return current_user
```

---

## 📊 Error Handling

### Custom Exceptions

```python
# app/core/exceptions.py
from fastapi import HTTPException, status

class NotFoundException(HTTPException):
    """Resource not found exception"""
    def __init__(self, detail: str = "Resource not found"):
        super().__init__(
            status_code=status.HTTP_404_NOT_FOUND,
            detail=detail
        )

class ConflictException(HTTPException):
    """Resource conflict exception"""
    def __init__(self, detail: str = "Resource already exists"):
        super().__init__(
            status_code=status.HTTP_409_CONFLICT,
            detail=detail
        )

class BadRequestException(HTTPException):
    """Bad request exception"""
    def __init__(self, detail: str = "Bad request"):
        super().__init__(
            status_code=status.HTTP_400_BAD_REQUEST,
            detail=detail
        )

class UnauthorizedException(HTTPException):
    """Unauthorized exception"""
    def __init__(self, detail: str = "Unauthorized"):
        super().__init__(
            status_code=status.HTTP_401_UNAUTHORIZED,
            detail=detail,
            headers={"WWW-Authenticate": "Bearer"}
        )

class ForbiddenException(HTTPException):
    """Forbidden exception"""
    def __init__(self, detail: str = "Forbidden"):
        super().__init__(
            status_code=status.HTTP_403_FORBIDDEN,
            detail=detail
        )
```

### Global Exception Handler

```python
# app/main.py
from fastapi import FastAPI, Request, status
from fastapi.responses import JSONResponse
from fastapi.exceptions import RequestValidationError
from sqlalchemy.exc import IntegrityError
import logging

logger = logging.getLogger(__name__)

app = FastAPI()

@app.exception_handler(RequestValidationError)
async def validation_exception_handler(request: Request, exc: RequestValidationError):
    """Handle validation errors"""
    logger.error(f"Validation error: {exc.errors()}")
    return JSONResponse(
        status_code=status.HTTP_422_UNPROCESSABLE_ENTITY,
        content={
            "detail": exc.errors(),
            "body": exc.body
        }
    )

@app.exception_handler(IntegrityError)
async def integrity_exception_handler(request: Request, exc: IntegrityError):
    """Handle database integrity errors"""
    logger.error(f"Database integrity error: {str(exc)}")
    return JSONResponse(
        status_code=status.HTTP_409_CONFLICT,
        content={"detail": "Database integrity error"}
    )

@app.exception_handler(Exception)
async def general_exception_handler(request: Request, exc: Exception):
    """Handle all other exceptions"""
    logger.error(f"Unhandled exception: {str(exc)}", exc_info=True)
    return JSONResponse(
        status_code=status.HTTP_500_INTERNAL_SERVER_ERROR,
        content={"detail": "Internal server error"}
    )
```

---

## 🧪 Testing Standards

### Pytest Configuration

```python
# tests/conftest.py
import pytest
import asyncio
from typing import AsyncGenerator, Generator
from httpx import AsyncClient
from sqlalchemy.ext.asyncio import AsyncSession, create_async_engine
from sqlalchemy.orm import sessionmaker
from app.main import app
from app.db.session import get_db
from app.models.base import Base

# Test database URL
TEST_DATABASE_URL = "sqlite+aiosqlite:///./test.db"

# Create async engine for testing
test_engine = create_async_engine(TEST_DATABASE_URL, echo=False)
TestSessionLocal = sessionmaker(
    test_engine,
    class_=AsyncSession,
    expire_on_commit=False
)

@pytest.fixture(scope="session")
def event_loop() -> Generator:
    """Create event loop for async tests"""
    loop = asyncio.get_event_loop_policy().new_event_loop()
    yield loop
    loop.close()

@pytest.fixture(scope="function")
async def db_session() -> AsyncGenerator[AsyncSession, None]:
    """Create test database session"""
    async with test_engine.begin() as conn:
        await conn.run_sync(Base.metadata.create_all)
    
    async with TestSessionLocal() as session:
        yield session
    
    async with test_engine.begin() as conn:
        await conn.run_sync(Base.metadata.drop_all)

@pytest.fixture(scope="function")
async def client(db_session: AsyncSession) -> AsyncGenerator[AsyncClient, None]:
    """Create test client"""
    async def override_get_db():
        yield db_session
    
    app.dependency_overrides[get_db] = override_get_db
    
    async with AsyncClient(app=app, base_url="http://test") as ac:
        yield ac
    
    app.dependency_overrides.clear()
```

### Test Example

```python
# tests/integration/test_api/test_users.py
import pytest
from httpx import AsyncClient
from sqlalchemy.ext.asyncio import AsyncSession
from app.models.user import User
from app.core.security import get_password_hash

@pytest.mark.asyncio
async def test_create_user(client: AsyncClient):
    """Test creating a new user"""
    user_data = {
        "email": "test@example.com",
        "password": "testpassword123",
        "full_name": "Test User"
    }
    
    response = await client.post("/api/v1/users/", json=user_data)
    
    assert response.status_code == 201
    data = response.json()
    assert data["email"] == user_data["email"]
    assert data["full_name"] == user_data["full_name"]
    assert "id" in data
    assert "hashed_password" not in data

@pytest.mark.asyncio
async def test_get_user(client: AsyncClient, db_session: AsyncSession):
    """Test getting a user by ID"""
    # Create test user
    user = User(
        email="test@example.com",
        hashed_password=get_password_hash("password"),
        full_name="Test User"
    )
    db_session.add(user)
    await db_session.commit()
    await db_session.refresh(user)
    
    response = await client.get(f"/api/v1/users/{user.id}")
    
    assert response.status_code == 200
    data = response.json()
    assert data["email"] == user.email
    assert data["id"] == user.id

@pytest.mark.asyncio
async def test_get_users_pagination(client: AsyncClient, db_session: AsyncSession):
    """Test getting users with pagination"""
    # Create multiple users
    for i in range(15):
        user = User(
            email=f"user{i}@example.com",
            hashed_password=get_password_hash("password"),
            full_name=f"User {i}"
        )
        db_session.add(user)
    await db_session.commit()
    
    response = await client.get("/api/v1/users/?page=1&page_size=10")
    
    assert response.status_code == 200
    data = response.json()
    assert len(data["items"]) == 10
    assert data["total"] == 15
    assert data["total_pages"] == 2
```

---

## 📦 Required Dependencies

```toml
# pyproject.toml (using Poetry)
[tool.poetry]
name = "your-project"
version = "0.1.0"
description = "FastAPI project"
authors = ["Your Name <you@example.com>"]

[tool.poetry.dependencies]
python = "^3.11"
fastapi = "^0.109.0"
uvicorn = {extras = ["standard"], version = "^0.27.0"}
sqlalchemy = {extras = ["asyncio"], version = "^2.0.25"}
alembic = "^1.13.0"
asyncpg = "^0.29.0"  # For PostgreSQL
pydantic = {extras = ["email"], version = "^2.5.0"}
pydantic-settings = "^2.1.0"
python-jose = {extras = ["cryptography"], version = "^3.3.0"}
passlib = {extras = ["bcrypt"], version = "^1.7.4"}
python-multipart = "^0.0.6"

[tool.poetry.dev-dependencies]
pytest = "^7.4.0"
pytest-asyncio = "^0.21.0"
httpx = "^0.26.0"
aiosqlite = "^0.19.0"  # For testing with SQLite
black = "^23.12.0"
ruff = "^0.1.9"
mypy = "^1.8.0"

# Or requirements.txt (using pip)
fastapi==0.109.0
uvicorn[standard]==0.27.0
sqlalchemy[asyncio]==2.0.25
alembic==1.13.0
asyncpg==0.29.0
pydantic[email]==2.5.0
pydantic-settings==2.1.0
python-jose[cryptography]==3.3.0
passlib[bcrypt]==1.7.4
python-multipart==0.0.6
```

---

## ✅ Code Review Checklist

Before committing code, verify:

- [ ] All functions have type hints
- [ ] All I/O operations are async
- [ ] Pydantic models used for all request/response data
- [ ] Pagination implemented for list endpoints
- [ ] Maximum page size enforced (≤100)
- [ ] Custom exceptions used for error handling
- [ ] No business logic in route handlers
- [ ] Database sessions properly managed (async context)
- [ ] No SQL injection vulnerabilities
- [ ] Passwords hashed before storage
- [ ] JWT tokens properly validated
- [ ] Proper logging in place
- [ ] Unit tests for business logic
- [ ] Integration tests for API endpoints
- [ ] No hardcoded secrets or credentials
- [ ] Environment variables used for configuration

---

## 🎯 Summary

**Golden Rules**:
1. Type Hints Always
2. Async All I/O
3. Pydantic for Validation
4. Paginate Everything
5. Dependency Injection
6. Custom Exceptions
7. Repository Pattern
8. Service Layer
9. Test Thoroughly
10. Secure by Default

**Remember**: FastAPI's async nature and type system are your biggest advantages - use them!

---

*For detailed examples and advanced patterns, refer to the specialized instruction files.*
