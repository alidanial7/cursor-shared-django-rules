---
alwaysApply: true
---

# Django Selectors

This document defines best practices for **selectors** in Django projects.  
Selectors follow **single responsibility principle**: they only query data, no business logic.

> **Note:** Example uses `User`, but pattern applies to any model.

---

## Core Concept

A **selector**:

- ✅ Reads data from the database only
- ✅ Uses ORM queries (`filter`, `get`, `annotate`, `Q`, etc.)
- ✅ Defines **cardinality** (many / one / optional)
- ❌ Does not contain business logic
- ❌ Does not perform validation or permissions
- ❌ Does not modify data
- ❌ Does not catch ORM exceptions

> **Comment:** Separates **data access** from **business logic**. Services handle rules, validations, and permissions.

---

## Cardinality & Return Types

| Function           | Cardinality | Return Type       | Behavior                                           | When to Use                                          |
| ------------------ | ----------- | ----------------- | -------------------------------------------------- | ---------------------------------------------------- |
| `get_items`        | Many        | `QuerySet[Model]` | Returns multiple objects (can be empty)            | Use when multiple results are possible               |
| `get_item`         | One         | `Model`           | Raises `DoesNotExist` or `MultipleObjectsReturned` | Use when exactly one object is expected              |
| `get_item_or_none` | Zero or One | `Model` or `None` | Returns a single object or `None`                  | Use when object may not exist; caller handles `None` |

**Comment:** Cardinality drives **return type**, **exception handling**, and **selector naming**.

---

## Why Use `Q` Objects

`Q` objects allow:

- Flexible combination of conditions (AND / OR)
- Handling optional filter parameters easily
- Cleaner and more maintainable code
- Easy future extension

### ✅ Correct Usage

```python
from django.db.models import Q
from django.db.models import QuerySet
from typing import Optional

def get_users(*, name: Optional[str] = None, id: Optional[int] = None) -> QuerySet:
    q = Q()
    if name is not None:
        q &= Q(name=name)
    if id is not None:
        q &= Q(id=id)

    return User.objects.filter(q)
```

**Comment:** `Q` objects are especially useful for selectors with multiple optional parameters.

---

## Prefetch Related Fields

Use `_prefetch_related(queryset, related_name)` to prefetch related objects efficiently.

### ✅ Correct Implementation

```python
from django.db.models import Prefetch, QuerySet
from typing import Optional, Dict, Any

def _prefetch_related(
    queryset: QuerySet,
    related_name: str,
    exclude: Optional[Dict[str, Any]] = None
) -> QuerySet:
    """
    Prefetch a related field with optional exclusion.

    Args:
        queryset: The queryset to prefetch on
        related_name: Name of the related field to prefetch
        exclude: Optional dict of exclusion filters

    Comment: Improves query efficiency by avoiding N+1 queries.
    """
    try:
        related_field = queryset.model._meta.get_field(related_name)
        related_model = related_field.related_model
        prefetch_qs = related_model.objects.all()

        if exclude:
            prefetch_qs = prefetch_qs.exclude(**exclude)

        return queryset.prefetch_related(
            Prefetch(related_name, queryset=prefetch_qs)
        )
    except Exception:
        # If field doesn't exist or is not a relation, return queryset unchanged
        return queryset
```

**Comment:** Keep selectors flexible by prefetching only when needed. Always handle edge cases gracefully.

---

## Internal Helper: `_build_queryset`

### ✅ Correct Implementation

```python
from django.db.models import Q, QuerySet
from typing import Optional, Dict, Any, Union
from django.db import models

def _build_queryset(
    model: type[models.Model],
    *,
    filters: Optional[Union[Dict[str, Any], Q]] = None,
    with_related: Optional[Dict[str, Optional[Dict[str, Any]]]] = None
) -> QuerySet:
    """
    Build a reusable queryset based on optional filters and prefetch related fields.

    Args:
        model: Django model class
        filters: Optional dict or Q object for filtering
        with_related: Optional dict mapping related names to exclusion filters

    Comment: Centralizes all filter and prefetch logic to avoid repetition.
    Supports both dict filters and Q objects for maximum flexibility.
    """
    if filters is None:
        qs = model.objects.all()
    elif isinstance(filters, Q):
        qs = model.objects.filter(filters)
    else:
        qs = model.objects.filter(**filters)

    if with_related:
        for related_name, exclude in with_related.items():
            qs = _prefetch_related(qs, related_name, exclude)

    return qs
```

**Comment:** Supports both dict filters and `Q` objects for maximum flexibility.

---

## Public Selectors (Generalized)

### 1. `get_items` - Many

```python
from django.db.models import QuerySet, Q
from typing import Optional, Dict, Any, Union
from django.db import models

def get_items(
    model: type[models.Model],
    *,
    filters: Optional[Union[Dict[str, Any], Q]] = None,
    with_related: Optional[Dict[str, Optional[Dict[str, Any]]]] = None
) -> QuerySet:
    """
    Retrieve multiple objects based on optional filters.

    Returns:
        QuerySet: Always returns QuerySet, never raises exceptions. Can return empty.

    Comment: Always returns QuerySet, never raises exceptions. Can return empty.
    Caller can chain additional filters, use `.first()`, `.get()`, `.exists()`, etc.
    """
    return _build_queryset(model, filters=filters, with_related=with_related)
```

**Cardinality:** Many  
**Use Case:** List views, filtering, when multiple results are expected

### 2. `get_item` - One

```python
from django.db.models import QuerySet, Q
from typing import Optional, Dict, Any, Union
from django.db import models

def get_item(
    model: type[models.Model],
    *,
    filters: Optional[Union[Dict[str, Any], Q]] = None,
    with_related: Optional[Dict[str, Optional[Dict[str, Any]]]] = None
) -> models.Model:
    """
    Retrieve exactly one object. Raises ORM exceptions if not found or duplicated.

    Returns:
        Model: Single model instance

    Raises:
        model.DoesNotExist: When no object matches
        model.MultipleObjectsReturned: When multiple objects match

    Comment: Use only when exactly one object is expected. Let exceptions propagate
    to the service layer for proper handling.
    """
    qs = _build_queryset(model, filters=filters, with_related=with_related)
    return qs.get()
```

**Cardinality:** One  
**Use Case:** Detail views, when unique identifier is provided (e.g., primary key)

### 3. `get_item_or_none` - Zero or One

```python
from django.db.models import QuerySet, Q
from typing import Optional, Dict, Any, Union
from django.db import models

def get_item_or_none(
    model: type[models.Model],
    *,
    filters: Optional[Union[Dict[str, Any], Q]] = None,
    with_related: Optional[Dict[str, Optional[Dict[str, Any]]]] = None
) -> Optional[models.Model]:
    """
    Retrieve a single object if exists, otherwise returns None.

    Returns:
        Model or None: Single model instance or None if not found

    Comment: Always returns safely, never raises exceptions. Caller must handle None.
    """
    qs = _build_queryset(model, filters=filters, with_related=with_related)
    return qs.first()
```

**Cardinality:** Zero or One  
**Use Case:** Optional lookups, existence checks, when object may not exist

---

## Usage Examples

### Example 1: Using `get_items` with filters

```python
# Get all active users
active_users = get_items(User, filters={"is_active": True})

# Get users with prefetch
users_with_profiles = get_items(
    User,
    filters={"is_active": True},
    with_related={"profile": None}
)

# Chain additional operations
count = get_items(User, filters={"is_active": True}).count()
```

### Example 2: Using `get_item` for unique lookups

```python
# In service layer - handle exceptions
try:
    user = get_item(User, filters={"id": user_id})
except User.DoesNotExist:
    raise NotFound("User not found")
```

### Example 3: Using `get_item_or_none` for optional lookups

```python
# Check if user exists
user = get_item_or_none(User, filters={"email": email})
if user is None:
    # Handle case when user doesn't exist
    pass
```

### Example 4: Using `Q` objects for complex filters

```python
from django.db.models import Q

# Complex filter with OR condition
q = Q(is_active=True) | Q(is_staff=True)
active_or_staff = get_items(User, filters=q)

# Combining multiple conditions
q = Q()
if name:
    q &= Q(first_name__icontains=name) | Q(last_name__icontains=name)
if email:
    q &= Q(email__icontains=email)
users = get_items(User, filters=q)
```

---

## Anti-Patterns (Forbidden)

### ❌ Forbidden: Business Logic in Selectors

```python
# ❌ Forbidden
def get_items_with_check(model, filters):
    qs = model.objects.filter(**filters)
    if not qs.exists():
        raise ValidationError("No objects")
    return qs
```

**Why:** Validation and business rules belong in the service layer, not selectors.

### ❌ Forbidden: Catching ORM Exceptions

```python
# ❌ Forbidden
def get_item_safe(model, filters):
    try:
        return model.objects.filter(**filters).get()
    except model.DoesNotExist:
        return None
```

**Why:** Use `get_item_or_none` instead. Let exceptions propagate to service layer.

### ❌ Forbidden: Writing to Database

```python
# ❌ Forbidden
def get_items_and_update(model, filters):
    qs = model.objects.filter(**filters)
    qs.update(last_seen=timezone.now())  # ❌ Modifying data
    return qs
```

**Why:** Selectors only read data. Use services for modifications.

### ❌ Forbidden: Returning Model Instances from `get_items`

```python
# ❌ Forbidden
def get_items(model, filters):
    return list(model.objects.filter(**filters))  # ❌ Returns list, not QuerySet
```

**Why:** Always return `QuerySet` for flexibility and lazy evaluation.

### ❌ Forbidden: Permission Checks in Selectors

```python
# ❌ Forbidden
def get_items_for_user(model, filters, user):
    qs = model.objects.filter(**filters)
    if not user.has_perm('view_model'):
        raise PermissionDenied()  # ❌ Permission check
    return qs
```

**Why:** Permission checks belong in views or services, not selectors.

---

## Summary

- ✅ Only three public selectors: `get_items`, `get_item`, `get_item_or_none`
- ✅ `_build_queryset` centralizes filter and prefetch logic
- ✅ Supports both dict filters and `Q` objects for maximum flexibility
- ✅ `Q` objects enable flexible and maintainable filtering
- ✅ Cardinality drives return type and exception handling
- ✅ Always return `QuerySet` from `get_items` for maximum flexibility
- ✅ Service layer handles business rules, validations, and permissions
- ✅ Let ORM exceptions propagate to service layer for proper handling

**Comment:** Pattern is fully generic and can be applied to any Django model, not just User.
