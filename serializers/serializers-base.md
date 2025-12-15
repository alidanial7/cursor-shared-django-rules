---
alwaysApply: true
---

# Django REST Framework Serializers

This document defines best practices for **serializers** in Django REST Framework projects.

---

## Core Concept

A **serializer**:

- ✅ Converts complex data types (models, querysets) to/from Python native types
- ✅ Handles validation of incoming data
- ✅ Defines API input/output structure
- ✅ Provides field-level and object-level validation
- ❌ Does not contain business logic (use services for that)
- ❌ Does not perform database queries (use selectors for that)
- ❌ Does not handle permissions (use views/mixins for that)

> **Comment:** Serializers focus on **data transformation and validation**. Business logic belongs in services.

---

## Naming Conventions

### ✅ Correct Naming

```python
# Input serializers (for request data)
class UserCreateInputSerializer(serializers.Serializer):
    pass

class UserUpdateInputSerializer(serializers.Serializer):
    pass

# Output serializers (for response data)
class UserOutputSerializer(serializers.Serializer):
    pass

class UserListOutputSerializer(serializers.Serializer):
    pass

# Swagger serializers (for documentation)
class UserSwaggerSerializer(serializers.Serializer):
    pass
```

### ❌ Incorrect Naming

```python
# ❌ Forbidden: Generic names
class UserSerializer(serializers.Serializer):
    pass

# ❌ Forbidden: Unclear purpose
class UserDataSerializer(serializers.Serializer):
    pass
```

**Why:** Clear naming indicates the serializer's purpose (input/output) and improves code readability.

---

## Serializer Types

### 1. Input Serializers (`*InputSerializer`)

Used for **validating and deserializing** incoming request data.

```python
from rest_framework import serializers

class UserCreateInputSerializer(serializers.Serializer):
    username = serializers.CharField(max_length=150)
    email = serializers.EmailField()
    password = serializers.CharField(write_only=True, min_length=8)
    confirm_password = serializers.CharField(write_only=True)

    def validate_username(self, value):
        # Field-level validation
        if not value.isalnum():
            raise serializers.ValidationError("Username must be alphanumeric.")
        return value

    def validate(self, attrs):
        # Object-level validation
        if attrs['password'] != attrs['confirm_password']:
            raise serializers.ValidationError({
                'password': 'Passwords do not match.'
            })
        return attrs
```

**Use Case:** POST, PUT, PATCH requests

### 2. Output Serializers (`*OutputSerializer`)

Used for **serializing** response data.

```python
from rest_framework import serializers

class UserOutputSerializer(serializers.Serializer):
    id = serializers.IntegerField(read_only=True)
    username = serializers.CharField(read_only=True)
    email = serializers.EmailField(read_only=True)
    date_joined = serializers.DateTimeField(read_only=True)
    is_active = serializers.BooleanField(read_only=True)
```

**Use Case:** GET requests, response formatting

### 3. Model Serializers

Use `ModelSerializer` when serializer closely maps to a model.

```python
from rest_framework import serializers
from users.models import User

class UserModelSerializer(serializers.ModelSerializer):
    class Meta:
        model = User
        fields = ['id', 'username', 'email', 'date_joined']
        read_only_fields = ['id', 'date_joined']
```

**When to Use:** When serializer fields directly map to model fields

---

## Validation Patterns

### Field-Level Validation

```python
class UserCreateInputSerializer(serializers.Serializer):
    username = serializers.CharField()

    def validate_username(self, value):
        # Custom field validation
        if User.objects.filter(username=value).exists():
            raise serializers.ValidationError("Username already exists.")
        return value
```

### Object-Level Validation

```python
class UserCreateInputSerializer(serializers.Serializer):
    password = serializers.CharField()
    confirm_password = serializers.CharField()

    def validate(self, attrs):
        # Validate multiple fields together
        if attrs['password'] != attrs['confirm_password']:
            raise serializers.ValidationError({
                'password': 'Passwords do not match.'
            })
        return attrs
```

### ✅ Correct: Raise ValidationError

```python
# ✅ Correct
def validate_username(self, value):
    if not value.isalnum():
        raise serializers.ValidationError("Username must be alphanumeric.")
    return value
```

### ❌ Forbidden: Return Error Dicts

```python
# ❌ Forbidden
def validate_username(self, value):
    if not value.isalnum():
        return {"error": "Invalid username"}  # ❌ Wrong format
    return value
```

**Why:** Always raise `serializers.ValidationError` for consistency with DRF's error handling.

---

## Field Options

### Common Field Options

```python
class UserSerializer(serializers.Serializer):
    # Read-only fields (for output)
    id = serializers.IntegerField(read_only=True)

    # Write-only fields (for input, not in response)
    password = serializers.CharField(write_only=True)

    # Required vs optional
    username = serializers.CharField(required=True)
    bio = serializers.CharField(required=False, allow_blank=True)

    # Nested serializers
    profile = ProfileSerializer(read_only=True)

    # Method fields (computed values)
    full_name = serializers.SerializerMethodField()

    def get_full_name(self, obj):
        return f"{obj.first_name} {obj.last_name}"
```

---

## Usage in Views

### ✅ Correct Usage

```python
from rest_framework import views, status
from rest_framework.response import Response

class UserCreateApiView(views.APIView):
    def post(self, request):
        serializer = UserCreateInputSerializer(data=request.data)
        serializer.is_valid(raise_exception=True)

        # Use service layer for business logic
        user = create_user_service(user_data=serializer.validated_data)

        # Use output serializer for response
        output_serializer = UserOutputSerializer(user)
        return Response(output_serializer.data, status=status.HTTP_201_CREATED)
```

### ❌ Forbidden: Business Logic in Serializers

```python
# ❌ Forbidden
class UserCreateInputSerializer(serializers.Serializer):
    def create(self, validated_data):
        # ❌ Don't create objects in serializer
        user = User.objects.create(**validated_data)
        send_welcome_email(user)  # ❌ Business logic
        return user
```

**Why:** Use services for business logic. Serializers should only validate and transform data.

---

## Anti-Patterns (Forbidden)

### ❌ Forbidden: Database Queries in Serializers

```python
# ❌ Forbidden
class UserSerializer(serializers.Serializer):
    def validate_username(self, value):
        # ❌ Direct database query
        if User.objects.filter(username=value).exists():
            raise serializers.ValidationError("Username exists.")
        return value
```

**Why:** Use selectors for database queries. Keep serializers focused on validation.

### ❌ Forbidden: Business Logic in Serializers

```python
# ❌ Forbidden
class UserSerializer(serializers.Serializer):
    def validate(self, attrs):
        # ❌ Business logic
        if attrs['role'] == 'admin' and not self.context['user'].is_staff:
            raise PermissionDenied()
        return attrs
```

**Why:** Permission checks and business rules belong in views or services.

### ❌ Forbidden: Returning Error Dicts Instead of Raising

```python
# ❌ Forbidden
def validate_username(self, value):
    if not value.isalnum():
        return {"error": "Invalid"}  # ❌ Wrong format
    return value
```

**Why:** Always raise `serializers.ValidationError` for proper error handling.

---

## Best Practices

1. **Separate Input and Output**: Use different serializers for request and response data
2. **Use Descriptive Names**: `*InputSerializer` and `*OutputSerializer` clearly indicate purpose
3. **Validate Early**: Use field-level and object-level validation
4. **Keep It Simple**: Serializers validate and transform, nothing more
5. **Raise ValidationError**: Always use `serializers.ValidationError` for consistency
6. **Use Services**: Delegate business logic to service layer
7. **Use Selectors**: Delegate database queries to selectors

---

## Summary

- ✅ Use `*InputSerializer` for request validation
- ✅ Use `*OutputSerializer` for response formatting
- ✅ Raise `serializers.ValidationError` for validation errors
- ✅ Keep serializers focused on data transformation and validation
- ✅ Delegate business logic to services
- ✅ Delegate database queries to selectors
- ❌ No business logic in serializers
- ❌ No direct database queries in serializers
- ❌ No permission checks in serializers

**Comment:** Serializers are the boundary between API and application logic. Keep them thin and focused.
