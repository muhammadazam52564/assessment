# Backend and Database Requirements

This document sets the technology choices and standards for the POC backend and database. It follows the team's existing FastAPI standard (the RIVA backend) so the code looks and behaves the same as our other projects. It supports `data-model-requirements.md` and `integration-requirements.md`.

## 1. Decisions Summary

| Concern | Choice | Why |
| --- | --- | --- |
| Language | Python 3.12+ | Matches our standard; strong SOAP, XML and HTTP libraries |
| Web framework | FastAPI (async) | Our standard; automatic API docs; good fit for I/O-heavy carrier calls |
| Server | uvicorn | Standard FastAPI server |
| Database | PostgreSQL 16 | Strong transactions, triggers (needed for immutable history), JSON columns, trigram search |
| ORM | SQLAlchemy 2.0 (async, `Mapped` / `mapped_column`) | Our standard |
| DB drivers | asyncpg (runtime), psycopg2 (Alembic only) | Our standard |
| Migrations | Alembic | Versioned, reviewable schema changes, including triggers |
| Validation / settings | Pydantic v2, pydantic-settings | Typed request/response schemas; config from `.env` |
| Background jobs | arq (Redis) | Our standard; used for sync, port polling and carrier calls that should not block requests |
| Scheduler | APScheduler, started in the FastAPI lifespan | Enqueues the periodic sync and port-polling jobs |
| Cache / broker | Redis | Job queue, sessions, rate-limit and lock keys |
| HTTP client | httpx (async) | BulkVS REST and Bandwidth calls |
| SOAP client | zeep, using its async transport over httpx | Peerless (WSDL provided) |
| XML | lxml | Bandwidth XML requests and responses |
| Retries | tenacity | Backoff on carrier timeouts and rate limits |
| Auth | python-jose (JWT) + passlib/bcrypt, as in our standard | Staff-only login |
| Encryption | cryptography (Fernet or AES-GCM) | Port PINs and account numbers at rest |
| Export | openpyxl (Excel), `csv` (CSV) | Number list export |
| Tests | pytest + pytest-asyncio (`asyncio_mode = "auto"`) | Our standard |
| Type checking / lint | pyright, ruff | Our standard |
| Package manager | uv | Our standard |
| Packaging | Docker + docker-compose | One command to run everything |

## 2. Project Structure

Follow the RIVA layout: one folder per domain module, each with the same five files.

```text
app/
├── main.py                    # app, lifespan, middleware, exception handlers, router mount
├── api/v1/
│   ├── router.py              # aggregates module routers, mounted at /api/v1
│   └── modules/
│       ├── auth/              # login, logout, refresh, me
│       ├── numbers/           # list, search, detail, export, assign/move/release, notes
│       ├── customers/         # customer CRUD and customer page
│       ├── history/           # history by number and by customer (read-only)
│       ├── purchasing/        # search available numbers, buy, disconnect
│       ├── ports/             # portability check, create, submit, track, resubmit, cancel
│       ├── carriers/          # carrier settings (admin), sync status, "sync now"
│       └── webhooks/          # carrier callbacks
├── carriers/                  # carrier adapters (see section 4)
│   ├── base.py                # CarrierAdapter interface and shared types
│   ├── registry.py            # maps carrier code to adapter
│   ├── peerless/
│   ├── bulkvs/
│   └── bandwidth/
├── core/                      # config, security, dependencies, exceptions, middleware, redis, arq pool
├── db/                        # base, session
├── models/                    # SQLAlchemy models, one file per table
├── services/                  # cross-module services (sync, export, encryption)
└── workers/                   # arq jobs and worker settings
```

Each module folder has:

| File | Responsibility |
| --- | --- |
| `router.py` | HTTP routes only. No business logic |
| `schema.py` | Pydantic request and response models |
| `service.py` | Business rules. Calls repositories and adapters |
| `repository.py` | Database queries only |
| `dependencies.py` | FastAPI dependency wiring (`get_db`, then repositories, then service) |

Layering rule: **router calls service, service calls repository and adapters. Routers never touch the database, and repositories never contain business rules.**

## 3. Backend Framework Requirements

### 3.1 API

- REST, JSON, versioned under `/api/v1`.
- Typed request and response schemas on every route. No raw dictionaries.
- Consistent error format: HTTP status, a stable error code, and a readable message. Carrier errors are translated into this format.
- Pagination on every list (`limit`, `offset` or cursor), with a sensible maximum page size.
- Sorting and filtering via query parameters (carrier, customer, status, search text, unassigned only).
- OpenAPI docs enabled in development.
- `GET /health` for liveness, and a readiness check that tests the database and Redis.

### 3.2 Core endpoints

| Area | Endpoints |
| --- | --- |
| Numbers | `GET /numbers` (search, filter, page), `GET /numbers/lookup?q=` (any number, fast), `GET /numbers/{id}`, `GET /numbers/export?format=csv\|xlsx` |
| Assignment | `POST /numbers/assign` (one or many), `POST /numbers/move`, `POST /numbers/release`, `POST /numbers/{id}/notes` |
| Customers | `GET/POST /customers`, `GET/PUT /customers/{id}`, `POST /customers/{id}/deactivate`, `GET /customers/{id}/numbers` |
| History | `GET /numbers/{id}/history`, `GET /customers/{id}/history`. **No write, update or delete routes exist.** |
| Purchasing | `GET /purchasing/search`, `POST /purchasing/order`, `POST /numbers/{id}/disconnect` |
| Ports | `POST /ports/check`, `POST /ports`, `PUT /ports/{id}`, `POST /ports/{id}/submit`, `POST /ports/{id}/cancel`, `GET /ports` (open ports), `GET /ports/{id}` |
| Unused numbers | `GET /numbers?unassigned=true&status=active` |
| Carriers (admin) | `GET /carriers`, `POST /carriers/{code}/sync`, `GET /carriers/{code}/sync-runs` |
| Webhooks | `POST /webhooks/{carrier_code}` |
| Auth | `POST /auth/login`, `/auth/logout`, `/auth/refresh`, `GET /auth/me` |

### 3.3 Async and concurrency

- All route handlers, repositories and HTTP calls are `async`.
- Blocking libraries (such as a sync SOAP call) run in a thread pool or a worker, never directly on the event loop.
- Use database transactions per request. A service method that changes data and writes history uses **one transaction**.
- Use row locks or optimistic checks when assigning or moving numbers, so two staff members cannot assign the same number at the same time.

### 3.4 Background jobs (arq)

| Job | Trigger | What it does |
| --- | --- | --- |
| `sync_carrier` | Schedule (every 15 to 60 min), manual, or webhook | Runs a carrier sync and records a `sync_run` |
| `poll_open_ports` | Schedule (every 15 to 30 min) | Updates every open port from its carrier |
| `process_webhook` | On receipt | Processes a stored webhook event idempotently |
| `import_carrier_numbers` | Manual | First import for a carrier |

Requirements:

- Jobs are idempotent and safe to retry.
- Only one sync per carrier runs at a time (Redis lock).
- A failing carrier does not stop other carriers.
- Failures are logged and visible on the sync status screen.

### 3.5 Security

- Staff sign-in is required on every route except `/health` and carrier webhooks.
- Webhooks are verified (shared secret, signature or IP allow list, depending on what the carrier supports) and stored raw before processing.
- Roles (`viewer`, `operator`, `admin`) are enforced in dependencies, not in the UI alone.
- Carrier credentials are loaded from environment variables or a secrets manager on the server only. No response, log line or error message may contain a secret.
- Port PIN and account number are encrypted at rest and excluded from logs and responses for the `viewer` role.
- HTTPS only outside local development. CORS limited to the frontend origin.
- Rate-limit the login route.
- Dependencies pinned in `uv.lock`; run a vulnerability check in CI.

### 3.6 Configuration

- All settings come from `.env` via pydantic-settings, with a committed `.env.example`.
- Separate settings for test (sandbox) and production carrier endpoints.
- No secrets in git.

### 3.7 Logging and observability

- Structured logs with a request ID.
- Log each carrier call: carrier, operation, duration, result. Never log secrets or PINs.
- A sync status screen or endpoint shows last run, result and counts per carrier.
- Alert (email or log marker) when a sync fails or has not succeeded within a set time.

### 3.8 Testing

| Test type | Scope |
| --- | --- |
| Unit | Services and status mapping, using fake repositories and the fake adapter |
| Adapter | Each adapter against recorded carrier responses (SOAP, JSON, XML samples) |
| Contract | One shared suite that every adapter must pass |
| Repository / integration | Real PostgreSQL (test database), including the history immutability test |
| API | Route tests with the test client for each feature |
| Smoke | Per carrier, against the sandbox |

Must-have tests:

- Editing or deleting a history row fails, even as the admin database role.
- Assign, move and release each write exactly one history event per number.
- Sync never changes `customer_id`.
- A retried purchase cannot buy a number twice.
- Credentials never appear in any API response.

## 4. Carrier Adapter Framework Requirements

The adapter layer is what makes the app carrier-independent.

- `carriers/base.py` defines the `CarrierAdapter` abstract class with the operations in `integration-requirements.md`, Step 2, using internal types only.
- Each adapter declares **capabilities** (for example `search_price`, `webhooks`, `async_orders`). The services and the UI read these flags and never check carrier names.
- `carriers/registry.py` maps a carrier `code` to its adapter. Adding a carrier means adding a folder and one registry entry.
- Adapters never touch the database. They take and return plain objects. Services do the saving.
- Each adapter has its own status-mapping module with tests.
- Shared helpers cover HTTP retry, timeouts, and error translation to the app's error format.
- A `FakeAdapter` ships with the project for tests and local development without carrier access.

| Carrier | Client | Notes |
| --- | --- | --- |
| Peerless | zeep with async transport, built from the provided WSDL files | Translate SOAP faults to app errors; confirm paging and async behavior |
| BulkVS | httpx REST client | Check for webhooks and price in search |
| Bandwidth | httpx plus lxml for XML | Orders are likely asynchronous: poll or use callbacks; site and location settings may be needed |

## 5. Database Requirements

### 5.1 General

- PostgreSQL 16 or newer.
- UTF-8, all timestamps stored as UTC (`timestamptz`).
- UUID primary keys for main tables. Use `bigint` identity for high-volume append-only tables if preferred.
- Foreign keys everywhere, with explicit `ON DELETE` behavior. Default to `RESTRICT`, since nothing is hard-deleted.
- `NOT NULL` by default; nullable only where `data-model-requirements.md` says so.
- Enums as PostgreSQL enums or constrained text with a check constraint. Pick one approach and use it throughout.
- `created_at` and `updated_at` on mutable tables.
- A separate test database. Tests never run against development or production data.

### 5.2 Immutable history (critical)

Implemented in Alembic migrations, not in application code alone:

1. The application connects as a role (`app_user`) that has `INSERT` and `SELECT` only on `number_events` and `port_status_history`. It has no `UPDATE`, `DELETE` or `TRUNCATE`.
2. A trigger on each history table raises an exception on `UPDATE` and `DELETE`, so even a privileged role gets blocked unless it deliberately drops the trigger.
3. Migrations run under a separate owner role. The application never uses the owner role.
4. History inserts happen in the same transaction as the change they describe.
5. A test proves points 1 and 2.
6. Back up the history tables as part of the regular backup, and test restores.

### 5.3 Indexing and search

- Unique index on (`carrier_id`, `e164`).
- Index on `numbers.e164` for exact and prefix lookup, and a trigram index (`pg_trgm`) for partial matches such as "5551".
- Indexes on `numbers.customer_id`, `numbers.status` and `numbers.carrier_id`.
- Trigram or case-insensitive index on `customers.name`.
- History indexes on (`number_id`, `occurred_at`) and (`customer_id`, `occurred_at`).
- A partial index for the unused-numbers view: `numbers (carrier_id) WHERE customer_id IS NULL AND status = 'active'`.
- A partial index for open ports on `port_requests (status)` excluding `completed` and `cancelled`.

### 5.4 Performance targets (POC)

| Operation | Target |
| --- | --- |
| Look up a number | Under 1 second |
| List and filter page | Under 1 second for 50,000 numbers |
| Customer page | Under 1 second |
| History for one number | Under 1 second |
| CSV export | Streams; no full load into memory |

### 5.5 Migrations

- Alembic only. Never change the schema by hand.
- One migration per change, numbered and reviewed, in the style of the RIVA project (`0001_...`, `0002_...`).
- Every migration has a working downgrade, except for the history immutability migration, which should document why.
- Migrations include triggers, extensions (`pg_trgm`) and roles' grants.
- CI checks that a fresh database can be built from migrations alone.

### 5.6 Seed data

- Seed the three carriers (`peerless`, `bulkvs`, `bandwidth`) with their adapter names and capabilities.
- Seed one admin user from environment variables.
- Optional demo data (fake customers and numbers) loaded only in development, using the `FakeAdapter`.

### 5.7 Backup and recovery

- Daily automated backups, with point-in-time recovery if the host supports it.
- Test a restore at least once before the demo.
- Document the process in the run notes.

### 5.8 Sizing assumptions

Confirm with DataPhone, but plan for:

- Numbers: tens of thousands (low hundreds of thousands at most).
- History: a few events per number per year, plus carrier-sync changes.
- Users: a small staff team (fewer than 50).

This is comfortably within a single PostgreSQL instance, so no sharding, caching layer or search engine is needed for the POC.

## 6. Deployment and Environment

| Item | Requirement |
| --- | --- |
| Containers | `app` (FastAPI), `worker` (arq), `db` (PostgreSQL 16), `redis` |
| Local run | `docker compose up` brings up everything; `.env.example` documents all settings |
| Environments | Local, staging (sandbox carrier accounts), production (real accounts, for the demo) |
| Outbound IP | Fixed outbound IP if any carrier uses an allow list |
| Secrets | Environment variables or a secrets manager. Never committed |
| CI | Lint (ruff), type check (pyright), tests, migration check |

## 7. Documentation Deliverables

- `README.md`: how to run the app, tests and migrations.
- `.env.example`: every setting, with comments.
- "How to add a new carrier": a short guide covering the interface, status mapping, registry, credentials and tests.
- Auto-generated API docs (OpenAPI).

## 8. Alternatives Considered

| Option | Why not chosen |
| --- | --- |
| Django | Capable, but the team standard is FastAPI, and the async carrier calls fit FastAPI better |
| Node.js / NestJS | Would break the team standard; weaker SOAP tooling than Python |
| MySQL | Weaker support for the triggers, JSON and trigram search used here |
| MongoDB | History and ownership need strict transactions and relations |
| Celery | arq is already our standard and is async-native |
| Separate search engine (Elasticsearch) | Not needed at this size; PostgreSQL trigram indexes are enough |

## 9. Open Questions

1. Where will the app be hosted (cloud provider, or on-premises), and who manages the database and backups?
2. Do we have a fixed outbound IP for carrier allow lists?
3. Is single sign-on (for example, Google or Microsoft) required, or is email and password enough?
4. How many numbers does DataPhone hold in total, and how many users will there be?
5. Must the POC support Python 3.12, or is the hosting limited to another version?
6. Who owns production credentials during the demo, and how are they delivered?
