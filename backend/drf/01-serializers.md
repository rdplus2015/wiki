# Chapter 1 — Serializers

## 1.0 The problem it solves

Django works with Python objects (model instances, querysets). HTTP speaks JSON (raw text). You need a bidirectional translator between these two worlds.

```
JSON (bytes)              ──→  Serializer.is_valid()  ──→  Python object / Django instance
Python object / QuerySet  ──→  Serializer.data         ──→  JSON (serializable dict)
```

Without a serializer, you'd do this by hand in every view:

```python
# Without DRF — what you avoid
import json
from django.http import JsonResponse

def article_list(request):
    articles = Article.objects.all()
    data = []
    for a in articles:
        data.append({
            'id': a.id,
            'title': a.title,
            'content': a.content,
            'author': a.author.username,  # FK relation — handled manually
            'created_at': a.created_at.isoformat(),  # datetime → string manually
        })
    return JsonResponse(data, safe=False)
    # And validation? And create/update? All by hand.
```

The serializer automates: conversion, validation, `create()`/`update()`, declaratively.

---

## 1.1 Serializer vs ModelSerializer

### `Serializer` — full control, everything declared by hand

```python
from rest_framework import serializers

class ArticleSerializer(serializers.Serializer):
    id = serializers.IntegerField(read_only=True)
    title = serializers.CharField(max_length=200)
    content = serializers.CharField()
    published = serializers.BooleanField(default=False)
    created_at = serializers.DateTimeField(read_only=True)

    def create(self, validated_data):
        # You implement creation yourself
        return Article.objects.create(**validated_data)

    def update(self, instance, validated_data):
        # And the update
        instance.title = validated_data.get('title', instance.title)
        instance.content = validated_data.get('content', instance.content)
        instance.published = validated_data.get('published', instance.published)
        instance.save()
        return instance
```

**When to use it:** data that doesn't map to a model (calculation results, aggregations, external data), or when you want absolute control over the structure — for example a login/contact form that saves nothing:

```python
class LoginSerializer(serializers.Serializer):
    email = serializers.EmailField()
    password = serializers.CharField(write_only=True)

class ContactSerializer(serializers.Serializer):
    name = serializers.CharField()
    message = serializers.CharField()
    # No create()/update() needed — we send an email, we don't save anything
```

### `ModelSerializer` — automatic generation from the model

```python
from rest_framework import serializers
from .models import Article

class ArticleSerializer(serializers.ModelSerializer):
    class Meta:
        model = Article
        fields = ['id', 'title', 'content', 'published', 'created_at', 'author']
        read_only_fields = ['id', 'created_at']
```

`ModelSerializer` does three things for you:
1. Generates the fields by inspecting `Article._meta`
2. Generates the validators (e.g. uniqueness if `unique=True` on the model)
3. Implements `create()` and `update()` via `instance.save()`

**Practical rule:** always use `ModelSerializer`, unless you have an explicit reason not to (data without a model behind it).

**Security rule:** always list `fields` explicitly. Never `__all__` in production — accidentally exposing a field (hashed password, internal token, internal notes) is a real vulnerability.

---

## 1.2 Field options — the details that matter

```python
class ArticleSerializer(serializers.ModelSerializer):

    # read_only=True: present in output (GET), ignored in input (POST/PUT)
    # Server-computed fields: id, created_at, auto-generated slug
    id = serializers.IntegerField(read_only=True)

    # write_only=True: accepted as input, never exposed in output
    # Typical case: password. Accepted at creation, never returned.
    password = serializers.CharField(write_only=True)

    # source: maps an API field name to a different name on the model
    # The field is called 'title' in the API, but 'headline' on the model
    title = serializers.CharField(source='headline')

    # source with relation traversal: accesses author.username via the author FK
    author_name = serializers.CharField(source='author.username', read_only=True)

    class Meta:
        model = Article
        fields = ['id', 'title', 'author_name', 'password']
        extra_kwargs = {
            'author': {'read_only': True},   # set server-side, never by the client
            'content': {'min_length': 10},
        }
```

| Option | What it does | Use case |
|---|---|---|
| `read_only=True` | Present in response, ignored in input | `id`, `created_at`, `author` |
| `write_only=True` | Accepted in input, absent from response | `password`, `token` |
| `required=False` | Optional in input | Fields with a default value |
| `source='field'` | Maps to a different attribute | Renaming a field, traversing a relation |
| `SerializerMethodField` | Computed field via `get_<name>` | Derived data |

### `SerializerMethodField` — read-only computed field

DRF automatically looks for a method named `get_<field_name>`. For values that aren't directly on the model but are computed from the instance.

```python
class ArticleSerializer(serializers.ModelSerializer):

    word_count = serializers.SerializerMethodField()

    def get_word_count(self, obj):
        # obj = the model instance being serialized
        return len(obj.body.split())

    is_recent = serializers.SerializerMethodField()

    def get_is_recent(self, obj):
        from django.utils import timezone
        from datetime import timedelta
        return obj.created_at >= timezone.now() - timedelta(days=7)

    class Meta:
        model = Article
        fields = ['id', 'title', 'word_count', 'is_recent']
```

---

## 1.3 Validation — three levels

DRF validates in cascade: first each field individually (automatic), then per-field validation, then cross-field validation.

```
request.data (raw JSON dict)
    │
    ▼
Each field converted to the correct Python type (automatic)
    │
    ▼
validate_<field>() called for each field that has one
    │
    ▼
validate() called for rules spanning multiple fields
    │
    ▼
serializer.validated_data  ← clean data, ready to save
    │
    ▼
serializer.save()  →  create() or update()
```

### Level 1 — automatic
DRF validates the type (`CharField`, `IntegerField`...), the model's constraints (`max_length`, `null=False`...), and the model's validators.

### Level 2 — `validate_<field_name>`: validating one specific field

```python
class CommentSerializer(serializers.ModelSerializer):

    def validate_body(self, value):
        # value has already passed type validation (it's a str)
        if len(value) < 10:
            # Error attached to the 'body' field:
            # {"body": ["The comment must be at least 10 characters."]}
            raise serializers.ValidationError("The comment must be at least 10 characters.")
        # Always return the value (possibly transformed)
        return value.strip()

    class Meta:
        model = Comment
        fields = ['id', 'body', 'created_at']
```

### Level 3 — `validate()`: cross-field validation

```python
class ArticleSerializer(serializers.ModelSerializer):

    def validate(self, data):
        # data = the full dict of already individually-validated fields
        if data.get('status') == 'published' and not data.get('title'):
            raise serializers.ValidationError(
                "A published article must have a title."
            )

        if data.get('published_at') and data.get('created_at'):
            if data['published_at'] < data['created_at']:
                raise serializers.ValidationError(
                    "The publication date cannot be earlier than the creation date."
                )

        # Always return data (possibly modified)
        return data

    class Meta:
        model = Article
        fields = ['id', 'title', 'status', 'published_at', 'created_at']
```

### In a view

```python
def post(self, request):
    serializer = ArticleSerializer(data=request.data)

    if serializer.is_valid():
        serializer.save(author=request.user)  # inject the author server-side
        return Response(serializer.data, status=201)

    return Response(serializer.errors, status=400)
    # serializer.errors → {'title': ['Title must be at least 5 characters.']}
```

Or cleaner with `raise_exception=True`:

```python
def post(self, request):
    serializer = ArticleSerializer(data=request.data)
    serializer.is_valid(raise_exception=True)  # automatically raises a 400 if invalid
    serializer.save(author=request.user)
    return Response(serializer.data, status=201)
```

---

## 1.4 `to_representation` — full control over the output

Called during serialization (object → JSON). Override it when you want to transform the final representation without affecting validation.

```python
class ArticleSerializer(serializers.ModelSerializer):

    def to_representation(self, instance):
        # Call the parent logic first to get the standard dict
        representation = super().to_representation(instance)

        # Case 1: conditionally remove fields
        # E.g. don't expose 'internal_notes' to non-admin users
        request = self.context.get('request')
        if request and not request.user.is_staff:
            representation.pop('internal_notes', None)

        # Case 2: transform a field
        if representation.get('title'):
            representation['title'] = representation['title'].upper()

        # Case 3: add complex computed data
        representation['meta'] = {
            'url': f"/api/articles/{instance.pk}/",
            'type': 'article'
        }

        return representation

    class Meta:
        model = Article
        fields = ['id', 'title', 'body', 'internal_notes']
```

**Key difference from `SerializerMethodField`**: `to_representation` modifies the entire output dict structure. `SerializerMethodField` adds one individual computed field. For a single computed field → `SerializerMethodField`. To transform the overall structure or conditionally remove fields → `to_representation`.

---

## 1.5 Relations and nested serializers

### The 3 strategies for representing a FK/M2M

```python
# Strategy 1: PrimaryKeyRelatedField (ModelSerializer's default)
# Represents the relation by its id. Simple, lightweight.
# Output: {"author": 42}
author = serializers.PrimaryKeyRelatedField(queryset=User.objects.all())

# Strategy 2: StringRelatedField
# Calls __str__() on the related object. Read-only.
# Output: {"author": "john_doe"}
author = serializers.StringRelatedField()

# Strategy 3: Nested Serializer
# Serializes the entire related object. The most expressive.
# Output: {"author": {"id": 42, "username": "john_doe", "email": "..."}}
author = UserSerializer(read_only=True)
```

### Nested serializer in practice — separate read and write

```python
class AuthorSerializer(serializers.ModelSerializer):
    """Lightweight serializer to represent an author in a nested context."""
    class Meta:
        model = User
        fields = ['id', 'username', 'email']


class CommentSerializer(serializers.ModelSerializer):
    # read_only=True on the nested serializer: return the full object on read
    author = AuthorSerializer(read_only=True)

    # For writing, a separate field that accepts the ID
    # write_only=True so it doesn't appear in GET responses
    author_id = serializers.PrimaryKeyRelatedField(
        queryset=User.objects.all(),
        source='author',  # maps to the same 'author' field on the model
        write_only=True
    )

    class Meta:
        model = Comment
        fields = ['id', 'body', 'author', 'author_id', 'created_at']
```

### Nested writable — creating related objects in a single request

```python
class ArticleWithTagsSerializer(serializers.ModelSerializer):
    # many=True for an M2M or reverse FK relation
    tags = TagSerializer(many=True)

    def create(self, validated_data):
        # Nested data must be extracted before creating the object,
        # since Article.objects.create() can't handle nested dicts
        tags_data = validated_data.pop('tags')
        article = Article.objects.create(**validated_data)

        for tag_data in tags_data:
            Tag.objects.create(article=article, **tag_data)

        return article

    def update(self, instance, validated_data):
        tags_data = validated_data.pop('tags', None)

        for attr, value in validated_data.items():
            setattr(instance, attr, value)
        instance.save()

        # Tag update: "replace all" strategy
        if tags_data is not None:
            instance.tags.all().delete()
            for tag_data in tags_data:
                Tag.objects.create(article=instance, **tag_data)

        return instance

    class Meta:
        model = Article
        fields = ['id', 'title', 'tags']
```

---

## 1.6 Serializer context — `self.context`

The serializer receives `request`, `view`, and `format` by default. Accessible via `self.context` for dynamic behavior.

```python
class ArticleSerializer(serializers.ModelSerializer):

    def to_representation(self, instance):
        rep = super().to_representation(instance)

        request = self.context.get('request')
        if request:
            rep['full_url'] = request.build_absolute_uri(f'/api/articles/{instance.pk}/')
            if not request.user.is_authenticated:
                rep.pop('private_notes', None)

        return rep

    class Meta:
        model = Article
        fields = ['id', 'title', 'private_notes']
```

Passing custom context from a view:

```python
serializer = ArticleSerializer(
    article,
    context={'request': request, 'include_stats': True}
)
```

---

## 1.7 The three uses of a serializer

```python
# ── Serialization (object → JSON) ──────────────────────────────────────────
article = Article.objects.get(id=1)
serializer = ArticleSerializer(article)
print(serializer.data)
# {'id': 1, 'title': 'My article', 'content': '...', ...}

# For a list:
articles = Article.objects.all()
serializer = ArticleSerializer(articles, many=True)
print(serializer.data)
# [{'id': 1, ...}, {'id': 2, ...}]

# ── Deserialization (JSON → object, creation) ───────────────────────────────
serializer = ArticleSerializer(data=request.data)
serializer.is_valid(raise_exception=True)
article = serializer.save(author=request.user)  # calls create()

# ── Deserialization (JSON → object, modification) ───────────────────────────
article = Article.objects.get(id=1)
serializer = ArticleSerializer(article, data=request.data)               # full PUT
serializer = ArticleSerializer(article, data=request.data, partial=True) # partial PATCH
serializer.is_valid(raise_exception=True)
serializer.save()  # calls update()
```

`partial=True` is essential for `PATCH` — without it, every `required` field must be present even if you're only modifying one.

---

## 1.8 The absolute basics (APIView + simple Serializer)

Before `ModelSerializer` and relations, here's the bare minimum to understand the flow: an `APIView` that receives JSON, validates it via a `Serializer`, and responds.

```python
class HealthCheck(APIView):
    """
    Simple health-check endpoint.
    """

    def get(self, request, format=None):
        data = {
            "firstRule": "Hello world",
        }
        return Response(data, status=status.HTTP_200_OK)


class HelloAPIView(APIView):
    def post(self, request, format=None):
        serializer = HelloSerializer(data=request.data)
        serializer.is_valid(raise_exception=True)

        name = serializer.validated_data["name"]
        age = serializer.validated_data["age"]

        return Response(
            {
                "hello": name,
                "age": age,
            },
            status=status.HTTP_200_OK,
        )
```

```python
from rest_framework import serializers


class HelloSerializer(serializers.Serializer):
    name = serializers.CharField(max_length=50)
    age = serializers.IntegerField()

    def validate_name(self, value):
        value = value.strip()
        if len(value) < 2:
            raise serializers.ValidationError("Name must be at least 2 characters.")
        return value
```

**Walkthrough:**
1. `HelloSerializer(data=request.data)` — creates the serializer. `request.data` is the JSON sent by the client; DRF converts it to a Python dict.
2. `serializer.is_valid(raise_exception=True)` — validation. If invalid, DRF automatically returns a 400 error; no need for try/catch.
3. `serializer.validated_data["name"]` / `["age"]` — retrieving the validated, cleaned data.
4. `Response(...)` — builds the JSON response.

---

## 1.9 Organizing serializers in a project

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
