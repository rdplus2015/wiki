# Chapter 3 — Routers, Pagination, Filtering

## 3.1 Routers — automatic URL generation

A router takes a `ViewSet` and automatically generates all the corresponding URLs (see Chapter 2, section 2.9 for the detail of the HTTP method → action mapping).

```python
# urls.py
from django.urls import path, include
from rest_framework.routers import DefaultRouter
from .views import NoteViewSet, CategoryViewSet

router = DefaultRouter()
# register(prefix, viewset, basename)
# prefix: URL segment ('notes' → /notes/ and /notes/{pk}/)
# basename: used to name the URL patterns (note-list, note-detail)
router.register('notes', NoteViewSet, basename='note')
router.register('categories', CategoryViewSet, basename='category')

urlpatterns = [
    path('api/', include(router.urls)),
]

# Generated URLs:
# GET/POST                api/notes/
# GET/PUT/PATCH/DELETE    api/notes/{pk}/
# POST                    api/notes/{pk}/pin/        (an @action)
# GET                     api/notes/pinned/          (an @action)
# GET/POST                api/categories/
# GET/PUT/PATCH/DELETE    api/categories/{pk}/
```

`basename` is required when your ViewSet doesn't have a `queryset` attribute defined directly on the class (e.g. if everything goes through `get_queryset()`) — DRF can't guess the name automatically in that case, so you must specify it.

`DefaultRouter` also generates a root `api/` route listing every available endpoint — handy during development to explore the API from the browser (browsable API).

**`SimpleRouter` vs `DefaultRouter`:** `SimpleRouter` does the same thing but without the root route or the `.json`/`.api` suffix format. Use `DefaultRouter` in development; both are equivalent in production once the frontend is locked onto its URLs.

---

## 3.2 Pagination

Without pagination, `GET /notes/` would return *all* of the user's notes in a single response. On a dataset with thousands of rows, that's disastrous for performance and response size. Pagination splits results into pages.

### Global configuration

```python
# settings.py
REST_FRAMEWORK = {
    'DEFAULT_PAGINATION_CLASS': 'rest_framework.pagination.PageNumberPagination',
    'PAGE_SIZE': 20,  # Number of items per page by default
}
```

### The three pagination strategies

```python
# pagination.py
from rest_framework.pagination import (
    PageNumberPagination,
    LimitOffsetPagination,
    CursorPagination
)


class NotePageNumberPagination(PageNumberPagination):
    """
    Strategy: page numbers
    URL: /notes/?page=2
    Response:
    {
        "count": 150,
        "next": "http://api/notes/?page=3",
        "previous": "http://api/notes/?page=1",
        "results": [...]
    }

    Pro: simple, lets you jump to any page
    Con: unstable data if rows are inserted while navigating
    """
    page_size = 10
    page_size_query_param = 'page_size'  # Client can control the page size
    max_page_size = 100  # Safety limit


class NoteLimitOffsetPagination(LimitOffsetPagination):
    """
    Strategy: limit + offset (like SQL)
    URL: /notes/?limit=10&offset=20

    Pro: flexible, intuitive for developers
    Con: degraded performance on large offsets (SQL has to scan)
    """
    default_limit = 10
    max_limit = 100


class NoteCursorPagination(CursorPagination):
    """
    Strategy: opaque cursor (the most robust)
    URL: /notes/?cursor=cD0yMDIz...  (encoded cursor)

    Pro: constant performance, stable even with ongoing inserts
    Con: can't jump to page N, sequential navigation only
    Ideal use: real-time feeds, large tables
    """
    page_size = 10
    ordering = '-created_at'  # Required for CursorPagination
```

**How to choose:**

| Need | Strategy |
|---|---|
| Classic admin UI with "page 1, 2, 3..." | `PageNumberPagination` |
| API consumed by external devs, flexibility expected | `LimitOffsetPagination` |
| Real-time feed (social, notifications), large table | `CursorPagination` |

### Per-ViewSet pagination

```python
class NoteViewSet(viewsets.ModelViewSet):
    pagination_class = NotePageNumberPagination  # Overrides the global pagination
    # ...

    # To disable pagination on a specific action:
    @action(detail=False, methods=['get'])
    def pinned(self, request):
        self.pagination_class = None  # No pagination for pinned notes
        notes = self.get_queryset().filter(is_pinned=True)
        serializer = self.get_serializer(notes, many=True)
        return Response(serializer.data)
```

---

## 3.3 Filtering — SearchFilter, OrderingFilter, django-filter

### SearchFilter — text search

```python
from rest_framework.filters import SearchFilter, OrderingFilter


class NoteViewSet(viewsets.ModelViewSet):
    filter_backends = [SearchFilter, OrderingFilter]

    # SearchFilter: searches these fields via ?search=term
    # Prefix ^ = startswith, = = exact, @ = full-text search (PostgreSQL only), $ = regex
    search_fields = ['title', 'body', 'category__name']

    # OrderingFilter: sort via ?ordering=created_at or ?ordering=-created_at
    # The - means descending
    ordering_fields = ['created_at', 'updated_at', 'title']
    ordering = ['-created_at']  # Default sort
```

`search_fields` with `category__name` shows you can traverse a relation (FK) directly in search — DRF translates that to `category__name__icontains` internally.

### django-filter — advanced field filtering

```bash
pip install django-filter
```

```python
# settings.py
INSTALLED_APPS = ['django_filters', ...]

REST_FRAMEWORK = {
    'DEFAULT_FILTER_BACKENDS': ['django_filters.rest_framework.DjangoFilterBackend'],
}
```

```python
# filters.py
import django_filters
from .models import Note


class NoteFilter(django_filters.FilterSet):
    """
    Custom FilterSet: defines more expressive filters than simple filtering.

    A FilterSet generates query params: /notes/?is_pinned=true&category=5&created_after=2024-01-01
    """

    # Filter on a date range
    # created_after → Note.objects.filter(created_at__gte=value)
    created_after = django_filters.DateTimeFilter(
        field_name='created_at',
        lookup_expr='gte'  # gte = greater than or equal
    )
    created_before = django_filters.DateTimeFilter(
        field_name='created_at',
        lookup_expr='lte'
    )

    # Filter by title (contains, case-insensitive)
    title = django_filters.CharFilter(lookup_expr='icontains')

    # Exact filter on is_pinned
    is_pinned = django_filters.BooleanFilter()

    # Filter by category (exact ID)
    category = django_filters.NumberFilter(field_name='category_id')

    # "No category" filter: /notes/?no_category=true
    no_category = django_filters.BooleanFilter(
        field_name='category',
        lookup_expr='isnull'
    )

    class Meta:
        model = Note
        fields = ['is_pinned', 'category', 'title', 'created_after', 'created_before']
```

```python
# views.py
from django_filters.rest_framework import DjangoFilterBackend
from rest_framework.filters import SearchFilter, OrderingFilter
from .filters import NoteFilter


class NoteViewSet(viewsets.ModelViewSet):
    filter_backends = [DjangoFilterBackend, SearchFilter, OrderingFilter]
    filterset_class = NoteFilter          # Our custom FilterSet
    search_fields = ['title', 'body']    # SearchFilter: ?search=term
    ordering_fields = ['created_at', 'title']
    ordering = ['-created_at']

    # All three backends coexist:
    # ?search=django          → SearchFilter (fulltext on title + body)
    # ?is_pinned=true         → DjangoFilterBackend (exact filtering)
    # ?ordering=title         → OrderingFilter (sort)
    # ?search=python&is_pinned=true&ordering=-created_at → all combined
```

### Simple filtering without a FilterSet (shortcut)

If you only need exact filtering on a few fields, with no custom logic (`lookup_expr`, date ranges...), you can skip the `FilterSet` and declare the fields directly:

```python
class NoteViewSet(viewsets.ModelViewSet):
    filter_backends = [DjangoFilterBackend]
    filterset_fields = ['is_pinned', 'category']  # strict equality only
    # equivalent to a minimal auto-generated FilterSet
```

**When to use which:**
- `filterset_fields`: simple filters, strict equality, no custom logic
- `filterset_class` (FilterSet): date ranges, `icontains`, combined filters, fields that don't literally exist on the model

---

## 3.4 Nested routes

A classic `DefaultRouter` only generates flat routes: `/notes/`, `/notes/{pk}/`. But some resources only make sense **under** another one — for example comments that always belong to a specific ticket: `/tickets/{ticket_id}/comments/`.

### Option 1 — `drf-nested-routers` (external package)

```bash
pip install drf-nested-routers
```

```python
# urls.py
from rest_framework_nested import routers
from .views import TicketViewSet, CommentViewSet

router = routers.DefaultRouter()
router.register('tickets', TicketViewSet, basename='ticket')

# Nested router: every 'comments' route will be prefixed with /tickets/{ticket_pk}/
comments_router = routers.NestedDefaultRouter(router, 'tickets', lookup='ticket')
comments_router.register('comments', CommentViewSet, basename='ticket-comments')

urlpatterns = [
    path('api/', include(router.urls)),
    path('api/', include(comments_router.urls)),
]

# Generates:
# /api/tickets/                              GET, POST
# /api/tickets/{pk}/                         GET, PUT, PATCH, DELETE
# /api/tickets/{ticket_pk}/comments/         GET, POST
# /api/tickets/{ticket_pk}/comments/{pk}/    GET, PUT, PATCH, DELETE
```

```python
# views.py
class CommentViewSet(viewsets.ModelViewSet):
    serializer_class = CommentSerializer
    permission_classes = [IsAuthenticated]

    def get_queryset(self):
        # 'ticket_pk' comes from the lookup='ticket' declared on NestedDefaultRouter
        return Comment.objects.filter(ticket_id=self.kwargs['ticket_pk'])

    def perform_create(self, serializer):
        serializer.save(
            author=self.request.user,
            ticket_id=self.kwargs['ticket_pk']  # injects the parent ticket from the URL
        )
```

### Option 2 — no external dependency (manual)

If you want to avoid the dependency, you can route manually with a plain `path()` pointing to the ViewSet's actions:

```python
# urls.py
from .views import CommentViewSet

comment_list = CommentViewSet.as_view({'get': 'list', 'post': 'create'})
comment_detail = CommentViewSet.as_view({'get': 'retrieve', 'put': 'update', 'delete': 'destroy'})

urlpatterns = [
    path('api/tickets/<int:ticket_pk>/comments/', comment_list),
    path('api/tickets/<int:ticket_pk>/comments/<int:pk>/', comment_detail),
]
```

`CommentViewSet`'s `get_queryset()`/`perform_create()` stay identical — `self.kwargs['ticket_pk']` works the same, whether the route comes from a nested router or a manual `path()`, since it's Django that captures `<int:ticket_pk>` in both cases.

**When to choose which:** `drf-nested-routers` if you have multiple nesting levels or many nested resources (the generated code stays consistent). Manual if it's an isolated case and you want to avoid one more dependency.

---

## 3.5 Customizing the pagination output format

By default, `PageNumberPagination` returns `{"count", "next", "previous", "results"}`. If you want a uniform format across your API (for example to match your custom exception handler's format from Chapter 5), override `get_paginated_response()`:

```python
# pagination.py
from rest_framework.pagination import PageNumberPagination
from rest_framework.response import Response


class NotePageNumberPagination(PageNumberPagination):
    page_size = 10
    page_size_query_param = 'page_size'
    max_page_size = 100

    def get_paginated_response(self, data):
        return Response({
            'success': True,
            'pagination': {
                'total': self.page.paginator.count,
                'page': self.page.number,
                'pages': self.page.paginator.num_pages,
                'next': self.get_next_link(),
                'previous': self.get_previous_link(),
            },
            'results': data,
        })
```

`self.page` is Django's `Page` object (`django.core.paginator.Page`), available as soon as `paginate_queryset()` has been called by DRF — it gives you `paginator.count`, `paginator.num_pages`, `number` (the current page number), etc. `get_next_link()`/`get_previous_link()` are methods already provided by `PageNumberPagination` that build the next/previous URLs.

---

## 3.6 Security pitfall — unrestricted `ordering_fields`

A common mistake: **never** leave `ordering_fields = '__all__'` (accepting any field for sorting) without thinking it through.

```python
# DANGEROUS
class NoteViewSet(viewsets.ModelViewSet):
    filter_backends = [OrderingFilter]
    ordering_fields = '__all__'  # any model field becomes sortable
```

The concrete problem: if your model has a sensitive field you don't expose in the serializer (say, an internal credit score, a moderation flag, a hash on a custom model), `'__all__'` still lets you sort by it via `?ordering=internal_score` — and **the relative ordering of results can reveal the field's value** even though it never appears in the JSON response itself. This is an indirect information leak (side-channel), not just a "field not exposed, therefore invisible" situation.

```python
# CORRECT — explicit list, only the fields you're willing to make sortable
class NoteViewSet(viewsets.ModelViewSet):
    filter_backends = [OrderingFilter]
    ordering_fields = ['created_at', 'updated_at', 'title']  # never '__all__'
```

Same caution rule as `fields` on a serializer (Chapter 1, section 1.1): explicit list, never an "accept everything" shortcut in production.

---

## 3.7 Combining pagination + filtering + search + ordering

The three mechanisms (pagination, filtering, ordering) are independent and combine naturally — DRF applies filtering and sorting *before* splitting into pages, never the other way around (otherwise pagination would operate on already-truncated, inconsistent results).

```
Actual order of application for GET /notes/?search=project&is_pinned=true&ordering=-created_at&page=2:

1. get_queryset()           → base queryset (filtered by author=request.user, etc.)
2. filter_backends           → DjangoFilterBackend applies is_pinned=true
                              → SearchFilter applies search=project
                              → OrderingFilter applies the -created_at sort
3. pagination_class          → splits the sorted/filtered result into pages, returns page 2
```

This means the order in which you declare `filter_backends` does **not** affect the final result (each backend filters independently on the queryset already reduced by the previous one) — but pagination always applies last, on the already-filtered and sorted full result.
