# Integration Requirements

This document explains, step by step, how the POC connects to the carriers. Carrier-specific details are in `provider-requirements.md`. Business requirements are in `requirements.md`.

## 1. Principles

1. **One internal model, many carriers.** The app only talks to its own model (Number, Customer, Port, History event). Carrier differences stay inside adapters.
2. **One adapter per carrier.** Each adapter implements the same interface. Adding a fourth carrier means writing one new adapter and registering it, with no screen changes.
3. **The carrier is the source of truth for what we hold. Our database is the source of truth for who owns it.** Sync never overwrites customer assignment.
4. **Every change is written to history**, including changes found by sync.
5. **Credentials are never exposed** to the browser or to regular users.

## 2. Step-by-Step Integration Plan

### Step 1: Define the internal model

Create these entities before writing any carrier code:

| Entity | Key fields |
| --- | --- |
| Carrier | id, name, adapter type, enabled, last sync time |
| Number | E.164 number, carrier, customer (nullable), status, acquired date, city, state, carrier reference ID |
| Customer | id, name, notes |
| NumberEvent (history) | number, event type, field changed, old value, new value, actor (user or `carrier-sync`), timestamp |
| PortRequest | customer, numbers, carrier, status, FOC date, rejection reason, subscriber details |
| SyncRun | carrier, start/end time, result, counts, errors |

**Done when:** the schema exists, and the history table is append-only (see Step 8).

### Step 2: Define the adapter interface

Every carrier adapter must provide the same operations:

| Operation | Purpose |
| --- | --- |
| `list_numbers()` | Return all numbers held at the carrier |
| `get_number(tn)` | Return one number's current details |
| `search_available(area_code, city_state, pattern)` | Find numbers to buy, with price if available |
| `order_numbers(tns)` | Buy numbers |
| `disconnect_number(tn)` | Release a number so we stop paying |
| `check_portability(tns)` | Check if numbers can be ported in |
| `create_port(request)` / `submit_port(id)` | Start and send a port-in |
| `get_port_status(id)` | Return status, FOC date, rejection reason |
| `update_port(id, changes)` | Fix and resubmit a rejected port |
| `cancel_port(id)` | Cancel a port |
| `handle_webhook(payload)` | Optional: turn a carrier callback into internal events |

Rules for the interface:

- Inputs and outputs use **internal types only** (E.164 numbers, internal status values).
- An adapter declares what it supports (for example, "no price in search", "no webhooks"). The UI reads these flags rather than checking carrier names.
- Unsupported operations raise a clear "not supported by this carrier" error.

**Done when:** a fake in-memory adapter passes the full test suite for the interface.

### Step 3: Set up credentials and configuration

1. Store each carrier's credentials in environment variables or a secrets manager. Never put them in code or in the database in plain text.
2. Load credentials on the server only. No API response may include them.
3. Restrict the "carrier settings" screen to admins, and show secrets masked.
4. Keep separate sandbox and production configurations.

**Done when:** a regular user cannot see a credential from the UI, API or logs.

### Step 4: Build a carrier adapter (repeat for each carrier)

For each carrier, in this order:

1. Make one authenticated read call and confirm it works.
2. Implement `list_numbers()` and `get_number()`.
3. Map the carrier's fields and statuses to internal ones (see the mapping table in Step 5).
4. Implement `search_available()` and `order_numbers()`.
5. Implement `disconnect_number()`.
6. Implement the port operations.
7. Add error handling: timeouts, retries with backoff, rate-limit handling, and readable error messages.
8. Add tests using recorded sample responses, and run a smoke test against the sandbox.

Build order: **Peerless first** (largest volume), then **BulkVS**, then **Bandwidth**. If a carrier's sandbox is delayed, continue with the others.

**Done when:** each adapter passes the shared interface tests and a sandbox smoke test.

### Step 5: Normalize data

Every adapter converts carrier data to the internal format.

| Item | Internal rule |
| --- | --- |
| Phone number | E.164 (for example `+17185551000`) |
| Number status | `active`, `pending`, `porting`, `disconnected`, `unknown` (extend if needed) |
| Port status | `draft`, `submitted`, `date_confirmed`, `completed`, `rejected`, `cancelled` |
| Dates | UTC, ISO 8601 |
| City / state | Taken from the carrier if provided, otherwise from the number's area code |
| Carrier reference | Stored as is, so we can call the carrier again |

Keep the carrier's raw response alongside each synced record for debugging and audit.

**Done when:** a mapping test exists for every carrier status, and unknown statuses map to `unknown` and raise an alert instead of failing.

### Step 6: Initial import

1. Run `list_numbers()` for each carrier.
2. Insert each number with `customer = none`.
3. Write a history event ("imported from carrier") for each.
4. Import customers via CSV or manual entry, then assign numbers (see `requirements.md`, Feature 2).
5. Show an import report: total per carrier, and any numbers that failed.

**Done when:** counts in the app match counts in each carrier portal.

### Step 7: Keep the list in sync

Use two methods together:

1. **Scheduled sync (required for all carriers).**
   - Run `list_numbers()` for each carrier on a schedule (start with every 15 to 60 minutes, then tune).
   - Compare to our database:
     - **New at carrier:** add the number, unassigned, with a history event.
     - **Changed at carrier:** update status or details, with a history event whose actor is `carrier-sync`.
     - **Missing at carrier:** mark as `disconnected` or `missing`, never delete, and flag for review.
   - Never change the customer assignment.
   - Record every run in `SyncRun`.
2. **Webhooks or callbacks (where the carrier supports them).**
   - Receive carrier events (port status changes, order completion).
   - Verify the sender, store the raw event, and process it idempotently.
   - Scheduled sync stays on as a safety net in case a webhook is missed.

Also:

- A "Sync now" button for admins.
- A visible "last synced" time per carrier.
- An alert when a sync fails or hasn't succeeded for a set time.
- One carrier failing must not stop the others.

**Done when:** a change made in a carrier portal appears in the app within one sync interval, with a history event.

### Step 8: Keep history trustworthy

1. History is **append-only**: the database user the app uses has insert and select rights only, with no update or delete.
2. Add database triggers that reject updates and deletes on the history table, even for admins.
3. Write the history event in the **same transaction** as the change it describes, so neither can exist without the other.
4. Each event stores: number, event type, old value, new value, actor, timestamp, and (for sync events) the sync run ID.
5. Corrections are new events, never edits.
6. Back up the history table separately.

**Done when:** an attempt to edit or delete a history row fails, and a test proves it.

### Step 9: Wire up the features

Connect each feature to the adapter interface, not to a specific carrier:

| Feature | Adapter operations used |
| --- | --- |
| Number list and search | Local database (kept fresh by sync) |
| Assign / move / release | Local database only, plus history |
| Buy numbers | `search_available`, `order_numbers`, then assign to the chosen customer in the same flow |
| Disconnect | `disconnect_number` |
| Port-in | `check_portability`, `create_port`, `submit_port`, `get_port_status`, `update_port`, `cancel_port` |

For buying and porting:

1. Show the user a confirmation before any action that costs money or can't be undone.
2. Record the intent first, call the carrier, then record the result. If the carrier call fails, show the error and leave the number unchanged.
3. Use idempotency keys or a duplicate check so a retry can't buy twice.
4. After a successful buy, add the number to the list assigned to the chosen customer and write a history event.
5. When a port completes, assign the numbers to the port's customer and write a history event.

**Done when:** every feature in `requirements.md` works the same way for all three carriers.

### Step 10: Port tracking

1. Poll `get_port_status` for every open port on a schedule, and process webhooks where available.
2. Map carrier statuses to the six internal statuses.
3. Save the FOC date when it is received.
4. On rejection, save the reason and allow edit and resubmit.
5. On completion, run the assignment from Step 9.
6. Show all open ports in one list.

**Done when:** a test port moves from Draft to Completed in the app without visiting the carrier portal.

### Step 11: Add a new carrier later

1. Create a new adapter class that implements the interface from Step 2.
2. Add its status mappings (Step 5).
3. Register it in the adapter registry and add its credentials (Step 3).
4. Add the carrier record, run the initial import (Step 6), and enable sync (Step 7).
5. Run the shared interface test suite against it.

No changes to screens, history or the database schema should be needed.

### Step 12: Test and demo

1. Unit tests for each adapter using recorded responses.
2. Shared contract tests that every adapter must pass.
3. Sandbox smoke tests per carrier.
4. An end-to-end run of every success criterion in `requirements.md`.
5. A demo script for the live demo with real accounts.

## 3. Non-Functional Requirements

| Area | Requirement |
| --- | --- |
| Speed | Number lookup returns in seconds (served from the local database) |
| Reliability | Retries with backoff; one carrier's outage doesn't affect the others |
| Security | Secrets server-side only; staff login required; admin-only carrier settings; HTTPS everywhere |
| Auditability | Append-only history; sync runs logged |
| Observability | Logs per carrier call (without secrets), sync status page, failure alerts |
| Rate limits | Respect each carrier's limits; queue and throttle bulk calls |
| Extensibility | New carrier = new adapter only |

## 4. Open Items

See `provider-requirements.md` for each carrier's open questions. Resolve them before starting Step 4 for that carrier.
