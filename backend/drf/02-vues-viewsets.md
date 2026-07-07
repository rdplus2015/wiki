# Chapter 2 — Views & ViewSets

## 2.0 Why this hierarchy exists

DRF designed its views in increasing layers of abstraction. Each layer solves the repetition of the previous one. Understanding each layer lets you pick the right level and know where to step in when you need custom behavior.

```
APIView                 ← full control, everything by hand
  └─ GenericAPIView     ← queryset + serializer_class provided
       └─ Mixins        ← reusable CRUD logic
            └─ Generic Views (ListAPIView, etc.)  ← ready-made combinations
                 └─ ViewSet / ModelViewSet         ← full CRUD + automatic routing
```

---

## 2.1 APIView — the base

`APIView` handles content negotiation, authentication, and permissions, but nothing else. You implement each HTTP method by hand.

```python
from rest_framework.views import APIView
from rest_framework.response import Response
from rest_framework import status
from .models import Note
from .serializers import NoteSerializer


class NoteListView(APIView):
    """
    GET /api/notes/     → lists all notes for the logged-in user
    POST /api/notes/    → creates a new note
    """

    def get(self, request):
        # request.user is available because APIView handles auth automatically
        notes = Note.objects.filter(author=request.user)

        # many=True tells the serializer it must serialize a queryset
        serializer = NoteSerializer(notes, many=True, context={'request': request})

        # Response automatically handles JSON serialization and Content-Type
        return Response(serializer.data)

    def post(self, request):
        # request.data contains the parsed request body (JSON → dict)
        serializer = NoteSerializer(data=request.data, context={'request': request})

        if serializer.is_valid():
            # save() calls the serializer's create()
            # We inject author=request.user, which isn't in the request body
            serializer.save(author=request.user)
            return Response(serializer.data, status=status.HTTP_201_CREATED)

        return Response(serializer.errors, status=status.HTTP_400_BAD_REQUEST)


class NoteDetailView(APIView):
    """
    GET /api/notes/{pk}/    → note detail
    PUT /api/notes/{pk}/    → full update
    PATCH /api/notes/{pk}/  → partial update
    DELETE /api/notes/{pk}/ → delete
    """

    def get_object(self, pk, user):
        """Helper: fetches the note or raises a 404."""
        try:
            return Note.objects.get(pk=pk, author=user)
        except Note.DoesNotExist:
            from django.http import Http404
            raise Http404

    def get(self, request, pk):
        note = self.get_object(pk, request.user)
        serializer = NoteSerializer(note, context={'request': request})
        return Response(serializer.data)

    def put(self, request, pk):
        note = self.get_object(pk, request.user)
        # Passing the existing instance: the serializer will call update() instead of create()
        serializer = NoteSerializer(note, data=request.data, context={'request': request})
        if serializer.is_valid():
            serializer.save()
            return Response(serializer.data)
        return Response(serializer.errors, status=status.HTTP_400_BAD_REQUEST)

    def patch(self, request, pk):
        note = self.get_object(pk, request.user)
        # partial=True: fields not sent keep their current value
        serializer = NoteSerializer(
            note, data=request.data, partial=True, context={'request': request}
        )
        if serializer.is_valid():
            serializer.save()
            return Response(serializer.data)
        return Response(serializer.errors, status=status.HTTP_400_BAD_REQUEST)

    def delete(self, request, pk):
        note = self.get_object(pk, request.user)
        note.delete()
        # 204 No Content: success with no response body
        return Response(status=status.HTTP_204_NO_CONTENT)
```

**Problem:** `NoteListView` and `NoteDetailView` contain a lot of boilerplate you'd rewrite identically for `Category`, `Tag`, etc. That's where `GenericAPIView` and mixins come in.

**When to still use APIView:** custom endpoints (login, health-check, special business action), logic that doesn't fit a standard CRUD, full control needed.

---

## 2.2 GenericAPIView + Mixins

`GenericAPIView` extends `APIView` by adding `queryset` and `serializer_class` handling, plus utility methods (`get_queryset()`, `get_serializer()`, `get_object()`).

Mixins add reusable CRUD behaviors:
- `ListModelMixin` → `list()` method
- `CreateModelMixin` → `create()` method
- `RetrieveModelMixin` → `retrieve()` method
- `UpdateModelMixin` → `update()` method
- `DestroyModelMixin` → `destroy()` method

```python
from rest_framework import generics, mixins
from rest_framework.permissions import IsAuthenticated


class NoteListView(
    mixins.ListModelMixin,
    mixins.CreateModelMixin,
    generics.GenericAPIView
):
    serializer_class = NoteSerializer
    permission_classes = [IsAuthenticated]

    def get_queryset(self):
        # get_queryset() is the method to override to customize the queryset
        return Note.objects.filter(author=self.request.user)

    def get(self, request, *args, **kwargs):
        # self.list() is provided by ListModelMixin: calls get_queryset(),
        # serializes, handles pagination automatically
        return self.list(request, *args, **kwargs)

    def post(self, request, *args, **kwargs):
        return self.create(request, *args, **kwargs)

    def perform_create(self, serializer):
        # perform_create is CreateModelMixin's hook for injecting data
        serializer.save(author=self.request.user)
```

### Ready-made generic views

DRF provides ready-made combinations that avoid manually declaring mixins:

```python
# Equivalent of List + Create, without declaring the mixins
class NoteListCreateView(generics.ListCreateAPIView):
    serializer_class = NoteSerializer
    permission_classes = [IsAuthenticated]

    def get_queryset(self):
        return Note.objects.filter(author=self.request.user)

    def perform_create(self, serializer):
        serializer.save(author=self.request.user)


# Retrieve + Update + Delete
class NoteDetailView(generics.RetrieveUpdateDestroyAPIView):
    serializer_class = NoteSerializer
    permission_classes = [IsAuthenticated]

    def get_queryset(self):
        return Note.objects.filter(author=self.request.user)
    # get_object() automatically uses get_queryset() + the pk from the URL
```

Available generic views, by use case:

```
ListAPIView                  → GET list
CreateAPIView                → POST
ListCreateAPIView            → GET list + POST

RetrieveAPIView              → GET detail
UpdateAPIView                → PUT + PATCH
DestroyAPIView                → DELETE
RetrieveUpdateAPIView        → GET + PUT + PATCH
RetrieveUpdateDestroyAPIView → GET + PUT + PATCH + DELETE
```

### Useful hooks on GenericAPIView

```python
class ArticleDetailView(generics.RetrieveUpdateDestroyAPIView):
    queryset = Article.objects.all()
    serializer_class = ArticleSerializer

    def get_queryset(self):
        # filter dynamically based on the user or query params
        return Article.objects.filter(author=self.request.user)

    def get_serializer_class(self):
        # different serializer depending on the HTTP method
        if self.request.method == 'GET':
            return ArticleReadSerializer
        return ArticleWriteSerializer

    def perform_update(self, serializer):
        serializer.save(updated_by=self.request.user)

    def perform_destroy(self, instance):
        # logic before deletion
        instance.delete()
```

**When to use Generic Views:** standard CRUD on a model, little custom logic. The most concise approach for simple resources, with no custom actions.

**Pros:** fast to write, clean, less code, pro standard.

---

## 2.3 ViewSet and ModelViewSet — maximum abstraction

A `ViewSet` groups all of a resource's actions into a single class, and lets the router generate URLs automatically.

```python
from rest_framework import viewsets
from rest_framework.permissions import IsAuthenticated


class NoteViewSet(viewsets.ModelViewSet):
    """
    ModelViewSet automatically generates:
    list()     → GET /notes/
    create()   → POST /notes/
    retrieve() → GET /notes/{pk}/
    update()   → PUT /notes/{pk}/
    partial_update() → PATCH /notes/{pk}/
    destroy()  → DELETE /notes/{pk}/
    """
    serializer_class = NoteSerializer
    permission_classes = [IsAuthenticated]

    def get_queryset(self):
        return Note.objects.filter(author=self.request.user)

    def perform_create(self, serializer):
        serializer.save(author=self.request.user)
```

```python
# urls.py — the router generates all URLs
from rest_framework.routers import DefaultRouter
from .views import NoteViewSet

router = DefaultRouter()
router.register('notes', NoteViewSet, basename='note')

urlpatterns = router.urls
# Automatically generates:
# /notes/          GET, POST
# /notes/{pk}/     GET, PUT, PATCH, DELETE
```

**When to use ViewSet:** as soon as you have several actions on one resource, custom endpoints, or several resources in the same project. This is the standard approach in production.

**Pros:** ultra fast, pro standard, clean, scalable
**Cons:** less explicit for a beginner, a lot of logic is implicit

---

## 2.4 @action — adding custom endpoints to a ViewSet

`@action` lets you add endpoints that don't fit the standard CRUD.

```python
from rest_framework.decorators import action
from rest_framework.response import Response
from rest_framework import status


class NoteViewSet(viewsets.ModelViewSet):
    serializer_class = NoteSerializer
    permission_classes = [IsAuthenticated]

    def get_queryset(self):
        return Note.objects.filter(author=self.request.user)

    def perform_create(self, serializer):
        serializer.save(author=self.request.user)

    @action(detail=True, methods=['post'])
    def pin(self, request, pk=None):
        """
        POST /api/notes/{pk}/pin/

        detail=True  → the action applies to a specific instance (/{pk}/pin/)
        detail=False → the action applies to the collection (/pin/)
        methods      → list of accepted HTTP methods
        """
        note = self.get_object()  # Fetches the note and checks permissions
        note.is_pinned = True
        note.save()
        return Response({'status': 'note pinned'})

    @action(detail=True, methods=['post'])
    def unpin(self, request, pk=None):
        """POST /api/notes/{pk}/unpin/"""
        note = self.get_object()
        note.is_pinned = False
        note.save()
        return Response({'status': 'note unpinned'})

    @action(detail=False, methods=['get'])
    def pinned(self, request):
        """
        GET /api/notes/pinned/

        detail=False → no {pk} in the URL, operates on the collection
        """
        pinned_notes = self.get_queryset().filter(is_pinned=True)

        # get_serializer() is provided by GenericAPIView
        # It automatically passes the context (request, etc.)
        serializer = self.get_serializer(pinned_notes, many=True)
        return Response(serializer.data)

    @action(detail=False, methods=['get'], url_path='by-category/(?P<category_id>[^/.]+)')
    def by_category(self, request, category_id=None):
        """
        GET /api/notes/by-category/{category_id}/

        url_path lets you define a custom URL pattern with parameters
        """
        notes = self.get_queryset().filter(category_id=category_id)
        serializer = self.get_serializer(notes, many=True)
        return Response(serializer.data)
```

`@action` can also receive its own `permission_classes`, independent from the class's:

```python
@action(detail=True, methods=['post'], permission_classes=[IsAuthenticated])
def publish(self, request, pk=None):
    article = self.get_object()
    if article.published:
        return Response({'error': 'Already published.'}, status=400)
    article.published = True
    article.save(update_fields=['published'])
    return Response({'status': 'Article published.'})
```

---

## 2.5 Dynamic get_queryset() and get_serializer_class()

These two methods are the main extension points for ViewSets and GenericViews.

### Dynamic get_queryset()

```python
class NoteViewSet(viewsets.ModelViewSet):
    permission_classes = [IsAuthenticated]

    def get_queryset(self):
        """
        Builds the queryset based on context.
        self.action holds the current action: 'list', 'create', 'retrieve', etc.
        self.request holds the request with user, query_params, etc.
        """
        queryset = Note.objects.filter(author=self.request.user)

        # Optional filter via query param: GET /notes/?pinned=true
        pinned = self.request.query_params.get('pinned')
        if pinned is not None:
            queryset = queryset.filter(is_pinned=pinned.lower() == 'true')

        category_id = self.request.query_params.get('category')
        if category_id:
            queryset = queryset.filter(category_id=category_id)

        # N+1 optimization: preload the relations used in the serializer
        return queryset.select_related('author', 'category')
```

### Dynamic get_serializer_class()

Very useful for a different serializer on read vs write (advanced pattern, widely used in production).

```python
class NoteListSerializer(serializers.ModelSerializer):
    """Lightweight serializer for the list: fewer fields, less data."""
    preview = serializers.SerializerMethodField()

    def get_preview(self, obj):
        return obj.body[:80] + '...' if len(obj.body) > 80 else obj.body

    class Meta:
        model = Note
        fields = ['id', 'title', 'preview', 'is_pinned', 'created_at']


class NoteDetailSerializer(serializers.ModelSerializer):
    """Full serializer for detail: all fields."""
    author_username = serializers.CharField(source='author.username', read_only=True)
    category = CategorySerializer(read_only=True)
    category_id = serializers.PrimaryKeyRelatedField(
        queryset=Category.objects.all(), source='category',
        write_only=True, required=False, allow_null=True
    )

    class Meta:
        model = Note
        fields = ['id', 'title', 'body', 'author_username',
                  'category', 'category_id', 'is_pinned', 'created_at', 'updated_at']


class NoteViewSet(viewsets.ModelViewSet):
    permission_classes = [IsAuthenticated]

    def get_queryset(self):
        return Note.objects.filter(author=self.request.user).select_related('author', 'category')

    def get_serializer_class(self):
        """
        self.action: 'list', 'create', 'retrieve', 'update', 'partial_update', 'destroy'
        + the names of your custom @action methods
        """
        if self.action == 'list':
            return NoteListSerializer
        return NoteDetailSerializer

    def perform_create(self, serializer):
        serializer.save(author=self.request.user)
```

---

## 2.6 perform_create / perform_update / perform_destroy

These methods are hooks from `CreateModelMixin`, `UpdateModelMixin`, `DestroyModelMixin`. They let you inject business logic without fully overriding the method.

```python
class NoteViewSet(viewsets.ModelViewSet):
    permission_classes = [IsAuthenticated]

    def get_queryset(self):
        return Note.objects.filter(author=self.request.user)

    def perform_create(self, serializer):
        # Injects author from the request context
        # The user can't forge the author in the request body
        serializer.save(author=self.request.user)

    def perform_update(self, serializer):
        # Hook for business logic on update
        serializer.save(updated_by=self.request.user)

    def perform_destroy(self, instance):
        # Instead of deleting, we could do a soft delete
        # instance.is_deleted = True
        # instance.save()
        instance.delete()  # Default behavior
```

---

## 2.7 Comparing the 3 approaches on the same example

The same `Article` endpoint (full CRUD) implemented with each approach, to clearly see the difference in verbosity.

### Approach 1 — APIView

```python
class ArticleListView(APIView):
    def get(self, request):
        articles = Article.objects.all()
        serializer = ArticleSerializer(articles, many=True)
        return Response(serializer.data)

    def post(self, request):
        serializer = ArticleSerializer(data=request.data)
        if serializer.is_valid():
            serializer.save(author=request.user)
            return Response(serializer.data, status=status.HTTP_201_CREATED)
        return Response(serializer.errors, status=status.HTTP_400_BAD_REQUEST)


class ArticleDetailView(APIView):
    def get_object(self, pk):
        return get_object_or_404(Article, pk=pk)

    def get(self, request, pk):
        serializer = ArticleSerializer(self.get_object(pk))
        return Response(serializer.data)

    def put(self, request, pk):
        serializer = ArticleSerializer(self.get_object(pk), data=request.data)
        if serializer.is_valid():
            serializer.save()
            return Response(serializer.data)
        return Response(serializer.errors, status=400)

    def delete(self, request, pk):
        self.get_object(pk).delete()
        return Response(status=status.HTTP_204_NO_CONTENT)
```

```python
# urls.py
urlpatterns = [
    path('articles/', ArticleListView.as_view()),
    path('articles/<int:pk>/', ArticleDetailView.as_view()),
]
```

### Approach 2 — Generic Views

```python
class ArticleListCreateView(generics.ListCreateAPIView):
    queryset = Article.objects.select_related('author')
    serializer_class = ArticleSerializer
    permission_classes = [IsAuthenticatedOrReadOnly]

    def perform_create(self, serializer):
        serializer.save(author=self.request.user)


class ArticleDetailView(generics.RetrieveUpdateDestroyAPIView):
    queryset = Article.objects.select_related('author')
    serializer_class = ArticleSerializer
    permission_classes = [IsAuthenticatedOrReadOnly]
```

```python
# urls.py — identical to APIView
urlpatterns = [
    path('articles/', ArticleListCreateView.as_view()),
    path('articles/<int:pk>/', ArticleDetailView.as_view()),
]
```

### Approach 3 — ViewSet + Router

```python
class ArticleViewSet(viewsets.ModelViewSet):
    queryset = Article.objects.select_related('author').prefetch_related('tags')
    serializer_class = ArticleSerializer
    permission_classes = [IsAuthenticatedOrReadOnly]

    def get_queryset(self):
        qs = super().get_queryset()
        author = self.request.query_params.get('author')
        if author:
            qs = qs.filter(author__username=author)
        return qs

    def get_serializer_class(self):
        if self.action == 'list':
            return ArticleListSerializer
        return ArticleDetailSerializer

    def perform_create(self, serializer):
        serializer.save(author=self.request.user)

    @action(detail=True, methods=['post'], permission_classes=[IsAuthenticated])
    def publish(self, request, pk=None):
        article = self.get_object()
        if article.published:
            return Response({'error': 'Already published.'}, status=400)
        article.published = True
        article.save(update_fields=['published'])
        return Response({'status': 'Article published.'})

    @action(detail=False, methods=['get'])
    def trending(self, request):
        qs = Article.objects.filter(published=True).order_by('-views')[:10]
        serializer = self.get_serializer(qs, many=True)
        return Response(serializer.data)
```

```python
# urls.py
router = DefaultRouter()
router.register('articles', ArticleViewSet, basename='article')

urlpatterns = [
    path('api/', include(router.urls)),
]
```

What the router automatically generates:

```
GET    /api/articles/                → list
POST   /api/articles/                → create
GET    /api/articles/{pk}/           → retrieve
PUT    /api/articles/{pk}/           → update
PATCH  /api/articles/{pk}/           → partial_update
DELETE /api/articles/{pk}/           → destroy
POST   /api/articles/{pk}/publish/   → publish  (@action)
GET    /api/articles/trending/       → trending (@action)
GET    /api/                         → root listing all routes (DefaultRouter only)
```

### Summary table

|                 | APIView      | Generic Views    | ViewSet + Router     |
|---|---|---|---|
| Code            | Verbose      | Concise          | Very concise           |
| URLs            | Manual       | Manual           | Automatic               |
| Flexibility     | Full         | Medium           | Good (+ `@action`)      |
| Custom actions  | Natural      | Not built-in     | `@action`                |
| Multi-actions   | 2 classes    | 2 classes        | 1 class                  |
| Typical use     | Auth, custom | Simple CRUD      | Full CRUD, production    |

*(see also Chapter 0 for the condensed version of this table)*

---

## 2.9 Deep dive — ViewSets

### ViewSet vs GenericViewSet vs ModelViewSet

There are actually 3 levels of ViewSet, not just "ModelViewSet does everything". Understanding the distinction helps you pick the right level when `ModelViewSet` does too much for your case.

```
ViewSet          ← like APIView, but with action names (list, create...)
                    instead of HTTP methods. Nothing automatic, everything by hand.
  └─ GenericViewSet   ← ViewSet + the same utilities as GenericAPIView
                        (get_queryset, get_serializer, get_object) but WITHOUT the CRUD mixins
       └─ ModelViewSet ← GenericViewSet + all CRUD mixins already wired in
```

```python
# Plain ViewSet — you write every action yourself, but keep automatic routing
class NoteViewSet(viewsets.ViewSet):
    permission_classes = [IsAuthenticated]

    def list(self, request):
        notes = Note.objects.filter(author=request.user)
        serializer = NoteSerializer(notes, many=True)
        return Response(serializer.data)

    def create(self, request):
        serializer = NoteSerializer(data=request.data)
        serializer.is_valid(raise_exception=True)
        serializer.save(author=request.user)
        return Response(serializer.data, status=status.HTTP_201_CREATED)

    def retrieve(self, request, pk=None):
        note = get_object_or_404(Note, pk=pk, author=request.user)
        serializer = NoteSerializer(note)
        return Response(serializer.data)
    # etc. — no get_queryset()/get_object() provided, everything is manual
```

```python
# GenericViewSet — you keep get_queryset()/get_object()/get_serializer(),
# but with NO automatic CRUD behavior. You choose your own mixins.
class NoteViewSet(mixins.ListModelMixin, mixins.RetrieveModelMixin, viewsets.GenericViewSet):
    """Read-only (list + retrieve), no create/update/delete — without hand-writing a ReadOnlyModelViewSet."""
    serializer_class = NoteSerializer
    permission_classes = [IsAuthenticated]

    def get_queryset(self):
        return Note.objects.filter(author=self.request.user)
```

**When to use which:**
- `ModelViewSet`: the standard case, full CRUD. **90% of the time.**
- `GenericViewSet` + chosen mixins: you want partial CRUD (e.g. read-only → use `ReadOnlyModelViewSet` directly, a shortcut for `ListModelMixin + RetrieveModelMixin + GenericViewSet`) or a specific subset of actions.
- Plain `ViewSet`: very rare — logic so custom that even `get_queryset()`/`get_object()` are useless. At that point, ask yourself if `APIView` wouldn't be more honest.

### How DRF routes actions internally

The key thing to understand: a `ViewSet` is **not** a classic Django view — it has no fixed `as_view()` that directly maps HTTP method → Python method. It's the **router** that does this mapping, at `router.register()` time.

```
The router turns each (URL, HTTP method) combination into a call to a named action:

GET    /notes/          →  action 'list'
POST   /notes/          →  action 'create'
GET    /notes/{pk}/     →  action 'retrieve'
PUT    /notes/{pk}/     →  action 'update'
PATCH  /notes/{pk}/     →  action 'partial_update'
DELETE /notes/{pk}/     →  action 'destroy'
```

Concretely, `router.register('notes', NoteViewSet)` internally generates something like:

```python
# Simplified — what the router does for you
NoteViewSet.as_view({'get': 'list', 'post': 'create'})       # for /notes/
NoteViewSet.as_view({'get': 'retrieve', 'put': 'update',
                      'patch': 'partial_update', 'delete': 'destroy'})  # for /notes/{pk}/
```

This is exactly why `self.action` exists in `get_queryset()`/`get_serializer_class()`: by the time your method runs, DRF already knows which "named action" was called — whether it's a standard action (`list`) or one of your custom `@action` methods (`pin`, `archive`...).

### Pitfalls with custom mixins and inheritance order (MRO)

When you combine several mixins yourself (like in the `GenericViewSet` example above), **order matters**. Python resolves methods left to right (MRO — Method Resolution Order).

```python
# CORRECT: mixins before GenericViewSet
class NoteViewSet(mixins.ListModelMixin, mixins.CreateModelMixin, viewsets.GenericViewSet):
    ...

# If you reverse the order, or mix in a custom mixin that overrides
# get_queryset(), the LEFTMOST mixin wins on conflict.
```

This pitfall also comes up with a **custom** mixin (e.g. a `TenantScopedMixin` that filters by tenant — see Chapter 7): it must be placed **before** `ModelViewSet` in the inheritance list so its `get_queryset()` is the one that actually runs first (via `super()`).

```python
# The custom mixin BEFORE ModelViewSet — its get_queryset() takes precedence
class NoteViewSet(TenantScopedMixin, viewsets.ModelViewSet):
    ...
```

### get_object() and check_object_permissions() on custom actions

A common pitfall: on a `detail=True` action, you're responsible for calling `self.get_object()` (not a direct model query) — that's what automatically triggers the `has_object_permission()` check from your permission classes.

```python
@action(detail=True, methods=['post'])
def archive(self, request, pk=None):
    # GOOD: goes through get_object(), which internally calls check_object_permissions()
    note = self.get_object()
    note.is_archived = True
    note.save()
    return Response({'status': 'archived'})

@action(detail=True, methods=['post'])
def archive_buggy(self, request, pk=None):
    # BAD: completely bypasses the object permission check
    note = Note.objects.get(pk=pk)  # any authenticated user can archive any note
    note.is_archived = True
    note.save()
    return Response({'status': 'archived'})
```

If you fetch an object some other way in a custom action, you must call `self.check_object_permissions(request, obj)` yourself — otherwise `has_object_permission` never fires.

### Per-action throttle_classes

Like permissions, throttling can be overridden per `@action`:

```python
class NoteViewSet(viewsets.ModelViewSet):
    throttle_classes = [UserRateThrottle]  # default limit on all actions

    @action(detail=False, methods=['post'], throttle_classes=[AnonRateThrottle])
    def bulk_import(self, request):
        # This action has its own limit, different from the rest of the ViewSet
        ...
```

### Custom url_name and url_path for reverse()

By default, the route name generated by the router for an action follows the `{basename}-{method_name}` pattern. You can customize it:

```python
@action(detail=True, methods=['post'], url_path='mark-as-read', url_name='mark-read')
def mark_read(self, request, pk=None):
    ...

# reverse() uses url_name, not the Python method name:
# reverse('note-mark-read', args=[note.pk])  →  /notes/{pk}/mark-as-read/
```

Useful when the Python method name (constrained by Python naming rules) needs to differ from the name you want in the URL or in `reverse()`.

---

## 2.10 Where to put different serializers for APIView / Generic Views

Unlike ViewSets where `get_serializer_class()` centralizes this, on a plain `GenericAPIView` you can also vary the serializer by HTTP method:

```python
class ArticleDetailView(generics.RetrieveUpdateDestroyAPIView):
    queryset = Article.objects.all()

    def get_serializer_class(self):
        if self.request.method == 'GET':
            return ArticleReadSerializer
        return ArticleWriteSerializer
```

Same principle as ViewSets (section 2.5), just based on `self.request.method` rather than `self.action` — because a plain `GenericAPIView` has no notion of a named `action`, only an HTTP method.
