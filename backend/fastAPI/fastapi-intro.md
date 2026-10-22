# FastAPI: introduction and how it works

## What is FastAPI?

A Python framework for building **web APIs** (REST mostly). Its strengths:

- **Built on Python type hints**: the types declared in the code are used to validate, convert and document.
- **Automatic documentation**: the API documents itself (Swagger UI).
- **Fast and lightweight**: it does not ship with a built-in ORM, admin or authentication system (unlike Django). You assemble the pieces you want.
- **Asynchronous**: natively supports `async`/`await`.

## The components (the stack)

| Component | Role |
|---|---|
| **FastAPI** | The layer that defines routes, validation, documentation and dependency injection. |
| **Starlette** | The web framework under FastAPI: routing, HTTP requests/responses, middlewares. |
| **Pydantic** | Data validation and serialization based on types. |
| **ASGI** | The standard interface between the server and the application (the asynchronous successor of WSGI). |
| **Uvicorn** | The ASGI server that receives HTTP requests and calls the application. |

Path of a request: **client → Uvicorn → Starlette/FastAPI → your function → JSON response**.

## Minimal application

```python
from fastapi import FastAPI

app = FastAPI()

@app.get("/")
def envoyer_salutation():
    return {"message": "Bonjour, API!"}
```

- `app` is the object the server loads (`uvicorn main:app` = file `main.py`, variable `app`).
- The `@app.get("/")` decorator binds the function to the **GET** method and the `/` path.
- The other decorators: `@app.post`, `@app.put`, `@app.patch`, `@app.delete`.
- The function name is free: the client never sees it (it only appears in the generated docs).
- You return Python types (dict, list, Pydantic objects) and FastAPI converts them to **JSON** (with the right `Content-Type`).

## Running the application

```bash
# Development (auto-reload, 127.0.0.1:8000 by default)
fastapi dev app/main.py

# With uvicorn directly (control over host and port)
uvicorn main:app --host 0.0.0.0 --port 8080 --reload
```

- `fastapi dev` is a development shortcut that runs Uvicorn underneath. It is installed with `fastapi[standard]`.
- `--reload` is for development only (never in production).
- Inside a container, you need `--host 0.0.0.0`, otherwise the application cannot be reached from outside the container.

## Automatic documentation

Generated from the code, without writing anything:

- `/docs`: **Swagger UI**, an interactive interface to test the routes.
- `/redoc`: alternative documentation (ReDoc).
- `/openapi.json`: the raw **OpenAPI** specification, usable by other tools (client generation, etc.).

## Parameters

FastAPI decides where each parameter comes from based on how it is declared.

```python
@app.get("/stations/{station_id}")          # station_id = path parameter
def obtenir_station(station_id: int, min_velos: int = 0):   # min_velos = query parameter
    ...
```

| Type | How it is declared | Example URL |
|---|---|---|
| **Path** | Name in `{}` in the URL, **same name** in the function | `/stations/6001` |
| **Query** | Function parameter not in the path (default value = optional) | `/stations?min_velos=5` |
| **Body** | Parameter typed with a Pydantic model | JSON in a `POST` |
| **Header / cookie** | `Header()`, `Cookie()` | |

The declared type does **two things**: it converts (`"6001"` becomes `6001`) and it validates. An invalid value (`/stations/abc`) automatically returns a **422** error with the details of the problem, without any code on your part.

## Pydantic models

```python
from pydantic import BaseModel

class Station(BaseModel):
    id: int
    nom: str
    longitude: float
    latitude: float
    nb_ancrages: int
    nb_velos_disponibles: int
```

- Describes the **shape** of the data and validates it when the object is created.
- Used for three things: validation, JSON serialization and documentation.
- In DRF, this is the equivalent of the **serializer** (not of the database model).
- With a real database, you usually have two classes: a **database model** (SQLAlchemy, for example) and **Pydantic schemas** (what goes in and out of the API).

## `response_model`

```python
@app.get("/stations", response_model=list[Station])
def obtenir_stations():
    return stations
```

- Validates what the function returns.
- **Filters out** fields that are not part of the model (useful to avoid exposing a sensitive field).
- Documents the shape of the response in `/docs`.

## Status codes and errors

```python
from fastapi import HTTPException

raise HTTPException(status_code=404, detail="Station inexistante")
```

- `raise` interrupts the function. FastAPI produces the response with the given code and `{"detail": "..."}`.
- Default code: 200. You change it in the decorator: `@app.post("/stations", status_code=201)`.
- Common codes: 200 (OK), 201 (created), 204 (no content), 400 (bad request), 404 (not found), 422 (validation failed, automatic).

## `def` vs `async def`

- **`def`**: FastAPI runs it in a **thread pool**. If the function is slow, it only blocks its own thread.
- **`async def`**: FastAPI runs it directly in the **event loop** (a single thread). With `await`, the function yields to other requests while it waits.
- **Pitfall**: a blocking call (`time.sleep`, `requests.get`, a long computation) inside an `async def` blocks the **whole** server.
- Simple rule:
  - **async** libraries with `await`: `async def`
  - **synchronous** libraries, or simply handling in-memory data: `def`

## Route order matters

FastAPI checks routes **in the order they are declared**, from top to bottom, and uses the **first route whose path matches**. It stops there and never looks at the routes below.

The catch: a path parameter such as `{station_id}` matches **any text** in its position in the URL. A parameterized route can therefore "swallow" a fixed route that comes after it.

**Wrong order** (parameterized route first):

```python
@app.get("/stations/{station_id}")   # 1st
def obtenir_station(station_id: int): ...

@app.get("/stations/proches")        # 2nd ("proches" = "nearby")
def stations_proches(): ...
```

What happens on `GET /stations/proches`:

1. FastAPI tests the 1st route: `/stations/{station_id}` matches, with `station_id = "proches"`.
2. It stops there. The 2nd route is never examined.
3. FastAPI then tries to convert `"proches"` to `int`, fails, and returns a **422** error.

The fixed route is **unreachable**, and nothing in the code warns you about it.

**Right order** (fixed route first):

```python
@app.get("/stations/proches")        # 1st
def stations_proches(): ...

@app.get("/stations/{station_id}")   # 2nd
def obtenir_station(station_id: int): ...
```

- `GET /stations/proches`: the 1st route matches exactly, so it is used.
- `GET /stations/6001`: the 1st route does not match (`6001` is not `proches`), so FastAPI moves on to the 2nd, which matches with `station_id = 6001`.

**Rule: from the most specific to the most general.** Fixed routes first, parameterized routes after.

Order only matters between routes that can overlap. `/stations` and `/stations/{station_id}` do not overlap (the first has nothing after `/stations`), so their relative order does not matter.

## Typical project structure

```
API/
 ├── app/
 │   ├── main.py          # app creation, router inclusion
 │   ├── routers/         # routes per resource
 │   ├── schemas/         # Pydantic models
 │   ├── models/          # database models
 │   └── dependencies.py  # shared dependencies
 ├── dependances.txt
 ├── Dockerfile
 └── docker-compose.yml
```

## Full example (stations API)

```python
from fastapi import FastAPI, HTTPException
from pydantic import BaseModel

app = FastAPI()


class Station(BaseModel):
    id: int
    nom: str
    longitude: float
    latitude: float
    nb_ancrages: int
    nb_velos_disponibles: int


stations = [
    Station(id=6001, nom="Drummond / de Maisonneuve", longitude=-73.5763,
            latitude=45.4996, nb_ancrages=27, nb_velos_disponibles=12),
    # ...
]


@app.get("/")
def envoyer_salutation():
    return {"message": "Bonjour, API!"}


@app.get("/stations", response_model=list[Station])
def obtenir_stations():
    return stations


@app.get("/stations/{station_id}", response_model=Station)
def obtenir_station_par_id(station_id: int):
    for station in stations:
        if station.id == station_id:
            return station
    raise HTTPException(status_code=404, detail="Station inexistante")
```

## FastAPI vs DRF (to find your bearings)

| DRF | FastAPI |
|---|---|
| Model (`models.Model`) | SQLAlchemy / SQLModel model (not provided) |
| Serializer | Pydantic model |
| View / ViewSet | route function (`@app.get`, ...) |
| `urls.py` / router | decorators + `APIRouter` |
| `Response`, renderers | direct return of Python types, automatic JSON conversion |
| Browsable API | Swagger UI (`/docs`) |
| Batteries included (ORM, admin, auth) | you assemble the pieces yourself |