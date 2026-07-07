# Exercises — Chapter 1: Serializers

> File independent from the guide. Solutions are at the bottom, in a separate section — try each exercise before looking.

Reference models (Notebook project):

```python
# models.py
from django.db import models
from django.contrib.auth.models import User

class Category(models.Model):
    name = models.CharField(max_length=100)
    owner = models.ForeignKey(User, on_delete=models.CASCADE)

    def __str__(self):
        return self.name

class Note(models.Model):
    title = models.CharField(max_length=200)
    body = models.TextField()
    category = models.ForeignKey(
        Category, on_delete=models.SET_NULL, null=True, blank=True,
        related_name='notes'
    )
    author = models.ForeignKey(User, on_delete=models.CASCADE, related_name='notes')
    is_pinned = models.BooleanField(default=False)
    created_at = models.DateTimeField(auto_now_add=True)
    updated_at = models.DateTimeField(auto_now=True)

    def __str__(self):
        return self.title
```

---

## Exercise 1 — Basic ModelSerializer

Create `CategorySerializer`:
- Exposes `id`, `name`
- `id` read-only
- Does NOT expose `owner` (will be injected server-side)

**Test in the Django shell** (`python manage.py shell`):
```python
from myapp.models import Category
from myapp.serializers import CategorySerializer

cat = Category.objects.first()
serializer = CategorySerializer(cat)
print(serializer.data)
```

---

## Exercise 2 — SerializerMethodField

Add a `note_count` field to `CategorySerializer`: the number of notes linked to this category.

Hint: `related_name='notes'` on the `Note` FK gives you access to `category.notes`.

---

## Exercise 3 — validate_<field>

On `NoteSerializer` (`ModelSerializer` based on `Note`, fields `id, title, body, is_pinned, created_at`):
- `title` must be at least 3 characters (strip before checking)
- `body` must be at least 10 characters (strip before checking)

Test with invalid data in the shell and check the content of `serializer.errors`.

---

## Exercise 4 — Cross-field validate()

Add a rule to `NoteSerializer`: a note cannot be pinned (`is_pinned=True`) if its `body` is less than 20 characters (arbitrary rule, just to practice cross-field validation).

Hint: this happens in `validate(self, data)`, not in `validate_body`.

---

## Exercise 5 — Read-only relation + separate write field

On `NoteSerializer`, expose the category in two different ways:
- `category`: full object on read (use `CategorySerializer`)
- `category_id`: accepts an ID on write (`PrimaryKeyRelatedField`, `write_only=True`, `source='category'`)

Test that `POST` with `{"category_id": 1, ...}` works, and that the response contains a full `category` object.

---

## Exercise 6 — to_representation

On `NoteSerializer`, format `created_at` as `DD/MM/YYYY HH:MM` instead of the default ISO format, only in the output (don't touch validation).

---

## Exercise 7 — Restricting a queryset using context

Take `category_id` (Exercise 5) and restrict its `queryset` to categories that belong to `self.context['request'].user` — not every category in the DB.

Hint: this happens in `__init__`, modifying `self.fields['category_id'].queryset` after calling `super().__init__()`.

---

## Exercise 8 — Serializer not tied to a model

Create a `NoteSearchSerializer` (no `Meta`/`model`) that validates search parameters:
- `query`: `CharField`, required, min 2 characters
- `pinned_only`: `BooleanField`, optional, default `False`

This serializer saves nothing — it just validates query params before building a queryset.

---

## Exercise 9 — `source` to rename a field

Imagine your `Note` model actually has a `content` field (not `body`) — sometimes you inherit an existing model you can't rename, but you want the API to expose a clearer name.

Create a `NoteSerializer` where the API field is called `body`, but it actually reads/writes the model's `content` attribute (use `source`).

Verify in the shell that `serializer.data` contains `body`, never `content`.

---

## Exercise 10 — `write_only` on a sensitive field + cross-field validation

Create a `RegisterSerializer` (not based on `Note`/`Category`, but on `User` — `from django.contrib.auth.models import User`):
- `username`, `email`: normal
- `password`: `write_only=True`
- `password2`: `write_only=True`, used for confirmation, doesn't exist on the `User` model
- `validate()`: checks that `password == password2`, otherwise raises an error
- `create()`: uses `User.objects.create_user(...)` (never `User.objects.create()`, which doesn't hash the password), and removes `password2` before creating

Verify in the shell that `serializer.data` (after `save()`) never contains `password` nor `password2`.

---

## Exercise 11 — Manual practice of the 3 uses (shell)

In `python manage.py shell`, on `NoteSerializer`:

1. **Serialize a single instance**: fetch an existing note, serialize it, print `.data`.
2. **Serialize a list**: `Note.objects.all()`, with `many=True`, print `.data`.
3. **Create**: `NoteSerializer(data={...})`, `is_valid(raise_exception=True)`, `save(author=some_user)`. Verify the object exists in the DB afterward.
4. **Full update (`PUT`)**: take the note you just created, `NoteSerializer(note, data={...all fields...})`, `is_valid()`, `save()`.
5. **Partial update (`PATCH`)**: the same note, but `NoteSerializer(note, data={"title": "New title"}, partial=True)`. Verify that `body` wasn't wiped out.
6. Try step 4 **without** `partial=True`, sending only one field — observe the DRF error about missing fields. This is to really understand why `partial=True` exists.

---

## Exercise 12 — Nested writable

Add a `Tag` model:

```python
class Tag(models.Model):
    note = models.ForeignKey(Note, on_delete=models.CASCADE, related_name='tags')
    label = models.CharField(max_length=50)
```

Create a `NoteWithTagsSerializer` that:
- Exposes `tags` as a list of `TagSerializer` (`many=True`), both writable AND readable (not `read_only`)
- Overrides `create()`: extracts `tags` from `validated_data` with `.pop()`, creates the `Note`, then creates each linked `Tag`
- Overrides `update()`: "replace all" strategy — deletes all existing tags on the note, recreates the ones sent

Test in the shell: create a note with 2 tags in a single request (simulated `POST`), then update by completely changing the tags.

---

## Exercise 13 — Organizing serializers in a real project (design, no code)

Your Notebook project is growing. You now have 3 Django apps: `accounts` (users/auth), `notes` (Note, Category), `tags` (Tag, shared across several resources).

Without writing code, answer:
1. Where does `RegisterSerializer` live?
2. Where do `NoteSerializer` and `CategorySerializer` live?
3. `TagSerializer` is used by both `notes` and a future `articles` app — where do you put it to avoid duplication?
4. Should the lightweight `NoteListSerializer` and the full `NoteDetailSerializer` live in separate files or the same `serializers.py`?

---
---

# Solutions

<details>
<summary>Exercise 1</summary>

```python
class CategorySerializer(serializers.ModelSerializer):
    class Meta:
        model = Category
        fields = ['id', 'name']
        read_only_fields = ['id']
```
</details>

<details>
<summary>Exercise 2</summary>

```python
class CategorySerializer(serializers.ModelSerializer):
    note_count = serializers.SerializerMethodField()

    def get_note_count(self, obj):
        return obj.notes.count()

    class Meta:
        model = Category
        fields = ['id', 'name', 'note_count']
        read_only_fields = ['id']
```
</details>

<details>
<summary>Exercise 3</summary>

```python
class NoteSerializer(serializers.ModelSerializer):

    def validate_title(self, value):
        value = value.strip()
        if len(value) < 3:
            raise serializers.ValidationError("Title must be at least 3 characters.")
        return value

    def validate_body(self, value):
        value = value.strip()
        if len(value) < 10:
            raise serializers.ValidationError("Body must be at least 10 characters.")
        return value

    class Meta:
        model = Note
        fields = ['id', 'title', 'body', 'is_pinned', 'created_at']
        read_only_fields = ['id', 'created_at']
```
</details>

<details>
<summary>Exercise 4</summary>

```python
class NoteSerializer(serializers.ModelSerializer):
    # ... validate_title, validate_body as before ...

    def validate(self, data):
        if data.get('is_pinned') and len(data.get('body', '')) < 20:
            raise serializers.ValidationError(
                "A pinned note must have a body of at least 20 characters."
            )
        return data

    class Meta:
        model = Note
        fields = ['id', 'title', 'body', 'is_pinned', 'created_at']
        read_only_fields = ['id', 'created_at']
```
</details>

<details>
<summary>Exercise 5</summary>

```python
class NoteSerializer(serializers.ModelSerializer):
    category = CategorySerializer(read_only=True)
    category_id = serializers.PrimaryKeyRelatedField(
        queryset=Category.objects.all(),
        source='category',
        write_only=True,
        required=False,
        allow_null=True
    )

    class Meta:
        model = Note
        fields = ['id', 'title', 'body', 'category', 'category_id', 'is_pinned', 'created_at']
        read_only_fields = ['id', 'created_at']
```
</details>

<details>
<summary>Exercise 6</summary>

```python
class NoteSerializer(serializers.ModelSerializer):

    def to_representation(self, instance):
        rep = super().to_representation(instance)
        rep['created_at'] = instance.created_at.strftime('%d/%m/%Y %H:%M')
        return rep

    class Meta:
        model = Note
        fields = ['id', 'title', 'body', 'is_pinned', 'created_at']
        read_only_fields = ['id', 'created_at']
```
</details>

<details>
<summary>Exercise 7</summary>

```python
class NoteSerializer(serializers.ModelSerializer):
    category = CategorySerializer(read_only=True)
    category_id = serializers.PrimaryKeyRelatedField(
        queryset=Category.objects.none(),  # placeholder, restricted in __init__
        source='category',
        write_only=True,
        required=False,
        allow_null=True
    )

    def __init__(self, *args, **kwargs):
        super().__init__(*args, **kwargs)
        request = self.context.get('request')
        if request and request.user.is_authenticated:
            self.fields['category_id'].queryset = Category.objects.filter(owner=request.user)

    class Meta:
        model = Note
        fields = ['id', 'title', 'body', 'category', 'category_id', 'is_pinned', 'created_at']
        read_only_fields = ['id', 'created_at']
```
</details>

<details>
<summary>Exercise 8</summary>

```python
class NoteSearchSerializer(serializers.Serializer):
    query = serializers.CharField(min_length=2)
    pinned_only = serializers.BooleanField(required=False, default=False)
    # No create()/update() — this serializer never saves anything
```
</details>

<details>
<summary>Exercise 9</summary>

```python
class NoteSerializer(serializers.ModelSerializer):
    # 'body' is the API-facing name, 'content' is the actual model attribute
    body = serializers.CharField(source='content')

    class Meta:
        model = Note
        fields = ['id', 'title', 'body', 'is_pinned', 'created_at']
        read_only_fields = ['id', 'created_at']

# Shell verification:
# note = Note.objects.first()
# NoteSerializer(note).data
# → {'id': ..., 'title': ..., 'body': '...', ...}  never a 'content' key
```

Note: when a field uses `source`, it must **not** appear twice in `Meta.fields` under its real name (`content`) — only the API name (`body`) goes there.
</details>

<details>
<summary>Exercise 10</summary>

```python
from django.contrib.auth.models import User
from rest_framework import serializers

class RegisterSerializer(serializers.ModelSerializer):
    password = serializers.CharField(write_only=True, min_length=8)
    password2 = serializers.CharField(write_only=True)

    class Meta:
        model = User
        fields = ['id', 'username', 'email', 'password', 'password2']

    def validate(self, data):
        if data['password'] != data['password2']:
            raise serializers.ValidationError({'password': 'Passwords do not match.'})
        return data

    def create(self, validated_data):
        validated_data.pop('password2')
        # create_user() correctly hashes the password — never User.objects.create()
        return User.objects.create_user(**validated_data)

# Shell verification:
# serializer = RegisterSerializer(data={
#     "username": "alice", "email": "a@a.com",
#     "password": "testpass123", "password2": "testpass123"
# })
# serializer.is_valid(raise_exception=True)
# user = serializer.save()
# serializer.data  → contains neither 'password' nor 'password2' (write_only excludes them from output)
```
</details>

<details>
<summary>Exercise 11</summary>

```python
from myapp.models import Note
from myapp.serializers import NoteSerializer
from django.contrib.auth.models import User

user = User.objects.first()

# 1. Serialize an instance
note = Note.objects.first()
print(NoteSerializer(note).data)

# 2. Serialize a list
notes = Note.objects.all()
print(NoteSerializer(notes, many=True).data)

# 3. Create
serializer = NoteSerializer(data={"title": "My note", "body": "Content long enough here"})
serializer.is_valid(raise_exception=True)
note = serializer.save(author=user)
print(Note.objects.filter(pk=note.pk).exists())  # True

# 4. Full update (PUT) — ALL required fields must be sent
serializer = NoteSerializer(note, data={"title": "Modified title", "body": "New content, long enough", "is_pinned": False})
serializer.is_valid(raise_exception=True)
serializer.save()

# 5. Partial update (PATCH)
serializer = NoteSerializer(note, data={"title": "New title"}, partial=True)
serializer.is_valid(raise_exception=True)
serializer.save()
note.refresh_from_db()
print(note.body)  # still "New content, long enough", not wiped out

# 6. Without partial=True, with only one field → error
serializer = NoteSerializer(note, data={"title": "Title alone"})  # no partial=True
print(serializer.is_valid())  # False
print(serializer.errors)  # {'body': ['This field is required.']}
```
</details>

<details>
<summary>Exercise 12</summary>

```python
# models.py
class Tag(models.Model):
    note = models.ForeignKey(Note, on_delete=models.CASCADE, related_name='tags')
    label = models.CharField(max_length=50)


# serializers.py
class TagSerializer(serializers.ModelSerializer):
    class Meta:
        model = Tag
        fields = ['id', 'label']


class NoteWithTagsSerializer(serializers.ModelSerializer):
    tags = TagSerializer(many=True)  # not read_only: accepted on write too

    def create(self, validated_data):
        tags_data = validated_data.pop('tags')
        note = Note.objects.create(**validated_data)
        for tag_data in tags_data:
            Tag.objects.create(note=note, **tag_data)
        return note

    def update(self, instance, validated_data):
        tags_data = validated_data.pop('tags', None)

        for attr, value in validated_data.items():
            setattr(instance, attr, value)
        instance.save()

        if tags_data is not None:
            instance.tags.all().delete()
            for tag_data in tags_data:
                Tag.objects.create(note=instance, **tag_data)

        return instance

    class Meta:
        model = Note
        fields = ['id', 'title', 'body', 'tags']

# Shell verification:
# serializer = NoteWithTagsSerializer(data={
#     "title": "Tagged note", "body": "Content long enough to validate",
#     "tags": [{"label": "urgent"}, {"label": "work"}]
# })
# serializer.is_valid(raise_exception=True)
# note = serializer.save(author=user)
# note.tags.count()  → 2
```
</details>

<details>
<summary>Exercise 13</summary>

1. `RegisterSerializer` → `accounts/serializers.py`. It deals with creating a `User`, so it logically belongs to the app that manages authentication, not to `notes`.

2. `NoteSerializer` and `CategorySerializer` → `notes/serializers.py`. Both models live in `notes/models.py`, so their serializers stay in the same app.

3. A `Tag` shared across several business apps shouldn't live inside one specific business app. Two common options:
   - Create a dedicated `tags/` app with its own `models.py` + `serializers.py`, and other apps import `from tags.serializers import TagSerializer`
   - Or, if `Tag` is fundamentally tied to one specific resource (e.g. every `Tag` has a FK to `Note` only, no real generic relation), keep it in `notes/` and don't over-engineer before you actually need to
   Practical rule: as soon as a model is used generically by 2+ apps (not just a one-directional FK), it deserves its own app.

4. Same file, `notes/serializers.py`. The "1 app = its own `serializers.py`" convention doesn't mean "1 file per serializer" — several serializers for the same resource (list, detail, write) stay grouped in the same file, since they share the same `Meta.model` and are often used together in the same `views.py`. Splitting into multiple files only becomes useful once `serializers.py` grows past several hundred lines.
</details>
