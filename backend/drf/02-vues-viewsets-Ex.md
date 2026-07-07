# Exercises — Chapter 2: Views & ViewSets

> File independent from the guide. Solutions folded below — try before looking.

Reference models (Notebook project, same as Chapter 1):

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

Use the `NoteSerializer` and `CategorySerializer` you built in Chapter 1.

---

## Exercise 1 — CategoryListView as plain APIView

Write `CategoryListView(APIView)`:
- `GET` → lists `request.user`'s categories
- `POST` → creates a category for `request.user` (inject `owner` server-side, never accept it from the client)

No router, manual `urls.py` with `path()`.

---

## Exercise 2 — The same thing with GenericAPIView + Mixins

Rewrite Exercise 1 using `ListModelMixin` + `CreateModelMixin` + `GenericAPIView`. Use `get_queryset()` and `perform_create()`.

Compare the code length with Exercise 1.

---

## Exercise 3 — The same thing with a ready-made Generic View

Rewrite it again, this time with `generics.ListCreateAPIView` directly (no manually declared mixins).

---

## Exercise 4 — Full CategoryViewSet

Write `CategoryViewSet(viewsets.ModelViewSet)`:
- `get_queryset()` filters by `request.user`
- `perform_create()` injects `owner`
- Wired up with a `DefaultRouter` in `urls.py`

Verify in a terminal (or Postman/curl) that the 5 standard routes work (`list`, `create`, `retrieve`, `update`, `destroy`).

---

## Exercise 5 — @action detail=True

Add an `archive` action on `NoteViewSet`:
- `POST /api/notes/{pk}/archive/`
- Add an `is_archived` field (`BooleanField`, `default=False`) to the `Note` model (migration required)
- The action sets `is_archived=True` and returns the updated note

---

## Exercise 6 — @action detail=False

Add a `stats` action on `NoteViewSet`:
- `GET /api/notes/stats/`
- Returns simple JSON: `{"total": X, "pinned": Y, "archived": Z}` for `request.user`'s notes

No serializer needed here — build the dict directly and return it via `Response()`.

---

## Exercise 7 — Dynamic get_queryset() with several filters

On `NoteViewSet.get_queryset()`, add support for these combinable query params:
- `?pinned=true` → pinned notes only
- `?category=<id>` → only notes from this category
- `?search=<word>` → notes whose title contains this word (`icontains`)

Test with several combinations in the same request.

---

## Exercise 8 — Dynamic get_serializer_class()

Create `NoteListSerializer` (fields: `id, title, is_pinned` only — very lightweight version) and keep your full `NoteSerializer` for the rest.

On `NoteViewSet.get_serializer_class()`: use `NoteListSerializer` for `list`, `NoteSerializer` for everything else.

Verify that `GET /api/notes/` returns the lightweight version, and `GET /api/notes/{pk}/` returns the full version.

---

## Exercise 9 — perform_destroy() with soft delete

Add an `is_deleted` field (`BooleanField`, `default=False`) to the `Note` model.

Override `perform_destroy()` on `NoteViewSet` so that a `DELETE` doesn't actually remove the row, but sets `is_deleted=True`.

Adapt `get_queryset()` to exclude notes with `is_deleted=True` from every result.

---

## Exercise 10 — Choosing the right approach (design, no code)

For each of these needs, state which approach (APIView / Generic View / ViewSet) you'd choose and why:

1. A `POST /api/auth/login/` endpoint that doesn't correspond to any model
2. A full CRUD on `Category` with 2 extra custom actions (`archive`, `restore`)
3. A `GET /api/notes/export-csv/` endpoint that generates a CSV file, unrelated to the standard notes CRUD
4. A simple read-only list `GET /api/public-notes/` (no create/update/delete, no auth required)

---

## Exercise 11 — Read-only GenericViewSet

Rewrite `CategoryViewSet` (from Exercise 4) to be **read-only** — `list` and `retrieve` only, no create/update/delete — using `ReadOnlyModelViewSet`.

Verify that `POST /api/categories/` correctly returns a 405 (Method Not Allowed).

---

## Exercise 12 — Permission pitfall on a custom action

Here's a deliberately buggy action:

```python
@action(detail=True, methods=['post'])
def restore(self, request, pk=None):
    note = Note.objects.get(pk=pk)  # !!
    note.is_deleted = False
    note.save()
    return Response({'status': 'restored'})
```

1. Explain precisely what the security problem is here.
2. Fix it so it respects permissions like the rest of the ViewSet.

---

## Exercise 13 — Custom url_name + reverse()

Add a `toggle-pin` action on `NoteViewSet`:
- Python method: `def toggle_pin(self, request, pk=None)`
- Desired URL: `/api/notes/{pk}/toggle-pin/` (with a dash, not an underscore)
- `url_name`: `toggle-pin-status`

In the Django shell, use `reverse()` to generate this action's URL for a specific note, and verify it matches the expected pattern.

---

## Exercise 14 — MRO with a custom mixin (design, no code)

You have this mixin:

```python
class LoggingMixin:
    def get_queryset(self):
        print(f"Query run by {self.request.user}")
        return super().get_queryset()
```

Explain what happens (and why) in these two cases:

```python
# Case A
class NoteViewSet(LoggingMixin, viewsets.ModelViewSet):
    def get_queryset(self):
        return Note.objects.filter(author=self.request.user)

# Case B
class NoteViewSet(viewsets.ModelViewSet, LoggingMixin):
    def get_queryset(self):
        return Note.objects.filter(author=self.request.user)
```

Does the `print()` run in either case? Why?

---
---

# Solutions

<details>
<summary>Exercise 1</summary>

```python
class CategoryListView(APIView):
    permission_classes = [IsAuthenticated]

    def get(self, request):
        categories = Category.objects.filter(owner=request.user)
        serializer = CategorySerializer(categories, many=True)
        return Response(serializer.data)

    def post(self, request):
        serializer = CategorySerializer(data=request.data)
        serializer.is_valid(raise_exception=True)
        serializer.save(owner=request.user)
        return Response(serializer.data, status=status.HTTP_201_CREATED)
```

```python
urlpatterns = [
    path('categories/', CategoryListView.as_view()),
]
```
</details>

<details>
<summary>Exercise 2</summary>

```python
class CategoryListView(mixins.ListModelMixin, mixins.CreateModelMixin, generics.GenericAPIView):
    serializer_class = CategorySerializer
    permission_classes = [IsAuthenticated]

    def get_queryset(self):
        return Category.objects.filter(owner=self.request.user)

    def get(self, request, *args, **kwargs):
        return self.list(request, *args, **kwargs)

    def post(self, request, *args, **kwargs):
        return self.create(request, *args, **kwargs)

    def perform_create(self, serializer):
        serializer.save(owner=self.request.user)
```

Noticeably shorter: serialization, error handling, and the 201 status code are handled automatically by the mixins.
</details>

<details>
<summary>Exercise 3</summary>

```python
class CategoryListView(generics.ListCreateAPIView):
    serializer_class = CategorySerializer
    permission_classes = [IsAuthenticated]

    def get_queryset(self):
        return Category.objects.filter(owner=self.request.user)

    def perform_create(self, serializer):
        serializer.save(owner=self.request.user)
```

Even shorter — no need to explicitly declare `get()`/`post()`, nor the mixins.
</details>

<details>
<summary>Exercise 4</summary>

```python
class CategoryViewSet(viewsets.ModelViewSet):
    serializer_class = CategorySerializer
    permission_classes = [IsAuthenticated]

    def get_queryset(self):
        return Category.objects.filter(owner=self.request.user)

    def perform_create(self, serializer):
        serializer.save(owner=self.request.user)
```

```python
router = DefaultRouter()
router.register('categories', CategoryViewSet, basename='category')

urlpatterns = [
    path('api/', include(router.urls)),
]
```
</details>

<details>
<summary>Exercise 5</summary>

```python
class Note(models.Model):
    # ... existing fields ...
    is_archived = models.BooleanField(default=False)
```

```bash
python manage.py makemigrations
python manage.py migrate
```

```python
class NoteViewSet(viewsets.ModelViewSet):
    # ...

    @action(detail=True, methods=['post'])
    def archive(self, request, pk=None):
        note = self.get_object()
        note.is_archived = True
        note.save(update_fields=['is_archived'])
        serializer = self.get_serializer(note)
        return Response(serializer.data)
```
</details>

<details>
<summary>Exercise 6</summary>

```python
class NoteViewSet(viewsets.ModelViewSet):
    # ...

    @action(detail=False, methods=['get'])
    def stats(self, request):
        queryset = self.get_queryset()
        data = {
            'total': queryset.count(),
            'pinned': queryset.filter(is_pinned=True).count(),
            'archived': queryset.filter(is_archived=True).count(),
        }
        return Response(data)
```
</details>

<details>
<summary>Exercise 7</summary>

```python
class NoteViewSet(viewsets.ModelViewSet):
    # ...

    def get_queryset(self):
        queryset = Note.objects.filter(author=self.request.user)

        pinned = self.request.query_params.get('pinned')
        if pinned is not None:
            queryset = queryset.filter(is_pinned=pinned.lower() == 'true')

        category_id = self.request.query_params.get('category')
        if category_id:
            queryset = queryset.filter(category_id=category_id)

        search = self.request.query_params.get('search')
        if search:
            queryset = queryset.filter(title__icontains=search)

        return queryset.select_related('author', 'category')

# Test: GET /api/notes/?pinned=true&search=project
```
</details>

<details>
<summary>Exercise 8</summary>

```python
class NoteListSerializer(serializers.ModelSerializer):
    class Meta:
        model = Note
        fields = ['id', 'title', 'is_pinned']


class NoteViewSet(viewsets.ModelViewSet):
    # ...

    def get_serializer_class(self):
        if self.action == 'list':
            return NoteListSerializer
        return NoteSerializer
```
</details>

<details>
<summary>Exercise 9</summary>

```python
class Note(models.Model):
    # ... existing fields ...
    is_deleted = models.BooleanField(default=False)
```

```python
class NoteViewSet(viewsets.ModelViewSet):
    # ...

    def get_queryset(self):
        return Note.objects.filter(author=self.request.user, is_deleted=False)

    def perform_destroy(self, instance):
        instance.is_deleted = True
        instance.save(update_fields=['is_deleted'])
```

Important: since `get_queryset()` already excludes `is_deleted=True`, a "deleted" note automatically becomes invisible in every other action (`list`, `retrieve`, etc.) with no extra logic.
</details>

<details>
<summary>Exercise 10</summary>

1. **APIView** — no model behind it, very specific authentication logic, no benefit from a ViewSet here.

2. **ViewSet** — full CRUD + custom actions is exactly the intended use case for `@action`. One class, automatic routing.

3. **APIView** (or a simple dedicated view) — this doesn't do CRUD on `Note`, it generates a file. Forcing it into a ViewSet via `@action` would be possible but less clear than a view dedicated to this single responsibility.

4. **Generic View** (`ListAPIView`) — read-only, no custom actions, no need for the full ViewSet machinery. `ListAPIView` with `permission_classes = [AllowAny]` is the most direct solution.
</details>

<details>
<summary>Exercise 11</summary>

```python
class CategoryViewSet(viewsets.ReadOnlyModelViewSet):
    serializer_class = CategorySerializer
    permission_classes = [IsAuthenticated]

    def get_queryset(self):
        return Category.objects.filter(owner=self.request.user)

# ReadOnlyModelViewSet = ListModelMixin + RetrieveModelMixin + GenericViewSet
# No create/update/destroy method exists → the router never maps them
# → POST /api/categories/ automatically returns 405 Method Not Allowed
```
</details>

<details>
<summary>Exercise 12</summary>

**The problem:** `Note.objects.get(pk=pk)` fetches any note in the database, regardless of who owns it. Since this line goes through neither `self.get_object()` nor `self.get_queryset()`, **no permission check runs** — `check_object_permissions()` is never called. Any authenticated user can restore anyone else's note, just by knowing its `pk`.

**Fix:**

```python
@action(detail=True, methods=['post'])
def restore(self, request, pk=None):
    note = self.get_object()  # goes through get_queryset() + check_object_permissions()
    note.is_deleted = False
    note.save()
    return Response({'status': 'restored'})
```

Note: this assumes `get_queryset()` doesn't exclude `is_deleted=True` notes by default, otherwise `get_object()` would return a 404 before even reaching the restore logic — a case to handle separately depending on how `get_queryset()` is written elsewhere in the ViewSet.
</details>

<details>
<summary>Exercise 13</summary>

```python
class NoteViewSet(viewsets.ModelViewSet):
    # ...

    @action(detail=True, methods=['post'], url_path='toggle-pin', url_name='toggle-pin-status')
    def toggle_pin(self, request, pk=None):
        note = self.get_object()
        note.is_pinned = not note.is_pinned
        note.save(update_fields=['is_pinned'])
        return Response({'is_pinned': note.is_pinned})
```

```python
# Shell:
from django.urls import reverse
note = Note.objects.first()
url = reverse('note-toggle-pin-status', args=[note.pk])
print(url)  # /api/notes/{pk}/toggle-pin/
```

The name used in `reverse()` follows `{basename}-{url_name}` — here `note` (basename set in `router.register('notes', NoteViewSet, basename='note')`) + `toggle-pin-status`.
</details>

<details>
<summary>Exercise 14</summary>

**Case A** — the `print()` runs... or does it? `LoggingMixin` is to the left of `viewsets.ModelViewSet` in the MRO. But note: here `NoteViewSet` defines `get_queryset()` **itself**, so `NoteViewSet`'s own method runs first — not `LoggingMixin`'s. For the mixin's `print()` to run, `NoteViewSet.get_queryset()` would need to call `super().get_queryset()` instead of returning its own queryset directly. As written, in Case A **the print never runs**, because `NoteViewSet.get_queryset()` completely short-circuits the chain — the mixin is present in the inheritance list but its method is never called.

**Case B** — same outcome: `NoteViewSet.get_queryset()` defines its own implementation with no `super().get_queryset()` call. The order of the parent classes (`LoggingMixin` on the right here) doesn't even matter in this specific case, since the class's own method always takes precedence over its parents, regardless of their order.

**Takeaway:** the MRO only determines resolution order **among parent classes** — not between a class and its own method defined directly on it. If `NoteViewSet` hadn't defined `get_queryset()` itself, then the order of the mixins in the inheritance list would have determined which parent provides `get_queryset()`.
</details>
