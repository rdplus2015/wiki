# Chapter 0 — Roadmap & Overview

> Entry point of the guide. Used to know where we are and where we're going.

---

## 1. The general roadmap of a DRF project

Regardless of project size, the logical build order is always the same:

1. **Models** — define your tables (`models.py`)
2. **Migrations** — `makemigrations` + `migrate`
3. **Serializers** — how models turn into JSON (and back)
4. **Views / ViewSets** — the business logic (who can do what)
   - **Services (optional, once business logic grows)** — a `services.py` layer between the view and the ORM to pull complex business logic out of views/serializers (see Chapter 5). No need for a DAO/Repository in Django: the queryset (`Note.objects.filter(...)`) is already an abstraction over the database, unlike in Java/C# where that pattern exists to decouple data access from the persistence technology.
5. **URLs / Routers** — wire the views to endpoints
6. **Permissions / Auth** — who has access to what (usually DRF + JWT or Session)
7. **Tests** — verify it works (`APITestCase`)
8. **Swagger / OpenAPI** — document the API (often last, or ongoing)
9. **Deployment** — Hostinger, Docker, AWS, etc.

Each chapter of this guide maps to one or more of these steps, in this order.

---

## 2. The 3 ways to build a REST API with DRF — quick comparison

There are 3 levels of abstraction for writing DRF views. The full detail (with identical code examples for each approach) is in **Chapter 2 — Views & ViewSets**. Here's the overview to guide your choices from the start.

|                 | APIView      | Generic Views    | ViewSet + Router    |
|---|---|---|---|
| Code            | Verbose      | Concise          | Very concise          |
| URLs            | Manual       | Manual           | Automatic              |
| Flexibility     | Full         | Medium           | Good (+ `@action`)     |
| Custom actions  | Natural      | Not built-in     | `@action`               |
| Multi-actions   | 2 classes    | 2 classes        | 1 class                 |
| Typical use     | Auth, custom | Simple CRUD      | Full CRUD, production   |

**Logical learning progression:** understand APIView → use Generic Views → adopt ViewSet in production.

**Typical distribution observed in production (2026):** ~70% ViewSets, ~20% APIView (login, payment, special actions), ~10% Generic Views.

---

## 3. Complete checklist — building a REST API with DRF

This checklist applies to every project, from the simplest to the most complete. Useful as a review before considering a project "production-ready".

### Setup
- [ ] Django + DRF installed and in `requirements.txt`
- [ ] `'rest_framework'` in `INSTALLED_APPS`
- [ ] `REST_FRAMEWORK` config in `settings.py` (auth, pagination, exception handler)
- [ ] App(s) created with `django-admin startapp`
- [ ] App in `INSTALLED_APPS`

### Models
- [ ] Models defined in `models.py` only (never serializer/view logic mixed in)
- [ ] `__str__` defined on every model
- [ ] Fields with correct options (`null`, `blank`, `default`, `max_length`)
- [ ] Relations with explicit `related_name`
- [ ] Migrations generated and applied
- [ ] Models registered in `admin.py`

### Serializers
- [ ] `ModelSerializer` for models, `Serializer` for raw data (not tied to a model)
- [ ] `fields` explicit (never `__all__` in production)
- [ ] `read_only_fields` for `id`, `created_at`, and anything the client must not write
- [ ] `write_only=True` on sensitive fields (`password`, `token`)
- [ ] `validate_<field>` validation for per-field rules
- [ ] `validate()` for cross-field rules
- [ ] Server-side data injection (`author`, etc.) via `perform_create` / `perform_update`, never accepted from the client

### Views
- [ ] Right level chosen: APIView / Generic View / ViewSet depending on complexity (see table above)
- [ ] `get_queryset()` overridden when the queryset depends on the user or context
- [ ] `get_serializer_class()` overridden when the serializer differs by action
- [ ] `select_related` / `prefetch_related` in `get_queryset()` to avoid N+1
- [ ] `perform_create()` to inject the author server-side

### URLs
- [ ] Router for ViewSets (`DefaultRouter`)
- [ ] App URLs included in the main `urls.py` with `include()`
- [ ] `/api/` prefix on all routes
- [ ] Consistent route naming (`name=`)

### Auth & Permissions
- [ ] Auth method chosen (JWT, Session, Token) and configured in `settings.py`
- [ ] `DEFAULT_PERMISSION_CLASSES` defined (`IsAuthenticated` in general)
- [ ] `permission_classes` overridden per view/action when needed
- [ ] `has_object_permission` implemented for resources where only the owner can modify
- [ ] Register/login endpoint exposed

### Security
- [ ] `DEBUG=False` in production
- [ ] `SECRET_KEY` in an environment variable (`.env`), never in code
- [ ] `ALLOWED_HOSTS` configured
- [ ] CORS configured (`django-cors-headers`)
- [ ] Throttling configured for public endpoints
- [ ] No sensitive data in exposed fields

### Quality & production
- [ ] Custom exception handler for a uniform error format
- [ ] Pagination configured globally
- [ ] Filtering/search configured on list views
- [ ] `APITestCase` tests at minimum: list, create, update by non-owner (403), validation errors
- [ ] `select_related`/`prefetch_related` checked on all list views
- [ ] No bare `queryset.all()` without optimization in views with relations

---

## 4. Table of contents of the full guide

| Chapter | Topic |
|---|---|
| 0 | Roadmap & overview *(this file)* |
| 1 | Serializers |
| 2 | Views & ViewSets |
| 3 | Routers, Pagination, Filtering |
| 4 | Auth, JWT & Roles |
| 5 | Architecture & Best Practices |
| 6 | Guided project — Notebook API |
| 7 | Multi-tenant Guide |