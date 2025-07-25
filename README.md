# FastAPI RestFull

FastAPI RestFull provides asynchronous utilities and base classes to accelerate building APIs with **FastAPI** and the **Beanie** ODM on MongoDB. It helps create repositories, services and controllers that share a common architecture while following SOLID principles and strong typing with Pydantic.

## Key Features

- **Generic classes** for repositories (`BaseRepository`), services (`BaseService`) and controllers (`BaseController`) ready to extend.
- **Unified response schemas** (`BaseResponse` and `BasePaginationResponse`).
- **Exception handlers** and custom exceptions for common errors.
- **Third-party services** like JWT token management and file uploads to Supabase.
- Built to work **fully asynchronously**, making horizontal scalability easier.

## Installation

```bash
pip install fastapi-restfull
```

## Quickstart

```python
from contextlib import asynccontextmanager
from fastapi import Depends, FastAPI
from beanie import Document, init_beanie
from pymongo import AsyncMongoClient

from fastapi_restfull.aio.repository.base import BaseRepository
from fastapi_restfull.aio.service.base import BaseService
from fastapi_restfull.aio.controller.base import BaseController
from fastapi_restfull.exceptions.handler import (
    api_exception_handler,
    validation_exception_handler,
)

app = FastAPI()

# Beanie model
class Item(Document):
    name: str

# Repository
class ItemRepository(BaseRepository):
    model = Item

# Service
class ItemService(BaseService):
    repository: ItemRepository
    search_fields = ["name"]
    duplicate_check_fields = ["name"]

# Controller
class ItemController(BaseController):
    service: ItemService = Depends()
    schema_class = Item


@asynccontextmanager
async def lifespan(app: FastAPI):
    client = AsyncMongoClient("mongodb://localhost:27017")
    await init_beanie(database=client.db, document_models=[Item])
    yield

app.add_exception_handler(Exception, api_exception_handler)
app.add_exception_handler(ValueError, validation_exception_handler)

controller = ItemController()
app.add_api_route("/items", controller.list, methods=["GET"])
app.add_api_route("/items/{id}", controller.retrieve, methods=["GET"])
app.add_api_route("/items", controller.create, methods=["POST"])
```

This snippet shows how to inherit the base classes and expose a basic CRUD in FastAPI.

## Advanced Example

Below is a more complete example that configures middlewares, custom error handlers and a user CRUD separated into repository, service and controller.

```python
from pymongo.errors import DuplicateKeyError

from fastapi import FastAPI, status
from fastapi.exceptions import RequestValidationError
from fastapi.middleware.cors import CORSMiddleware

from slowapi import _rate_limit_exceeded_handler


from app.api.v1.routers import api_router

from app.base.fastapi_restfull.exceptions import APIException
from app.config.database import lifespan
from app.config.limit import limiter
from app.config.settings import get_settings

from app.base.fastapi_restfull.exceptions import (
    api_exception_handler,
    duplicate_key_exception_handler,
    global_exception_handler,
    validation_exception_handler,
    value_exception_handler,
)

settings = get_settings()


app = FastAPI(
    title=settings.PROJECT_NAME,
    version=settings.VERSION,
    lifespan=lifespan,
    redirect_slashes=True,
)


app.add_middleware(
    CORSMiddleware,
    allow_origins=["*"],
    allow_credentials=True,
    allow_methods=[
        "*"
    ],  # allow all methods
    allow_headers=["*"],  # allow all headers
)

app.state.limiter = limiter

app.include_router(api_router, prefix=settings.API_V1_STR)

# Register custom handlers
app.add_exception_handler(
    status.HTTP_429_TOO_MANY_REQUESTS, _rate_limit_exceeded_handler
)
app.add_exception_handler(APIException, api_exception_handler)
app.add_exception_handler(RequestValidationError, validation_exception_handler)
app.add_exception_handler(ValueError, value_exception_handler)
app.add_exception_handler(Exception, global_exception_handler)
app.add_exception_handler(DuplicateKeyError, duplicate_key_exception_handler)


# Import and assign the custom_openapi function
from app.config.openapi import custom_openapi  # noqa

app.openapi = custom_openapi
```

```python
from beanie import PydanticObjectId
from fastapi import APIRouter, Body, Depends, Query, status
from fastapi_restful.cbv import cbv

from app.base.fastapi_restfull.aio.controller import BaseController
from app.base.fastapi_restfull.schema.base import BaseResponse
from app.models.user import User
from app.schemas.user.user import (
    UserDResponseSchema,
    UserPResponseSchema,
    UserResponseSchema,
    UserURequestSchema,
)
from app.services.dependency.current_user import get_dependency_service
from app.services.system.user.user_service import UserService, get_user_service

router = APIRouter()


@cbv(router)
class UserController(BaseController):
    service: UserService = Depends(get_user_service)
    user: User = Depends(get_dependency_service)

    def get_schema_class(self):
        if self.action == "retrieve":
            return UserDResponseSchema
        else:
            return UserResponseSchema

    @router.get("/", response_model=UserPResponseSchema)
    async def list(
        self,
        page: int = Query(1, ge=1, description="Page number"),
        limit: int = Query(10, ge=1, description="Items per page"),
        search: str | None = Query(None, description="Search term"),
    ):
        """List users with pagination and optional search."""
        return await super().list(page=page, count=limit, search=search)

    @router.get("/{id}", response_model=BaseResponse[UserDResponseSchema])
    async def retrieve(self, id: PydanticObjectId):
        return await super().retrieve(id)

    @router.patch("/{id}", response_model=BaseResponse[UserResponseSchema])
    async def update(
        self,
        id: PydanticObjectId,
        data: UserURequestSchema = Body(...),
    ):
        return await super().update(id, data)

    @router.delete("/{id}", status_code=status.HTTP_204_NO_CONTENT)
    async def delete(self, id: PydanticObjectId):
        return await super().delete(id)
```

```python
from fastapi import Request
from app.base.fastapi_restfull.aio.service import BaseService
from app.repositories.user.user import UserRepository


class UserService(BaseService):
    repository: UserRepository

    def __init__(
        self, repository: UserRepository = None, request: Request = None
    ):
        super().__init__(repository, request)
        self.search_fields = ["name", "email"]

    def get_kwargs_query(self):
        """
        Return the query arguments for the repository.
        Allows customizing queries in child services.
        """
        if self.action == "retrieve":
            return {
                "fetch_links": True,
                "nesting_depths_per_field": {"role": 1, "company": 1},
            }
        return super().get_kwargs_query()


def get_user_service(request: Request) -> UserService:
    repo = UserRepository()
    service = UserService(request=request, repository=repo)
    return service
```

```python
from app.base.fastapi_restfull.aio.repository import BaseRepository
from app.models.user import User


class UserRepository(BaseRepository):
    model: User = User
```

## Additional Services

The package includes utilities for common integrations:

- **JWTService**: create, validate and refresh JWT tokens.
- **SupabaseService**: upload and delete files in Supabase Storage.

You can import them from `fastapi_restfull.servicios` and use them like any other FastAPI dependency.

## Tests

The project contains tests that validate the main functionality. Install the dependencies defined in `pyproject.toml` and run:

```bash
pytest
```

## License

MIT
