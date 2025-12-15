---
alwaysApply: true
---

# Django REST Framework API Views

All API views should be class-based views and their name must end with `ApiView`.

### ✅ Correct Usage

```python
from rest_framework.views import APIView

class UserListApiView(APIView):
    pass

class ProductDetailApiView(APIView):
    pass
```

### ❌ Incorrect Usage

```python
class UserList(APIView):  # ❌ Missing ApiView suffix
    pass

def user_list(request):  # ❌ Should be class-based view
    pass
```