---
alwaysApply: true
---

# Django URL Configuration

## URL Structure

### Base URL (`config/urls.py`)

The base URL configuration includes API schema, documentation, admin, and app routing.

```python
from django.conf import settings
from django.conf.urls.static import static
from django.urls import include, path

urlpatterns = [
    path("api/schema/", TranslatableSchemaView.as_view(), name="schema"),
    path(
        "api/docs/",
        SpectacularSwaggerView.as_view(url_name="schema"),
        name="swagger-ui",
    ),
    path("api/redoc/", SpectacularRedocView.as_view(url_name="schema"), name="redoc"),
    path("api/admin/", admin.site.urls),
    path("api/", include(("app_name.api.urls", "api"))),
] + static(settings.MEDIA_URL, document_root=settings.MEDIA_ROOT)
```

### App API URLs (`app_name/api/urls.py`)

Routes to individual app URL configurations.

```python
from django.urls import include, path

urlpatterns = [
    path("auth/", include(("app_name.users.urls.auth", "auth"))),
    path("captcha/", include(("app_name.captcha.urls", "captcha"))),
    path("users/", include(("app_name.users.urls.users", "users"))),
]
```

## URL File Structure

### Simple Apps

For simple apps with a single URL file:

```python
from django.urls import path

from app_name.captcha.apis import CaptchaApiView

app_name = "captcha"

urlpatterns = [
    path(
        "",
        CaptchaApiView.as_view(),
        name="get_captcha",
    ),
]
```

### Complex Apps

For apps with multiple URL files, organize them in a `urls/` folder:

```
app_name/
└── urls/
    ├── auth.py
    └── users.py
```

Each file follows the same structure as simple apps:

```python
from django.urls import path

from app_name.users.apis.users import (
    UserChangePasswordApiView,
    UserListCreateApiView,
    UserMyApiView,
    UserMyChangePasswordApiView,
    UserRetrieveUpdateDestroyApiView,
)

app_name = "users"

urlpatterns = [
    path("", UserListCreateApiView.as_view(), name="list_create_user"),
    path("my/", UserMyApiView.as_view(), name="my_user"),
    path(
        "my/change_password/",
        UserMyChangePasswordApiView.as_view(),
        name="my_change_password_user",
    ),
    # Parameterized routes
    path(
        "<int:user_id>/",
        UserRetrieveUpdateDestroyApiView.as_view(),
        name="retrieve_update_destroy_user",
    ),
    path(
        "<int:user_id>/change_password/",
        UserChangePasswordApiView.as_view(),
        name="change_password_user",
    ),
]
```

## Rules

- **URLs must be in alphabetical order** within each `urlpatterns` list
- Always include `app_name` for namespace consistency
- Use descriptive `name` parameters for URL reversing
- Group related routes together with comments