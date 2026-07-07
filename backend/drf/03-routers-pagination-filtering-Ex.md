# Exercises — Chapter 3: Routers, Pagination, Filtering

> File independent from the guide. Solutions folded below.

Reference models (Notebook project):

```python
class Category(models.Model):
    name = models.CharField(max_length=100)
    owner = models.ForeignKey(User, on_delete=models.CASCADE)

class Note(models.Model):
    title = models.CharField(max_length=200)
    body = models.TextField()
    category = models.ForeignKey(Category, on_delete=models.SET_NULL, null=True, blank=True, related_name='notes')
    author = models.ForeignKey(User, on_delete=models.CASCADE, related_name='notes')
    is_pinned = models.BooleanField(default=False)
    created_at = models.DateTimeField(auto_now_add=True)
    updated_at = models.DateTimeField(auto_now=True)
```

---

## Exercise 1 — Basic router

Wire up `NoteViewSet` and `CategoryViewSet` (already built in Chapter 2) with a `DefaultRouter`. Verify in the browser that the root `/api/` route lists both resources.

---

## Exercise 2 — Custom PageNumber pagination

Create `NotePageNumberPagination` with `page_size = 5`, `page_size_query_param = 'page_size'`, `max_page_size = 50`.

Wire it up on `NoteViewSet` only (not globally in `settings.py`).

Test: create 12 notes, verify `GET /api/notes/` returns only 5, `?page=2` returns the next 5, and `?page_size=20` (above the max) is capped at 50.

---

## Exercise 3 — Switching pagination strategy

Redo Exercise 2 but with `LimitOffsetPagination`. Compare the query params used (`?page=` vs `?limit=&offset=`) and the JSON response structure.

---

## Exercise 4 — SearchFilter + OrderingFilter

On `NoteViewSet`, add:
- `SearchFilter` on `title` and `body`
- `OrderingFilter` on `created_at`, `title`, with `created_at` descending by default

Test `?search=project`, `?ordering=title`, and both combined.

---

## Exercise 5 — Custom FilterSet

Create `NoteFilter(django_filters.FilterSet)` with:
- `is_pinned` (exact)
- `category` (exact, by ID)
- `created_after` / `created_before` (date range on `created_at`)

Wire it up on `NoteViewSet` alongside `SearchFilter`/`OrderingFilter`. Test a request combining all three backends at once.

---

## Exercise 6 — Disabling pagination on a custom action

On `NoteViewSet`'s `pinned` action (Chapter 2), disable pagination locally — even though `NoteViewSet` has global pagination configured for `list`.

---

## Exercise 7 — Nested routes

Add a `Comment` model:

```python
class Comment(models.Model):
    note = models.ForeignKey(Note, on_delete=models.CASCADE, related_name='comments')
    author = models.ForeignKey(User, on_delete=models.CASCADE)
    body = models.TextField()
    created_at = models.DateTimeField(auto_now_add=True)
```

Set up `/api/notes/{note_pk}/comments/` with `drf-nested-routers` (or the manual method, your choice). `CommentViewSet.get_queryset()` must filter by `note_id=self.kwargs['note_pk']`, and `perform_create()` must inject `author` and `note_id` automatically.

---

## Exercise 8 — Custom pagination format

Override `get_paginated_response()` on `NotePageNumberPagination` to return this format:

```json
{
    "success": true,
    "pagination": {"total": 12, "page": 1, "pages": 3},
    "results": [...]
}
```

---

## Exercise 9 — Spotting the security pitfall (design, no code)

This code is in production:

```python
class NoteViewSet(viewsets.ModelViewSet):
    filter_backends = [OrderingFilter]
    ordering_fields = '__all__'
```

The `Note` model has a hidden `internal_priority_score` field (never in the serializer). Explain concretely how an attacker could infer values of this field without ever seeing it in a JSON response, then fix the code.

---
---

# Solutions

<details>
<summary>Exercise 1</summary>

```python
from rest_framework.routers import DefaultRouter
from .views import NoteViewSet, CategoryViewSet

router = DefaultRouter()
router.register('notes', NoteViewSet, basename='note')
router.register('categories', CategoryViewSet, basename='category')

urlpatterns = [
    path('api/', include(router.urls)),
]
# Visit /api/ in the browser → DefaultRouter lists 'notes' and 'categories'
```
</details>

<details>
<summary>Exercise 2</summary>

```python
from rest_framework.pagination import PageNumberPagination

class NotePageNumberPagination(PageNumberPagination):
    page_size = 5
    page_size_query_param = 'page_size'
    max_page_size = 50

class NoteViewSet(viewsets.ModelViewSet):
    pagination_class = NotePageNumberPagination
    # ...

# Tests:
# GET /api/notes/                 → 5 results, "next" points to page=2
# GET /api/notes/?page=2          → the next 5
# GET /api/notes/?page_size=20    → capped at 50 (but with only 12 notes here, returns 12)
```
</details>

<details>
<summary>Exercise 3</summary>

```python
class NoteLimitOffsetPagination(LimitOffsetPagination):
    default_limit = 5
    max_limit = 50

class NoteViewSet(viewsets.ModelViewSet):
    pagination_class = NoteLimitOffsetPagination
```

Observed differences:
- `PageNumberPagination`: `?page=2` — response `{"count", "next", "previous", "results"}`
- `LimitOffsetPagination`: `?limit=5&offset=5` — same response structure, but navigation by numeric offset rather than page number. More flexible (`offset=7` is possible, not just whole pages), but less intuitive for a "page 1, 2, 3" frontend.
</details>

<details>
<summary>Exercise 4</summary>

```python
class NoteViewSet(viewsets.ModelViewSet):
    filter_backends = [SearchFilter, OrderingFilter]
    search_fields = ['title', 'body']
    ordering_fields = ['created_at', 'title']
    ordering = ['-created_at']

# Tests:
# GET /api/notes/?search=project
# GET /api/notes/?ordering=title
# GET /api/notes/?search=project&ordering=title
```
</details>

<details>
<summary>Exercise 5</summary>

```python
import django_filters
from .models import Note

class NoteFilter(django_filters.FilterSet):
    created_after = django_filters.DateTimeFilter(field_name='created_at', lookup_expr='gte')
    created_before = django_filters.DateTimeFilter(field_name='created_at', lookup_expr='lte')

    class Meta:
        model = Note
        fields = ['is_pinned', 'category', 'created_after', 'created_before']

class NoteViewSet(viewsets.ModelViewSet):
    filter_backends = [DjangoFilterBackend, SearchFilter, OrderingFilter]
    filterset_class = NoteFilter
    search_fields = ['title', 'body']
    ordering_fields = ['created_at', 'title']

# Combined test:
# GET /api/notes/?is_pinned=true&search=project&ordering=-created_at
```
</details>

<details>
<summary>Exercise 6</summary>

```python
class NoteViewSet(viewsets.ModelViewSet):
    pagination_class = NotePageNumberPagination  # active for list, retrieve, etc.

    @action(detail=False, methods=['get'])
    def pinned(self, request):
        self.pagination_class = None  # disabled only for this action
        notes = self.get_queryset().filter(is_pinned=True)
        serializer = self.get_serializer(notes, many=True)
        return Response(serializer.data)
```
</details>

<details>
<summary>Exercise 7</summary>

```python
class Comment(models.Model):
    note = models.ForeignKey(Note, on_delete=models.CASCADE, related_name='comments')
    author = models.ForeignKey(User, on_delete=models.CASCADE)
    body = models.TextField()
    created_at = models.DateTimeField(auto_now_add=True)

class CommentViewSet(viewsets.ModelViewSet):
    serializer_class = CommentSerializer
    permission_classes = [IsAuthenticated]

    def get_queryset(self):
        return Comment.objects.filter(note_id=self.kwargs['note_pk'])

    def perform_create(self, serializer):
        serializer.save(author=self.request.user, note_id=self.kwargs['note_pk'])

# urls.py — with drf-nested-routers
from rest_framework_nested import routers

router = routers.DefaultRouter()
router.register('notes', NoteViewSet, basename='note')

comments_router = routers.NestedDefaultRouter(router, 'notes', lookup='note')
comments_router.register('comments', CommentViewSet, basename='note-comments')

urlpatterns = [
    path('api/', include(router.urls)),
    path('api/', include(comments_router.urls)),
]
```
</details>

<details>
<summary>Exercise 8</summary>

```python
class NotePageNumberPagination(PageNumberPagination):
    page_size = 5

    def get_paginated_response(self, data):
        return Response({
            'success': True,
            'pagination': {
                'total': self.page.paginator.count,
                'page': self.page.number,
                'pages': self.page.paginator.num_pages,
            },
            'results': data,
        })
```
</details>

<details>
<summary>Exercise 9</summary>

**How the attacker exploits it:** `ordering_fields = '__all__'` accepts `?ordering=internal_priority_score` even though this field never appears in `NoteSerializer.fields`. By observing the **relative order** in which notes come back (e.g. sorting ascending vs descending, and comparing the positions of notes whose identity is otherwise known), an attacker can infer information about each note's `internal_priority_score` — without ever reading it directly in the JSON. It's slower than a direct leak, but it's still an exploitable information leak, especially if the attacker can make many requests and cross-reference results.

**Fix:**

```python
class NoteViewSet(viewsets.ModelViewSet):
    filter_backends = [OrderingFilter]
    ordering_fields = ['created_at', 'updated_at', 'title']  # explicit, never '__all__'
```
</details>
