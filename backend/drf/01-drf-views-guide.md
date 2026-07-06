# DRF — Types of Views

## 1. Overview

Django REST Framework offers 3 levels of views (class-based). For each endpoint, you pick one.

| Level | Status |
|---|---|
| APIView | base, always useful |
| Generic Views | widely used (clean, fast) |
| ViewSets | pro standard (most used in production) |

Function-based views (`@api_view`) also exist but are rarely used in professional environments.

**Typical modern endpoint structure:**
1. Serializer
2. ViewSet or APIView
3. Router / urls
4. Auth / permissions

---

## 2. APIView

The most explicit level: you write `get` / `post` / `put` / `delete` by hand.

**When to use it:**
- custom endpoints (login, health-check, special business action)
- logic that doesn't fit a standard CRUD
- when you want full control over the request/response

**Example:**
```python
class HelloAPIView(APIView):
    def post(self, request):
        ...
```

**Pros:** full control, explicit, flexible
**Cons:** more code needed for a complete CRUD

---

## 3. Generic Views

Ready-made views for standard CRUD operations, to combine with mixins or use directly.

| View | Used for |
|---|---|
| `ListAPIView` | list only |
| `CreateAPIView` | create only |
| `RetrieveAPIView` | detail |
| `UpdateAPIView` | update |
| `DestroyAPIView` | delete |
| `ListCreateAPIView` | list + create |
| `RetrieveUpdateDestroyAPIView` | detail + update + delete |

**Example:**
```python
class ProductListCreateView(ListCreateAPIView):
    queryset = Product.objects.all()
    serializer_class = ProductSerializer
```

You can also compose your own with mixins:
```python
class X(mixins.ListModelMixin, GenericAPIView):
    ...
```

**Pros:** fast to write, clean, less code, pro standard

---

## 4. ViewSets

The highest level: a single class handles the whole CRUD (`list`, `create`, `retrieve`, `update`, `destroy`), wired to a router.

**Example:**
```python
class ProductViewSet(ModelViewSet):
    queryset = Product.objects.all()
    serializer_class = ProductSerializer
```

```python
router.register("products", ProductViewSet)
```

Which automatically generates:
```
GET    /products/
POST   /products/
GET    /products/1/
PUT    /products/1/
DELETE /products/1/
```

**Pros:** ultra fast, pro standard, clean, scalable
**Cons:** less explicit for a beginner, since a lot of logic is implicit

---

## 5. Which view to choose?

| Situation | View to use |
|---|---|
| custom endpoint | APIView |
| simple CRUD | ModelViewSet ⭐ |
| partial CRUD (list only, create only, etc.) | Generic Views |
| special business logic | APIView |

**Typical distribution observed in production (2026):**
- ~70% ViewSets
- ~20% APIView (login, payment, special actions)
- ~10% Generic Views

**Separation-of-concerns pattern to keep in mind:**
- **serializer** → data validation
- **view** → request/response orchestration
- **service** → business logic (once it grows)

---

## 6. Organizing serializers in a project

Convention: each Django app has its own `serializers.py`.

```
project/
├── users/
│   ├── models.py
│   ├── serializers.py
│   ├── views.py
│   └── urls.py
│
├── products/
│   ├── models.py
│   ├── serializers.py
│   ├── views.py
│   └── urls.py
```

**Simple rule:** 1 Django app = its own serializers.

**Why:**
- modularity
- readability
- scalability
- pro architecture (ready for an eventual split into microservices)

Within a single app, you can have:
- several serializers
- one serializer per model
- special-purpose serializers (login, register, etc.)
