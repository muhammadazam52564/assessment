# Technical Proposal: DataPhone Number Management POC

**Prepared for:** DataPhone
**Date:** 2026-10-12
**Scope:** Features 1 to 5 and "Must work with many carriers" from `requirements.md`, connecting Peerless, BulkVS and Bandwidth.

---

## 1. Summary

We will build a **modular monolith**: one FastAPI backend, one React single-page app, one PostgreSQL database and one Redis. Carriers are hidden behind a single **adapter interface**, so screens and business logic never know which carrier they are talking to.

Three decisions shape everything else:

1. **Our database answers every read.** Number lookup, lists, customer pages and history never call a carrier. Carriers are called only to sync, buy, disconnect and port. This is why lookups return in well under a second.
2. **Carrier is the source of truth for what we hold. We are the source of truth for who owns it.** Sync updates carrier facts and never touches `customer_id`.
3. **History is enforced by the database, not by the app.** Even an admin, or a bug in our code, cannot edit or delete a history row.

| Layer | Choice |
| --- | --- |
| Backend | Python 3.12, FastAPI (async), SQLAlchemy 2.0 async, Alembic, Pydantic v2 |
| Jobs and scheduling | arq on Redis (queue, cron schedule, retries) |
| Database | PostgreSQL 16 (transactions, triggers, JSONB, `pg_trgm`) |
| Cache, locks, throttling | Redis (the same instance arq needs) |
| Carrier clients | `zeep` (Peerless SOAP), `httpx` (BulkVS REST), `httpx` + `lxml` (Bandwidth XML), `tenacity` for retries |
| Frontend | React 19 + TypeScript + Vite, axios instance, TanStack Query for server state |
| Packaging | Docker Compose: `app`, `worker`, `db`, `redis` |

The backend and frontend follow the team's RIVA standards (folder layout, naming, auth, tooling), so any developer on the team can pick the code up.

---

## 2. How we would build it, and why

### 2.1 Modular monolith

A POC does not need five deployed services. It does need clean seams, so any module can be extracted later without a rewrite. Each domain is a folder with the same five files as RIVA:

```text
app/
├── api/v1/modules/
│   ├── auth/  numbers/  customers/  history/
│   ├── purchasing/  ports/  carriers/  webhooks/
│   │      └── router.py · schema.py · service.py · repository.py · dependencies.py
├── carriers/            # adapter interface, registry, one folder per carrier, FakeAdapter
├── workers/             # arq jobs, cron schedule, worker settings
├── models/ db/ core/    # SQLAlchemy models, session, config, security, redis
```

Rules that keep the seams real:

- **Router calls service. Service calls repository and adapters.** Routers hold no logic, repositories hold no rules.
- **A module never imports another module's repository or models.** It calls the other module's *service* (a small public interface). If `ports` needs to assign numbers on completion, it calls `NumbersService.assign_from_port(...)`, not the numbers table.
- **Modules do not share transactions implicitly.** A service receives the session as a parameter. Cross-module flows (port completes, then assign) are explicit and visible in one place.
- **Modules talk by commands and events, not by reaching into each other's data.** In the monolith these are plain function calls. If `ports` becomes its own service, the same call becomes an HTTP call or a queue message, and the caller does not change.

What is easy to extract later, and what it costs:

| Candidate | Why it extracts cleanly | What would change |
| --- | --- | --- |
| `carriers` + `workers` (integration service) | Adapters already take and return plain objects and never touch the DB | Services call it over HTTP/queue instead of in-process |
| `history` | Append-only, read-mostly, already behind its own service | Own database; writes become events |
| `ports` | Own tables and its own state machine | Replace direct calls to `numbers` with messages |

We keep **one database with per-module table ownership** for the POC. Splitting databases is the expensive step, and we do not need it yet.

### 2.2 Why this stack

| Choice | Reason |
| --- | --- |
| FastAPI async | Workload is mostly waiting on carrier APIs. Async handles that without thread juggling |
| SQLAlchemy 2.0 + Alembic | Typed models, versioned migrations. The history triggers and role grants live in migrations |
| PostgreSQL | Needs real transactions (history written with the change), triggers (immutability), `pg_trgm` (partial number search) and JSONB (raw carrier payloads) |
| arq | Async-native, already our standard, and has built-in cron, retries and unique job IDs |
| Redis | arq needs it anyway. We reuse it for locks, throttling and a small, careful cache (section 6) |
| React SPA | Internal staff tool: no SEO, speed of first load is not critical, and the team already knows React |

### 2.3 Frontend

- **React 19, TypeScript, Vite**, matching the RIVA frontend: Tailwind 4, shadcn/Radix primitives, React Router 7, react-hook-form + zod, TanStack Table.
- **One axios instance** (`lib/api.ts`) holds the base URL, `withCredentials`, a request-ID header and response interceptors. The interceptors do three jobs: refresh once on 401, translate the API's standard error format into a typed `ApiError`, and surface 403 as "you don't have permission". Feature code never calls `axios` directly. Each module has a thin `api.ts` that uses the instance.
- **Auth:** HttpOnly cookies set by the backend, as in RIVA. No tokens in `localStorage`.
- **Server state:** we recommend TanStack Query on top of axios for caching, background refetch and mutation invalidation. RIVA uses Redux Toolkit, so this is the one place we ask for a decision (see section 11, question 12). Axios stays the transport either way.
- **Screens:** Number lookup (the home screen: one search box), Numbers list with filters and export, Customer list and Customer page, Buy numbers, Port requests (open ports list plus detail), Carriers (admin: status, "Sync now", last synced).
- **Capability-driven UI.** The frontend reads the carrier's capability flags from the API (for example `search_price: false`) and adapts: it shows "price not provided by this carrier" or "cancel this port in the carrier portal". It never checks a carrier name.

---

## 3. Handling three carriers that work very differently

### 3.1 The adapter pattern

The three carriers differ in nearly every way (from `provider-api-research.md`):

| | Peerless | BulkVS | Bandwidth |
| --- | --- | --- | --- |
| Protocol | SOAP | REST/JSON | REST/XML |
| Auth | customer + passcode + user ID | HTTP Basic | OAuth 2.0 client credentials (token expires hourly) |
| List numbers | Inventory report | One call, full detail | List returns numbers only; detail is one call per number |
| Buy | Async | Synchronous | Async |
| Search by city / pattern | State + rate center / vanity digits | Neither | City yes / partial pattern |
| Price in search | Not seen | Yes | No |
| Port cancel / resubmit | Unclear | Neither exists | Both exist |
| Webhooks | None seen | None for inventory or ports | Yes, not retried |

The app holds **one internal model** and one interface:

```python
class CarrierAdapter(ABC):
    code: str
    capabilities: Capabilities            # declared flags, read by services and the UI

    async def list_numbers(self) -> AsyncIterator[CarrierNumber]: ...
    async def get_number(self, e164: str) -> CarrierNumber: ...
    async def search_available(self, q: SearchQuery) -> SearchResult: ...
    async def order_numbers(self, req: OrderRequest) -> OrderResult: ...
    async def get_order(self, ref: str) -> OrderResult: ...             # for async carriers
    async def disconnect_number(self, e164: str) -> DisconnectResult: ...
    async def check_portability(self, tns: list[str]) -> list[PortabilityResult]: ...
    async def create_port(self, req: PortRequestIn) -> PortRef: ...
    async def submit_port(self, ref: PortRef) -> PortStatus: ...
    async def get_port_status(self, ref: PortRef) -> PortStatus: ...
    async def update_port(self, ref: PortRef, changes: PortChanges) -> PortStatus: ...
    async def cancel_port(self, ref: PortRef) -> PortStatus: ...
    async def handle_webhook(self, payload: bytes, headers: dict) -> list[CarrierEvent]: ...
```

Rules:

- **Internal types only** cross the boundary: E.164 numbers, internal statuses, UTC datetimes, `Decimal` prices. Each adapter owns its own mapping module (carrier status to internal status) with a test for every known status.
- **Adapters never touch the database.** They take and return plain objects. Services do the saving.
- **Capabilities are declared, not discovered by trial and error:**

  ```python
  Capabilities(search_price=True, search_by_city=False, search_by_pattern=False,
               port_cancel=False, port_foc_date=False, webhooks=False,
               async_orders=False, inventory_detail_in_list=True, sandbox=False)
  ```

- **Unsupported operations raise `NotSupportedByCarrier`**, which the API turns into a clear message. The UI should already have hidden or explained the action from the flags.
- **Unknown carrier statuses map to `unknown`** and raise an alert. They never crash a sync.
- **Raw responses are kept** (JSONB) next to each synced record, for debugging and audit.

### 3.2 How each difference is absorbed

| Difference | Where it is handled |
| --- | --- |
| SOAP vs REST vs XML | Inside each adapter's client. Nothing above it sees SOAP, JSON or XML |
| Number formats (10 digits, 11 digits, `+1`) | Normalised to `+1XXXXXXXXXX` at the adapter boundary, with one shared `normalize_e164()` |
| Bandwidth's token | Adapter fetches and refreshes it. The token is cached in Redis, so web and worker processes share it |
| Bandwidth's per-number detail calls | Adapter streams the list, and the sync service fetches detail only for new or changed numbers, throttled |
| Async orders (Peerless, Bandwidth) | Order is saved as `pending` in a `carrier_orders` row. An arq job polls `get_order` until done, then finalises (section 5.2) |
| BulkVS has no port cancel or resubmit | Capability flag `port_cancel=false`. The UI shows "Cancel in BulkVS portal" and the app records a note event. Raised with DataPhone in section 11 |
| BulkVS has no FOC date field | Adapter returns its requested date and sets `foc_date_is_estimate=true`. UI labels it accordingly |
| Search city/pattern gaps | Search service filters on our side where the carrier cannot, and shows a per-carrier note ("BulkVS needs area code + exchange") |
| Credentials differ | Each adapter declares the secret fields it needs. Carrier row stores only a `credentials_ref` (section 5.4) |

### 3.3 Adding a fourth carrier

1. Create `app/carriers/<name>/` with a client, a status-mapping module and an adapter class implementing the interface.
2. Declare its capabilities.
3. Add one line to `registry.py`.
4. Add its credentials to the environment or secret manager.
5. Insert a `carriers` row (migration or admin screen), run the initial import, enable sync.
6. Run the shared **contract test suite** against it (see below).

No screen, history, schema or service change is needed. This is the claim we will prove in the demo by adding the `FakeAdapter` as a live fourth carrier.

**Contract tests.** One parametrised suite runs against every adapter, including `FakeAdapter`. It checks that outputs are internal types, numbers are E.164, unsupported operations raise the right error, and every mapped status is valid. A new carrier is "done" when it passes the same suite.

---

## 4. Keeping history complete and trustworthy

### 4.1 Design

- **Two tables per concept:** `numbers` holds current state, `number_events` holds everything that happened. Port progress has its own `port_status_history`.
- **One writer.** Every state change goes through a single `NumberStateService`. No code updates `numbers.customer_id` or `numbers.status` directly, so no change can skip the history write. Repositories expose no direct update for those columns.
- **Same transaction.** The change and its event commit together or not at all. A failed event rolls the change back.
- **One event per number**, including bulk actions. Moving 40 numbers writes 40 events with a shared `batch_id` in metadata.
- **Readable values.** Events store customer names as a snapshot in metadata, so history still reads correctly after a customer is renamed.
- **Corrections are new events**, never edits.
- **Actors are explicit:** a user ID, `carrier_sync` (with the sync run ID) or `system` (with the job name).
- **"Who had this number on date X?"** is answered by replaying events for that number up to the date. We include this as a query and a test.

### 4.2 Enforcement in PostgreSQL

All of this ships in Alembic migrations:

1. The app connects as `app_user`, which has `INSERT` and `SELECT` only on `number_events` and `port_status_history`. No `UPDATE`, `DELETE` or `TRUNCATE`.
2. A `BEFORE UPDATE OR DELETE` trigger and a `BEFORE TRUNCATE` trigger raise an exception on both history tables, so even a privileged role is blocked unless it deliberately drops the trigger.
3. Migrations run as a separate owner role. The app never uses it.
4. `occurred_at` defaults in the database (`now()`), so the app cannot back-date events.
5. The API has **no** write, update or delete route for history. Notes are a separate action that creates a `note_added` event.

**Stretch (if time allows):** a hash chain. Each event stores a hash of its content plus the previous event's hash for that number, and a nightly job verifies the chain. This gives tamper *evidence* even against someone with owner access.

### 4.3 Proof

These tests ship with the POC and run in CI against real PostgreSQL:

- `UPDATE` and `DELETE` on a history row fail as `app_user`, and also fail as the owner role while the trigger exists.
- Assign, move and release each write exactly one event per number.
- If the event insert fails, the assignment rolls back.
- Sync never changes `customer_id`.
- Replaying events reproduces the current owner for every number in the seed data.

---

## 5. Keeping the list in sync with the carriers

### 5.1 Two methods together

1. **Scheduled sync (required, all carriers).** An arq cron job enqueues `sync_carrier(carrier_id)` per enabled carrier, starting at every 30 minutes and tuned from real run times and carrier limits.
2. **Webhooks (where available).** Bandwidth callbacks are stored raw in `webhook_events`, verified, deduplicated, then processed by `process_webhook`. A callback only *triggers* a re-fetch of that order or port, as Bandwidth itself recommends. The scheduled poll remains the safety net, because Bandwidth does not retry failed callbacks.

### 5.2 Sync algorithm

For each carrier, in one arq job under a Redis lock `lock:sync:{carrier}` so two runs never overlap:

1. Open a `sync_runs` row (`running`).
2. Stream `list_numbers()`. Load existing numbers for that carrier into memory (tens of thousands of rows is small).
3. Diff:

   | Case | Action | Event |
   | --- | --- | --- |
   | New at carrier | Insert, unassigned | `imported` |
   | Changed (status, city, state, acquired date, reference) | Update carrier-side fields only | `carrier_changed` with old and new value, actor `carrier_sync` |
   | Missing at carrier | Mark `missing` after **two consecutive misses** | `missing_at_carrier` |
   | Reappeared | Back to `active` | `reappeared_at_carrier` |

4. Never delete. Never change `customer_id`.
5. Close the run with counts and status (`success`, `partial`, `failed`).

Safety rules that matter more than the happy path:

- **A failed or truncated listing must never mark numbers missing.** If the stream errors midway, the run is `partial` or `failed` and the missing step is skipped. Otherwise a carrier outage would flag thousands of numbers.
- **Mass-change guard.** If more than a set percentage of a carrier's numbers would go missing in one run, stop, mark the run `failed` and alert, instead of applying it.
- **One carrier failing never blocks the others.** Each carrier is its own job.
- **Bandwidth detail fan-out:** only unknown numbers, or numbers whose status changed, get a `tndetails` call, throttled by a per-carrier token bucket in Redis. First import is slow once. Later syncs are cheap.
- **Each sync batch commits in chunks** with its events, so a crash leaves a consistent state and the next run resumes by diffing again. Jobs are idempotent.

### 5.3 Orders and ports (writes to carriers)

For buying, disconnecting and porting, the order of operations is fixed:

1. **Record intent first:** insert a `carrier_orders` row (`pending`) with a client-generated **idempotency key** and the user, before calling the carrier.
2. Call the carrier. A retry with the same key finds the existing row and does not buy twice.
3. Synchronous carrier (BulkVS): on success, in one transaction create the number, assign it to the chosen customer, write the `bought` event, and mark the order `completed`.
4. Async carrier (Peerless, Bandwidth): keep the order `pending` and let `poll_pending_orders` (arq) finalise it the same way.
5. On failure: mark the order `failed` with a readable error. The number list stays unchanged.
6. **Spend guards:** explicit confirmation screen showing count and price, a configurable per-request and per-day purchase cap, and a `dry_run` mode (`Checkonly=true` on BulkVS). Carriers have no sandbox for some of these, so mistakes cost real money.

The port flow uses the same pattern, with the six-status state machine from `data-model-requirements.md` enforced in the service. `poll_open_ports` runs every 15 minutes for open ports and on every Bandwidth callback. On `completed`, one transaction assigns the numbers to the port's customer and writes `ported_in` events.

### 5.4 Credentials

- Secrets live in environment variables (or a secrets manager), read **only** by the backend and worker processes. The `carriers` table holds a non-secret `credentials_ref` (for example `BANDWIDTH`), which names a group of variables.
- Each adapter declares its required fields (`customer, passcode, userid`; `username, password`; `account_id, client_id, client_secret, site_id, location_id`).
- API responses show `configured: true/false` only. A test asserts that no response, log line or exception message contains a secret value.
- Dedicated API credentials per carrier, read-only first, ordering rights only when the demo needs them.

---

## 6. Where Redis helps (and where it does not)

Redis is already required for arq, so using it well is nearly free. We use it for what is genuinely repeated and expensive, and **never as a source of truth**.

| Use | Detail | Benefit |
| --- | --- | --- |
| arq queue and cron | Sync, polling, webhook processing, retries | Required |
| Sync and job locks | `lock:sync:{carrier}`, `lock:port:{id}` with TTL | No overlapping runs |
| Carrier throttling | Token bucket per carrier; honours `Retry-After` | Stays inside rate limits, protects the account |
| Bandwidth OAuth token | Cached until about 30 seconds before expiry | One token request per hour, not per call |
| Available-number search | Cached 60 to 120 seconds, keyed by carrier plus normalised query | Fewer paid or rate-limited carrier calls when staff refine and repeat searches. Short TTL because stock changes, and a buy always re-checks with the carrier |
| Carrier static lookups | Rate-centre trees, area code to city and state, capability flags | Slow-changing data fetched once |
| Dashboard counts | Totals per carrier, unassigned count, open ports. Invalidated when sync or assignment commits | Cheap home-screen numbers |
| Login rate limit | Sliding window per IP and email | Basic brute-force protection |

What we deliberately do **not** cache: number lookup, the numbers list, customer pages and history. At tens of thousands of rows, PostgreSQL with the right indexes meets the sub-second targets, and cached ownership data would add stale-data risk to the one thing DataPhone cares most about. We will measure first and add a cache to a read path only if a measurement says it is needed.

Cost impact: Redis lowers **carrier API calls** and rate-limit pressure, which is where the cost and failure risk sit. It is not there to protect the database.

---

## 7. Data model (summary)

We adopt `data-model-requirements.md` as written (carriers, customers, numbers, number_events, port_requests, port_request_numbers, port_status_history, sync_runs, webhook_events, users) with these additions:

| Addition | Why |
| --- | --- |
| `carrier_orders` (id, carrier, type, idempotency_key unique, status, requested_by, request/response JSON, carrier_ref, error, timestamps) | Idempotent, auditable buy/disconnect, with a place to track async orders |
| `numbers.is_estimated_location` flag | Distinguishes city/state from the carrier from a value derived from the area code |
| `number_events.batch_id` | Groups the events of one bulk action |
| `port_requests.foc_date_is_estimate` | BulkVS has no real FOC date |
| `audit_log` (append-only, same protections) for non-number actions: logins, carrier setting views, exports, sync triggers | "Proper logging and record tracking" beyond number history |

Key indexes: unique `(carrier_id, e164)`, `pg_trgm` GIN on `e164` and `customers.name`, a partial index for unused numbers (`customer_id IS NULL AND status = 'active'`), a partial index for open ports, and history indexes on `(number_id, occurred_at)` and `(customer_id, occurred_at)`.

---

## 8. Deployment

Hosting is not decided yet (open question 10), so this section gives a recommended default that works on almost any provider. Nothing in it is tied to a specific cloud.

### 8.1 Runtime layout

Four containers, defined in one `docker-compose.yml`, used for local development and for the deployed environments:

| Container | Role | Notes |
| --- | --- | --- |
| `proxy` | Caddy (or Nginx): HTTPS, serves the built React app, forwards `/api` to the backend | Automatic certificates. Same origin for app and API, so cookies work without CORS tuning |
| `app` | FastAPI under uvicorn | Stateless. Can run several copies behind the proxy |
| `worker` | arq worker with the cron schedule | Same image as `app`, different command. Scales independently |
| `redis` | Queue, locks, throttling, cache | Persistence on (AOF), so queued jobs survive a restart |
| `db` | PostgreSQL 16 | See 8.3: managed database preferred over a container |

The React app is a static build (`vite build`) copied into the proxy image. It needs no server of its own.

### 8.2 Hosting recommendation

| Option | Fit | Trade-off |
| --- | --- | --- |
| **One small VM with Docker Compose + managed PostgreSQL (recommended)** | Cheap, simple, one fixed public IP for carrier allow lists, easy to demo | Single host, so brief downtime on deploy. Acceptable for an internal POC |
| Container platform (ECS, Cloud Run, Fly.io and similar) | Better scaling and zero-downtime deploys | Fixed outbound IP needs extra setup (NAT gateway or static egress), more moving parts for a POC |
| On-premises at DataPhone | Data stays in-house | Needs their ops team, backups and a public HTTPS endpoint for Bandwidth callbacks |

Requirements that drive the choice, whatever the provider:

- **A fixed outbound IP** for the backend and worker, because Peerless (and possibly others) allow-list by IP. Only the backend talks to carriers, so only that IP matters.
- **A public HTTPS URL** for `/api/v1/webhooks/bandwidth`, so Bandwidth callbacks can reach us.
- The proxy exposes only ports 80 and 443. Database and Redis are reachable only on the private network.

If DataPhone already has a cloud account or standard, we deploy there instead. The Compose files translate directly to any container platform.

### 8.3 Database

- **Managed PostgreSQL 16** if the provider offers it (automated backups, point-in-time recovery, patching). Otherwise a Postgres container with a persistent volume and scheduled `pg_dump` plus WAL archiving.
- **Two roles:** `owner` (runs migrations only) and `app_user` (INSERT/SELECT on history, no UPDATE/DELETE/TRUNCATE). The deployed app only ever receives `app_user` credentials. This is what makes the immutable history real in production, not just in tests.
- `pg_trgm` is enabled by migration.

### 8.4 Environments

| Environment | Purpose | Carrier accounts | Data |
| --- | --- | --- | --- |
| **Local** | Development | `FakeAdapter` | Seeded demo data |
| **Staging** | Integration testing, smoke tests | Sandbox or test accounts, read-only where no sandbox exists | Imported from the test accounts, never copied from production |
| **Production** | The live demo and real use | Real accounts, dedicated credentials, spend caps on | Real |

Configuration comes from environment variables via pydantic-settings, with a committed `.env.example`. Each environment has its own carrier credentials, database and Redis. Staging and production never share secrets.

### 8.5 CI/CD

GitHub Actions, in DataPhone's repository:

1. **On every pull request:** `ruff`, `pyright`, frontend `tsc --noEmit`, backend and frontend tests against a real PostgreSQL service container (includes the history immutability tests), a migration check that builds a fresh database from migrations alone, and a dependency vulnerability scan.
2. **On merge to `main`:** build the `app` and `proxy` images, tag with the commit SHA, push to GitHub Container Registry, deploy to staging automatically.
3. **Production:** manual approval, then deploy.

Deploy steps (same for both environments):

1. Pull the new images.
2. Run migrations as a one-shot container using the `owner` role. Abort the deploy if it fails.
3. Start new `app` and `worker` containers. The worker finishes or re-queues in-flight jobs on shutdown. Jobs are idempotent, so a restart mid-job is safe.
4. Readiness check (`/health/ready`: database and Redis) must pass before the proxy sends traffic.

**Rollback:** redeploy the previous image tag. Migrations are written backward-compatible (add before use, remove later), so the old code runs against the new schema. The history immutability migration is the one exception, and it is never rolled back.

### 8.6 Secrets

- Carrier credentials, `SECRET_KEY`, the encryption key for port PINs and account numbers, and database passwords live in the host's secret store or an `.env` file readable only by the deploy user. They are never in git, images or logs.
- GitHub Actions secrets hold only deploy credentials, not carrier credentials.
- Rotation: documented steps for each carrier credential. Bandwidth supports two active secrets for zero-downtime rotation.
- The encryption key is backed up separately from the database. Losing it makes stored PINs unrecoverable.

### 8.7 Backup and recovery

| Item | Approach |
| --- | --- |
| Database | Daily automated backup with point-in-time recovery where available |
| History tables | Included in the above, plus a separate weekly export to a different storage location |
| Redis | Not a source of truth. Persistence is for convenience only. Losing it loses queued jobs, which sync and polling recreate on the next run |
| Restore test | Performed once before the demo, with the steps written in the run notes |

### 8.8 Monitoring and operations

- `GET /health` (liveness) and `GET /health/ready` (database and Redis).
- Structured JSON logs with a request ID. Each carrier call logs carrier, operation, duration and result, never secrets or PINs.
- Container logs shipped to the host's log service, or kept with rotation on a single VM.
- An uptime check on `/health`.
- **Alerts** (email or chat webhook): a sync failed or has not succeeded within twice its interval, a mass-change guard tripped, an unknown carrier status appeared, a purchase cap was hit, or a carrier returned repeated authentication errors.
- The Carriers screen shows last sync time, result and counts per carrier, so staff can see health without asking a developer.

### 8.9 Sizing and growth

For tens of thousands of numbers and fewer than 50 users, a small VM (2 vCPU, 4 GB RAM) plus a small managed database is enough. Growth path, in order of cost: raise worker concurrency, run a second `app` container, move to a container platform, split the integration service out (section 2.1). None of these need code changes beyond configuration.

---

## 9. Plan, phases and timeline

Assumption: one full-stack engineer working with AI tooling, roughly **6 weeks**. Dates move if carrier access arrives late (see risks). Each phase ends with something demonstrable.

| Phase | Weeks | Build | Done when |
| --- | --- | --- | --- |
| **0. Foundations** | 0.5 | Repo, Docker Compose, CI (ruff, pyright, pytest, migration check), auth, roles, base models, **history tables with triggers and the immutability tests**, `CarrierAdapter`, registry, `FakeAdapter`, contract tests | `docker compose up` works. History cannot be edited. Fake carrier passes contract suite |
| **1. First slice on the fake carrier** | 1 | Numbers list, search, filters, lookup, CSV/Excel export, customers, assign/move/release, notes, history views, React shell with axios instance | Features 1 to 3 demoable end to end on fake data |
| **2. Peerless** | 1.5 | Adapter from the WSDL, mapping, initial import, scheduled sync, sync status screen. Port and order operations as the WSDL allows | Peerless counts match the portal. A portal change appears in the app with a history event |
| **3. BulkVS** | 1 | Adapter, import, sync, search with price, buy (with `Checkonly` dry run), disconnect | Feature 4 working for two carriers, side by side |
| **4. Bandwidth** | 1.5 | OAuth, XML client, throttled detail fan-out, async orders, port lifecycle, callbacks | Feature 4 and 5 working on all three |
| **5. Ports and hardening** | 1 | Port UI and open-ports list, reject-fix-resubmit, unused-numbers view, alerts, spend caps, security tests, backup and restore test, documentation, demo script | Every success criterion in `requirements.md` runs on real accounts |

Order rationale: the fake carrier lets us finish and demonstrate all five features before any carrier access arrives. Peerless goes first because it has the highest volume and the highest uncertainty, so schedule risk surfaces early. BulkVS is the simplest real integration. Bandwidth is the most work (OAuth, XML, async, fan-out) and has the richest port support.

**Deliverables:** source in DataPhone's GitHub, `README.md` (run, test, migrate), `.env.example`, a "How to add a carrier" guide, OpenAPI docs, and the demo script.

---

## 10. Risks and what we would do about them

| # | Risk | Likelihood / impact | Mitigation |
| --- | --- | --- | --- |
| 1 | **Peerless is undocumented publicly.** Operation behaviour, auth transport and pricing are unverified. Largest carrier, so highest schedule risk | High / High | Start from the WSDL the day it arrives. Everything else builds on the FakeAdapter in the meantime. Send the Peerless question list on day 1. Phase 2 has buffer |
| 2 | **No sandbox** for BulkVS (confirmed) and Bandwidth (none documented). Test purchases cost real money | High / Medium | Read-only first. `Checkonly=true` for BulkVS. Spend caps and confirmation. Agree a test budget and test numbers with DataPhone |
| 3 | **Carrier feature gaps** (BulkVS no port cancel/resubmit, no FOC date, no city/pattern search; Bandwidth no price) | Certain / Medium | Capability flags, honest UI labels, our-side filtering, and agreeing the "do it in the portal and log a note" fallback with DataPhone early |
| 4 | **Bandwidth Basic auth ends 2026-12-02** | Certain / Medium | Build on OAuth from day one |
| 5 | **Sync wrongly marks numbers missing** after a partial listing or outage | Medium / High | Two-miss rule, no-missing-on-partial-run, mass-change guard, alerts |
| 6 | **Double purchase** from a retry, double click or timeout | Medium / High | Idempotency key, order record written first, confirmation step, spend cap, test for it |
| 7 | **IP allow-listing** blocks carrier calls | Medium / Medium | Ask for the outbound IP and hosting plan in week 0. Use a fixed-egress host |
| 8 | **Bandwidth callbacks lost** (not retried, auth handshake quirks) | Medium / Low | Polling is the baseline. Callbacks are an accelerator only |
| 9 | **Rate limits are unpublished** | Medium / Medium | Per-carrier token bucket, `Retry-After` handling, tenacity backoff, tune after real runs |
| 10 | **Sensitive data** (port PINs, account numbers, raw carrier payloads) | Medium / High | Encrypt at rest, hide from `viewer`, exclude from logs, prune raw payloads after 12 months, optionally clear PIN after completion |
| 11 | **Ownership data wrong at start** (nobody knows who owns what today) | High / Medium | CSV import for customers and existing assignments, a "needs review" import report, bulk-assign tools. This is a data problem, not a code problem |
| 12 | **Scope creep** (Rev.io sync, E911, port-out, SMS, customer portal) | Medium / Medium | Listed as out of scope. The `external_ref` column leaves the Rev.io door open without work now |

---

## 11. Questions we need answered before starting

**Access and carriers**

1. Can we get the Peerless WSDL files, the Animate API guide, a staging account and IP allow-listing in week 0? Is the account now on Infobip, and does that change the API?
2. Which production credentials may be used read-only for the initial import and demo? Who owns each carrier relationship and can raise API issues?
3. For Bandwidth: which **Site** and **Location** should new numbers use, and which roles can the OAuth credential have (inventory read, ordering, porting)?
4. How should we handle the BulkVS port gaps (no cancel, no resubmit, no FOC field)? Is "do it in the portal and log a note" acceptable for the POC?
5. How is price obtained for Bandwidth and Peerless? Is a rate card acceptable, or should the UI show "price not provided"?

**Money and risk**

6. For the demo: may we buy and disconnect real numbers? What is the spending limit, and are there test numbers we can safely buy and release?

**Data**

7. Roughly how many numbers per carrier, and how many users? Are toll-free or non-US numbers in scope?
8. Can we have the starting customer list (CSV) and any existing number-to-customer records?
9. When a number disconnects and later returns, should it reuse its old row or create a new one? (Draft assumes a new row.)

**Platform**

10. Where will this be hosted, who manages the database and backups, and what is the fixed outbound IP?
11. Sign-in: email and password, or single sign-on (Google or Microsoft)? Are the three roles (viewer, operator, admin) enough?
12. Frontend state: follow RIVA's Redux Toolkit exactly, or approve TanStack Query for server state? (We recommend TanStack Query. Axios is the transport in both cases.)

---

## 12. How this answers the brief

| What DataPhone asked for | Where |
| --- | --- |
| How you would build it, and why | Sections 1, 2 |
| Three different carriers, and adding a fourth | Section 3 |
| Complete, trustworthy history | Section 4 |
| Keeping the list in sync | Sections 5, 6 |
| What to build first, and a phased timeline | Section 9 |
| Deployment and operations | Section 8 |
| Risks and responses | Section 10 |
| Questions before starting | Section 11 |
