# SSO API Project Context

This document provides context about the project structure, conventions, and key decisions made during development.

## Project Overview

This is a Django REST Framework (DRF) project for Single Sign-On (SSO) functionality, following the HackSoftware Django Styleguide best practices.

## Project Structure

### Core Architecture Pattern

The project follows a **layered architecture** with clear separation of concerns:

```
sso/
├── users/              # Users app
│   ├── models/         # Database models
│   ├── manager/        # Custom model managers
│   ├── selector/       # Query functions (returns QuerySets)
│   ├── services/       # Business logic
│   ├── apis/           # API views and serializers
│   ├── urls/           # URL routing
│   ├── utils/          # App-specific utilities
│   └── constants.py    # App constants
├── utils/              # Shared utilities
│   ├── handlers/       # Exception handlers, integrity error handlers
│   ├── validators/     # Validation functions
│   ├── query_builder/  # Query building utilities
│   └── swagger/        # Swagger/OpenAPI utilities
├── api/                # API-level utilities
│   ├── mixins/         # API mixins
│   ├── pagination/     # Pagination classes
│   └── serializers.py  # Base serializers
└── common/             # Common utilities
    └── services.py     # Generic services (model_update)
```

## Key Conventions & Best Practices

### 1. Selectors (`selector/` directory)

**Rule**: Selectors MUST return `QuerySet`, never model instances.

- ✅ **Good**: `def get_user(...) -> QuerySet[BaseUser]`
- ❌ **Bad**: `def get_user(...) -> BaseUser`

**Rationale**:

- Follows Django Styleguide best practices
- Provides flexibility (caller can chain filters, use `.first()`, `.get()`, `.exists()`)
- Enables lazy evaluation and better composition
- Consistent API across all selectors

**Example**:

```python
# Selector returns QuerySet
def get_user(username: str = None, user_id: int = None, with_groups: bool = False) -> QuerySet[BaseUser]:
    # ... build query ...
    return qs

# Caller uses .get() or .first()
user = get_user(user_id=1).get()  # Raises DoesNotExist if not found
user = get_user(user_id=1).first()  # Returns None if not found
```

### 2. Services (`services/` directory)

**Rule**: Services contain business logic and orchestrate operations.

- Services call selectors to get data
- Services perform business operations (create, update, delete)
- Services raise appropriate exceptions (ValidationError, APIException, etc.)
- Services use transactions for atomic operations
- Services include logging for important events

**Example**:

```python
def create_user_service(*, user_data: dict) -> BaseUser:
    with transaction.atomic():
        user = BaseUser.objects.create_user(...)
        add_user_to_default_group(user=user)
        return user
```

### 3. Validators (`utils/validators/`)

**Rule**: Two types of validators:

1. **Boolean validators** (`utils/validators/string.py`): Return `True`/`False`

   - Functions like `validate_english(value: str) -> bool`
   - Pure validation functions without side effects

2. **Field validators** (`utils/validators/field_validators.py`): Raise `ValidationError`
   - Methods that raise exceptions on validation failure
   - Used in serializers and model validators

**Example**:

```python
# Boolean validator
def validate_english(value: str) -> bool:
    return re.fullmatch(r"^[a-zA-Z0-9_]+$", value) is not None

# Field validator
class FieldValidator:
    @staticmethod
    def validate_english_username(value: str) -> None:
        if not validate_english(value):
            raise ValidationError("Username can only contain English letters...")
```

### 4. Exception Handling

**Custom Exception Handler**: `sso/utils/handlers/custom_exception.py`

All exceptions are automatically formatted to match the standardized API response format:

```json
{
  "result": [],
  "status": <http_status_code>,
  "success": false,
  "messages": {
    "field_name": ["error message"],      // for ValidationError
    "non_field_errors": ["error message"] // for other exceptions
  }
}
```

**Usage in Views**:

- Simply raise exceptions (AuthenticationFailed, ValidationError, NotFound, etc.)
- The custom exception handler formats them automatically
- No need to manually create error responses

**Example**:

```python
# In API views - just raise exceptions
try:
    user = get_user(user_id=user_id).get()
except User.DoesNotExist:
    raise exceptions.NotFound("user not found.")
```

### 5. API Response Format

**Standardized Response Structure**:

**Success Response**:

```json
{
  "result": <data>,
  "status": 200,
  "success": true,
  "messages": {}
}
```

**Error Response**:

```json
{
  "result": [],
  "status": <http_status_code>,
  "success": false,
  "messages": {
    "field_name": ["error message"],
    "non_field_errors": ["general error message"]
  }
}
```

### 6. Serializers

**Naming Convention**:

- `*InputSerializer`: For request data
- `*OutputSerializer`: For response data
- `*SwaggerSerializer`: For Swagger documentation (extends base serializers)

**Validation**:

- Use `CustomValidationError.validate_username` for username validation
- Password validation: check `password` and `confirm_password` match
- Raise `serializers.ValidationError` (not dict format) for consistency

### 7. URL Organization

**Structure**:

- URLs separated by domain (`auth.py`, `users.py`)
- Specific routes come BEFORE parameterized routes
- Use `app_name` for namespacing

**Example**:

```python
urlpatterns = [
    path("", UserListCreateApiView.as_view(), name="list_create_user"),
    path("my/", UserMyApiView.as_view(), name="my_user"),  # Specific route first
    path("<int:user_id>/", UserRetrieveUpdateDestroyApiView.as_view(), name="..."),  # Parameterized route
]
```

### 8. Query Builders

**Pattern**: Use `QueryBuilder` base class for complex queries

- `SEARCH_FIELDS`: List of searchable fields
- `SORT_FIELDS`: List of sortable fields with display names
- `filter_query()`: Apply domain-specific filters
- `build()`: Orchestrate search, sort, and filter operations

### 9. Code Organization Principles

1. **DRY (Don't Repeat Yourself)**: Extract duplicate logic into helper functions
2. **Single Responsibility**: Each function/class has one clear purpose
3. **Type Hints**: Always include type hints for function parameters and return values
4. **Docstrings**: Use Google-style docstrings with Args, Returns, Raises sections
5. **Imports**: Use local imports when appropriate to avoid circular dependencies

### 10. Removed Features

The following features have been removed from the users app:

- ❌ `is_staff` field (removed from manager, not used)
- ❌ `is_active` field (inherited from AbstractBaseUser, not actively used)
- ❌ `avatar` field (removed from all serializers and services)
- ❌ `organization` field (removed from all serializers and services)
- ❌ `captcha` functionality (removed from login flow)

### 11. Selector Functions

**Available Selectors**:

- `get_users(*, ids: list[int] = None, with_groups: bool = False) -> QuerySet[BaseUser]`

  - Get all users or filter by IDs
  - Optional group prefetching

- `get_user(username: str = None, user_id: int = None, with_groups: bool = False) -> QuerySet[BaseUser]`
  - Get user by username and/or user_id
  - Optional group prefetching
  - Returns QuerySet (caller uses `.get()` or `.first()`)

**Removed Functions**:

- ❌ `get_user_by_id()` → Use `get_user(user_id=...)` instead
- ❌ `get_all_users()` → Use `get_users()` instead
- ❌ `get_active_user_by_id()` → Use `get_user(user_id=...)` instead
- ❌ `get_active_user_by_username()` → Use `get_user(username=...)` instead

### 12. Service Functions

**Auth Services**:

- `authenticate_user_service(username: str, password: str) -> BaseUser`
- `get_tokens_for_user_service(user: BaseUser) -> dict`
- `refresh_user_token_service(refresh_token: str) -> dict`
- `auth_login_service(username: str, password: str) -> dict`

**User Services**:

- `create_user_service(*, user_data: dict) -> BaseUser`
- `full_update_user_my(*, user: BaseUser, user_data: dict) -> BaseUser`
- `partial_update_user_my(*, user: BaseUser, user_data: dict) -> BaseUser`
- `update_user_without_avatar(*, user: BaseUser, user_data: dict, actor: BaseUser) -> BaseUser`
- `delete_user(*, user: BaseUser) -> bool`
- `change_user_password(*, user: BaseUser, data: dict, skip_current_password_check: bool = False) -> BaseUser`

### 13. Error Handling

**Integrity Errors**: Handled by `sso/utils/errors/integrity.py`

- Automatically converts Django IntegrityError to appropriate DRF exceptions
- Handles unique violations, foreign key violations, NOT NULL violations

**Custom Exception Handler**: `sso/utils/handlers/custom_exception.py`

- Formats all exceptions to standardized response format
- Handles ValidationError with field-specific errors
- Handles other exceptions (AuthenticationFailed, NotFound, etc.) with `non_field_errors`

### 14. Swagger/OpenAPI

**Custom Schema Decorator**: `sso/utils/swagger/custom_extend_schema.py`

- Automatically builds response serializers for Swagger
- Supports multiple status codes
- Generates error response schemas automatically

**Base Serializers**: `sso/api/serializers.py`

- `BaseOutputSwaggerSerializer`: Base for all responses
- `ListPaginatedResultOutputSwaggerSerializer`: For paginated lists
- `ListAllOutputSwaggerSerializer`: For unpaginated lists

### 15. Important Files

**Settings**:

- `config/settings/drf.py`: DRF configuration including `EXCEPTION_HANDLER`

**Constants**:

- `sso/users/constants.py`: App-specific constants (tags, parameters, exclude groups)

**Handlers**:

- `sso/utils/handlers/custom_exception.py`: Custom exception handler
- `sso/utils/handlers/integrity_error.py`: Integrity error handling

**Validators**:

- `sso/utils/validators/string.py`: Boolean validation functions
- `sso/utils/validators/field_validators.py`: Field validators that raise exceptions

### 16. Code Quality Standards

- ✅ All functions have type hints
- ✅ All functions have comprehensive docstrings (Args, Returns, Raises)
- ✅ No linter errors
- ✅ Consistent error handling
- ✅ Proper logging for security events
- ✅ DRY principle (no code duplication)
- ✅ Follows Django Styleguide recommendations

### 17. Testing Considerations

- Selectors return QuerySets (easy to mock and test)
- Services are pure functions (easy to unit test)
- Exception handler formats errors consistently (easy to test error responses)

### 18. Security

- Authentication uses JWT tokens (access_token + refresh_token)
- Password validation enforces matching passwords
- Username validation restricts to English letters, numbers, underscores
- Security events are logged (failed login attempts, token refresh failures)
- Generic error messages (don't reveal if username exists)

## Quick Reference

### When to use Selectors vs Services

**Selectors**: Data retrieval only

- Returns QuerySets
- No business logic
- No side effects

**Services**: Business logic

- Calls selectors
- Performs operations (create, update, delete)
- Uses transactions
- Raises exceptions

### When to raise exceptions

- **In Services**: Raise appropriate exceptions (ValidationError, APIException, AuthenticationFailed)
- **In Views**: Catch exceptions and let custom exception handler format them
- **In Serializers**: Raise `serializers.ValidationError` for validation failures

### Response Format

Always use the standardized format:

- Success: `{"result": <data>, "status": 200, "success": true, "messages": {}}`
- Error: `{"result": [], "status": <code>, "success": false, "messages": {...}}`

The custom exception handler automatically formats all exceptions to this structure.
