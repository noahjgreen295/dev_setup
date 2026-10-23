# The Python API Stack

Follow one request down the wire. It enters through the contract you published, passes the edge, gets picked up by a server process, routed by the framework, checked for identity, validated, handled by your code, turned into a query, sent through a driver, and finally hits disk. Then the response climbs back up. Every slab is a choice.

Companion to [the Python Toolchain Stack](python_toolchain_stack.md) and [the Python Docs Switchboard](python_docs_switchboard.md). State as of October 2026. For the styled, interactive version, open [python_api_stack.html](../html/python_api_stack.html) in a browser.

**Status key:** **PICK** = modern default · alt = solid alternative · ⚠️ edge = maintained, but teetering, too new, or overtaken · ◆×N = appears in N places

## Default builds

- **Default build: greenfield service:** OpenAPI 3.1 → Caddy → Uvicorn → FastAPI → OIDC + PyJWT → Pydantic → pydantic-settings → SQLAlchemy 2 + Alembic → psycopg 3 → PostgreSQL
- **Django shop build:** OpenAPI via drf-spectacular → nginx → Gunicorn → Django + DRF (or Django Ninja) → simplejwt / allauth → DRF serializers → Django ORM + migrations → psycopg 3 → PostgreSQL

---

## L9 · The contract

What the outside world sees: the shape of every endpoint, payload and error. Clients are built against this, so changing it carelessly breaks people you've never met.

**Pick:** REST over HTTP, described by OpenAPI 3.1, errors as RFC 9457

### Description format

| Tool | Status | Notes |
|---|---|---|
| OpenAPI 3.1 | **PICK** | Generated from your code by the framework, not written by hand. Aligned with JSON Schema; every tool reads it. |
| OpenAPI 3.2 | ⚠️ edge | Released Sept 2025 with streaming, nested tags and a QUERY method. Tooling support is still catching up. |
| RFC 9457 problem details | **PICK** | A standard JSON shape for errors (type, title, status, detail). Clients handle failures uniformly. |

### Interface style

| Tool | Status | Notes |
|---|---|---|
| REST / HTTP+JSON | **PICK** | The default. Cacheable, debuggable with curl, understood by every client. |
| GraphQL ◆×2 | alt | One endpoint, client-chosen fields. Good for many front ends with different data needs; harder to cache and rate-limit. |
| gRPC ◆×2 | alt | Binary protobuf over HTTP/2. Fast and strictly typed; best for service-to-service traffic, awkward for browsers. |
| AsyncAPI | alt | The contract for event-driven APIs: Kafka, MQTT, WebSockets. |

---

## L8 · Edge & gateway

The front door. Terminates TLS, speaks HTTP/2 and HTTP/3 to clients, compresses, rate-limits and load-balances, so your Python processes only see clean HTTP/1.1.

**Pick:** Caddy for a single box; your platform's load balancer in the cloud

| Tool | Status | Notes |
|---|---|---|
| Caddy | **PICK** | Automatic HTTPS with zero config, HTTP/3 out of the box, readable config file. |
| nginx | alt | The veteran. Fastest and most documented; certificates and HTTP/3 take more setup. |
| Traefik | alt | Discovers services from Docker and Kubernetes labels automatically. Natural fit for container setups. |
| Cloud load balancer | **PICK** | AWS ALB, Google Cloud Load Balancing, Azure Front Door. On a managed platform this is your edge. |
| Managed API gateway | alt | AWS API Gateway, Kong, Apigee: keys, quotas, analytics. Worth it when you sell API access or front many services. |

---

## L7 · Process server

The program that actually runs your Python app: accepts connections, speaks ASGI (async) or WSGI (sync) to the framework, and manages worker processes.

**Pick:** Uvicorn (one process per container, or under Gunicorn on a VM)

| Tool | Status | Notes |
|---|---|---|
| Uvicorn | **PICK** | The standard ASGI server for FastAPI and Starlette. On Kubernetes or Cloud Run, run one process per container and let the platform scale. |
| Gunicorn | **PICK** | The process manager: restarts crashed workers, graceful reloads. Default for sync Django and Flask; supervises Uvicorn via the uvicorn-worker package. |
| Granian | alt | Rust server for ASGI, WSGI and its own RSGI. 2.x series; faster in benchmarks, roughly even with Uvicorn in DB-bound apps. |
| Hypercorn | alt | The most protocol-complete: HTTP/2, HTTP/3 and trio support. Pick it only if Python itself must speak those. |
| uWSGI | ⚠️ edge | Powerful and sprawling. In maintenance mode since 2022; keep working deployments, don't start new ones. |
| “The Rest” | ⚠️ edge | Daphne, Waitress. Still maintained; Daphne mainly for Django Channels, Waitress for Windows-hosted WSGI. Rarely the right default. |

---

## L6 · Framework

Routing, middleware, request parsing, dependency injection and response building. The layer people mean when they say "use FastAPI".

**Pick:** FastAPI for new services; Django + DRF or Django Ninja inside a Django shop

### Async-first

| Tool | Status | Notes |
|---|---|---|
| FastAPI ◆×2 | **PICK** | Type hints drive validation, docs and DI. The most popular Python API framework; huge ecosystem. |
| Litestar | alt | FastAPI's main rival. More batteries (DTOs, channels, built-in auth primitives), multiple maintainers, fast msgspec core. |
| Starlette | alt | The toolkit FastAPI is built on. Use directly for minimal services where you want no magic. |

### Django & Flask world

| Tool | Status | Notes |
|---|---|---|
| Django REST Framework | **PICK** | The Django standard: serializers, viewsets, permissions, browsable API. Sync-first and mature. |
| Django Ninja | alt | FastAPI-style typed endpoints on top of Django and its ORM. Faster and less boilerplate than DRF. |
| Flask + APIFlask | alt | Flask stays great for small services; APIFlask adds typed schemas and OpenAPI. |

### Other contracts

| Tool | Status | Notes |
|---|---|---|
| Strawberry ◆×2 | **PICK** | GraphQL from Python type hints. Integrates with FastAPI and Django. |
| grpcio ◆×2 | **PICK** | The official gRPC runtime. Generate stubs from .proto files with grpcio-tools. |
| “The Rest” | ⚠️ edge | Sanic, Falcon, Quart, BlackSheep, Tornado, Bottle, Pyramid, Robyn. All maintained, but smaller ecosystems; rarely the right default for a new API. |

---

## L5 · Auth & security

Who is calling and what they may do. Authentication (identity), authorization (permissions), plus CORS, rate limits and secure headers.

**Pick:** Delegate login to an OIDC provider; verify JWTs with PyJWT

### Identity

| Tool | Status | Notes |
|---|---|---|
| OIDC provider | **PICK** | Auth0, Okta, Keycloak, Cognito, Entra ID. They handle login, MFA and password storage; your API just checks tokens. |
| Authlib | **PICK** | The complete OAuth 2 and OpenID Connect toolkit for Python, client and server side. |
| PyJWT | **PICK** | Decode and verify JWTs. The library FastAPI's own docs now use. |
| pwdlib + Argon2 | **PICK** | If you must store passwords yourself. Built because passlib stopped working on modern Python. |
| djangorestframework-simplejwt | alt | JWT auth for DRF. Pair with django-allauth for social login. |
| fastapi-users | ⚠️ edge | Ready-made user management for FastAPI. Maintenance mode as of 2026: security fixes only, successor in development. |
| python-jose | ⚠️ edge | Older JWT library with a long maintenance gap. Replaced by PyJWT in most guides. |

### Perimeter

| Tool | Status | Notes |
|---|---|---|
| CORS middleware | **PICK** | Built into every framework. Allow-list exact origins; never * with credentials. |
| slowapi / limits | alt | In-app rate limiting backed by Redis. Better done at the edge (L8) if you can. |

---

## L4 · Validation & serialization

Turn untrusted JSON into typed Python objects on the way in, and objects back into JSON on the way out. Most API bugs are shape bugs.

**Pick:** Pydantic

| Tool | Status | Notes |
|---|---|---|
| Pydantic | **PICK** | Rust-core validation from type hints. The same model validates, serializes and feeds the OpenAPI spec. |
| msgspec | alt | Faster and leaner than Pydantic for JSON and MessagePack. Litestar supports it natively. |
| orjson | alt | Very fast JSON encoding. Mostly unneeded now that Pydantic v2 serializes in Rust, but useful for raw dict responses. |
| attrs + cattrs | alt | Plain typed classes plus a separate structuring layer. Clean when you want domain objects free of validation logic. |
| DRF serializers | alt | Django REST Framework's built-in validation. Slower and more verbose, but integrated with the ORM. |
| “The Rest” | ⚠️ edge | marshmallow, Cerberus, voluptuous. Still maintained, but Pydantic has absorbed most of their audience. |

---

## L3 · App wiring

Your actual code and how it's assembled: configuration, dependency injection, and the separation between HTTP handlers, business logic and data access.

**Pick:** pydantic-settings + FastAPI Depends, routers → services → repositories

| Tool | Status | Notes |
|---|---|---|
| pydantic-settings | **PICK** | Typed config from environment variables and .env files, validated at startup so bad config fails fast. |
| FastAPI Depends ◆×2 | **PICK** | Built-in dependency injection: DB sessions, current user, settings, each injected per request and easy to override in tests. |
| Routers → services → repositories | **PICK** | Keep HTTP code thin, business logic framework-free, and database access behind one interface. Makes testing and framework changes cheap. |
| dependency-injector / svcs | alt | Container-style DI for larger apps or when you need it outside request handlers (workers, CLIs). |

---

## L2 · Data access & migrations

How Python objects become SQL, and how the schema evolves without losing data. Migrations are code; review them like code.

**Pick:** SQLAlchemy 2.0 + Alembic

| Tool | Status | Notes |
|---|---|---|
| SQLAlchemy 2.0 | **PICK** | The Python ORM and SQL toolkit. Fully typed 2.0 API, first-class async, drops to raw SQL whenever you want. |
| Alembic | **PICK** | Migrations for SQLAlchemy. Autogenerates diffs from your models; always read them before applying. |
| SQLModel | alt | SQLAlchemy and Pydantic in one class, from FastAPI's author. Pleasant for simple CRUD; slower-moving project. |
| Django ORM | **PICK** | Inside Django, use it. Built-in migrations and admin; async support is still partial. |
| Tortoise ORM / Piccolo | alt | Async-native ORMs with Django-like ergonomics. Smaller communities. |
| Raw SQL | alt | asyncpg or psycopg with hand-written queries. Fastest and most transparent; you own the mapping. |
| “The Rest” | ⚠️ edge | Peewee, Pony ORM. Still maintained, but few new projects choose them over SQLAlchemy. |

---

## L1 · Drivers & clients

The libraries that speak each datastore's wire protocol, plus the HTTP client for calling other services.

**Pick:** psycopg 3, redis-py, httpx

| Tool | Status | Notes |
|---|---|---|
| psycopg 3 | **PICK** | The modern PostgreSQL driver. Sync and async in one package, binary protocol, connection pooling. |
| asyncpg | alt | Async-only PostgreSQL driver, the fastest available. Great with SQLAlchemy async or raw SQL. |
| psycopg2 | ⚠️ edge | Still receiving maintenance releases, but psycopg 3 is the successor. Fine in existing code. |
| redis-py | **PICK** | The official Redis client, with built-in asyncio. Works with Valkey too. |
| PyMongo (async API) | **PICK** | For MongoDB, PyMongo now ships a native async API. |
| Motor | ⚠️ edge | The old async MongoDB driver. Deprecated May 2025, end of life May 2026; migrate to PyMongo async. |
| httpx | **PICK** | Sync and async HTTP client with a requests-style API. Also powers FastAPI's TestClient. |
| aiohttp client | alt | Mature async HTTP client; slightly faster, less friendly API. |

---

## L0 · Datastore

Where the data finally rests. Choose the boring, durable option unless you have a measured reason not to.

**Pick:** PostgreSQL

| Tool | Status | Notes |
|---|---|---|
| PostgreSQL | **PICK** | The default answer. Relational, JSONB for documents, full-text search, pgvector for embeddings. |
| SQLite | alt | Zero-ops, single file. Great for development, tests, small internal services and read-heavy apps. |
| MySQL / MariaDB | alt | Solid and widespread. Pick it when your org or host already runs it. |
| Redis / Valkey | **PICK** | In-memory cache, rate-limit counters, sessions, queues. Valkey is the Linux Foundation fork. |
| MongoDB | alt | Document store. Reach for it when your data really is schemaless; PostgreSQL JSONB covers many such cases. |
| DuckDB | alt | In-process analytics database. Ideal behind read-only analytical endpoints over Parquet or large tables. |

---

## Sidecars

Cross-cutting systems that ride alongside the request path rather than sitting in it. Skip these and the stack above still runs, right up until it breaks in production and nobody can see why.

### Testing

| Tool | Status | Notes |
|---|---|---|
| pytest + httpx | **PICK** | Call your app in-process with FastAPI's TestClient or httpx.AsyncClient. Override dependencies for fakes. |
| Schemathesis | **PICK** | Reads your OpenAPI spec and fires generated requests at every endpoint, finding crashes and contract violations you didn't think of. |
| Testcontainers | **PICK** | Spin up a real PostgreSQL or Redis in Docker for integration tests. No more SQLite-in-tests surprises. |
| polyfactory | alt | Generates test data straight from Pydantic models and dataclasses. |
| respx | alt | Mock outbound httpx calls. |
| Locust / k6 | alt | Load testing. Locust scripts are Python; k6 scripts are JavaScript. |

### Observability

| Tool | Status | Notes |
|---|---|---|
| OpenTelemetry | **PICK** | Vendor-neutral traces, metrics and logs. Auto-instrumentation for FastAPI, SQLAlchemy and httpx. |
| structlog | **PICK** | Structured JSON logging with request IDs bound to every line. |
| Sentry | alt | Error tracking with stack traces and the request that caused them. |
| Prometheus client | alt | Expose /metrics for scraping. Standard on Kubernetes. |
| Logfire | alt | Hosted observability from the Pydantic team, built on OpenTelemetry. Very low setup for FastAPI. |

### Background work

| Tool | Status | Notes |
|---|---|---|
| Celery | **PICK** | Still the standard task queue: retries, schedules, huge ecosystem. Heavy, but battle-tested. |
| Taskiq | alt | Async-native task queue with FastAPI-style dependency injection. |
| Dramatiq | alt | Simpler, saner Celery alternative for sync code. |
| arq / RQ | alt | Lightweight Redis-backed queues. arq is async; RQ is sync. |
| Temporal | alt | Durable workflows that survive crashes and redeploys. For multi-step business processes. |
| BackgroundTasks | alt | FastAPI's built-in fire-and-forget. Runs in-process, so it dies with the process; only for trivial work. |

### Ship it

| Tool | Status | Notes |
|---|---|---|
| Docker + uv | **PICK** | Multi-stage build, uv sync from the lockfile, slim runtime image, non-root user. |
| Kubernetes / ECS / Cloud Run | **PICK** | Pick the least platform that fits. Cloud Run and similar give autoscaling without running a cluster. |
| Mangum | alt | Runs ASGI apps on AWS Lambda behind API Gateway. Watch cold starts. |
| Health & readiness endpoints | **PICK** | /healthz for liveness, /readyz checking the DB. Orchestrators route traffic on these. |

---

## Rules of the road

- **One schema, three jobs.** The same Pydantic model validates the request, serializes the response and generates the OpenAPI spec. Keep it that way and docs can't drift from code.
- **Don't build auth.** Delegate login to an identity provider and only verify tokens in your API. Hand-rolled password storage is where breaches start.
- **Async all the way, or not at all.** One sync database call inside an async endpoint blocks the whole event loop. Pick async drivers or use plain def endpoints, which FastAPI runs in a thread pool.
- **Errors are part of the contract.** Return RFC 9457 problem-details JSON so clients can handle failures without parsing your prose.

## Pydantic & Redis footnotes

Pydantic sits in three layers here and its company also sells Logfire, so the ecosystem leans on one team. Redis switched licenses in 2024, prompting the Linux Foundation fork Valkey, then added an open-source AGPL option in 2025. Both remain drop-in for Python clients via redis-py. For docs and client generation, see the Docs Switchboard.

## Dog Solitude

The scrapyard. Dead, removed, deprecated or abandoned tools, by layer.

| Code | Short | Layer | Obsolete |
|---|---|---|---|
| L9 | contract | The contract | Swagger 2.0, RAML, API Blueprint |
| L7 | server | Process server | meinheld, uvicorn.workers (bundled Gunicorn worker) |
| L6 | framework | Framework | Hug, API Star, Molten, flask-restplus |
| L5 | identity | Auth & security | passlib, Flask-JWT, djangorestframework-jwt |
| L4 | shape | Validation & serialization | Pydantic v1, schematics |
| L2 | data | Data access & migrations | GINO |
| L1 | drivers | Drivers & clients | aioredis (merged into redis-py) |

## Sources

- [Motor docs (deprecation notice)](https://motor.readthedocs.io/en/latest/)
- [Python Application Servers in 2026 (DeployHQ)](https://www.deployhq.com/blog/python-application-servers-wsgi-vs-asgi-guide)
- [uvicorn-worker on PyPI](https://pypi.org/project/uvicorn-worker)
- [fastapi-users status (openapps.pro)](https://openapps.pro/packages/fastapi-users)
- [Introducing pwdlib (François Voron)](https://polar.sh/frankie567/posts/introducing-pwdlib-a-modern-password-hash-helper-for-python)
