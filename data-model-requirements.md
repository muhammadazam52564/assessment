# Data Model Requirements

This document defines the data the POC stores. It supports `requirements.md` (what the team needs) and `integration-requirements.md` (how carriers connect). Column types are written database-neutral and map directly to PostgreSQL (the recommended database).

## 1. Principles

1. **One row per number.** A number belongs to exactly one carrier and at most one customer at a time.
2. **Current state and history are separate.** The `numbers` table holds the latest state. The `number_events` table holds everything that ever happened.
3. **History is append-only.** No updates or deletes, enforced by the database, not just the app.
4. **Sync never touches ownership.** Carrier sync updates carrier-side facts only. Only a person (or a completed port or purchase) changes the customer.
5. **Nothing is hard-deleted.** Numbers, customers and carriers are marked inactive or disconnected, so history stays meaningful.
6. **Carrier differences stay out of the core tables.** Raw carrier data goes in a JSON column. Core columns use internal values only.
7. **Every row records who and when** (created and updated timestamps, actor where relevant).

## 2. Entity Overview

```text
carriers ──< numbers >── customers
                │            │
                │            └──< port_requests >──< port_request_numbers >── numbers
                │
                └──< number_events (append-only)

carriers ──< sync_runs
users ──< number_events (as actor)
```

| Table | Purpose |
| --- | --- |
| `carriers` | The carriers we buy from |
| `customers` | Customers who own numbers |
| `numbers` | Every number, with its current state |
| `number_events` | Append-only history of everything that happens |
| `port_requests` | Port-in requests |
| `port_request_numbers` | Numbers included in each port request |
| `sync_runs` | One row per carrier sync attempt |
| `users` | DataPhone staff who sign in |
| `webhook_events` | Raw carrier callbacks, stored before processing |

## 3. Tables

### 3.1 `carriers`

| Column | Type | Rules |
| --- | --- | --- |
| `id` | UUID, PK | |
| `code` | text, unique | Short key, for example `peerless`, `bulkvs`, `bandwidth` |
| `name` | text | Display name |
| `adapter` | text | Which adapter code handles this carrier |
| `enabled` | boolean | Default true. Disabled carriers are skipped by sync |
| `capabilities` | JSON | Flags such as `search_price`, `webhooks`, `async_orders` |
| `last_synced_at` | timestamp, nullable | Last successful sync |
| `created_at`, `updated_at` | timestamp | |

Credentials are **not** stored here. They live in the secrets manager or environment variables (see `integration-requirements.md`, Step 3). This table may hold a non-secret `credentials_ref` that names the secret.

### 3.2 `customers`

| Column | Type | Rules |
| --- | --- | --- |
| `id` | UUID, PK | |
| `name` | text | Required. Unique among active customers (case-insensitive) |
| `external_ref` | text, nullable, unique | Reserved for the future Rev.io customer ID |
| `notes` | text, nullable | |
| `active` | boolean | Default true. Customers are deactivated, never deleted |
| `created_at`, `updated_at` | timestamp | |

### 3.3 `numbers`

| Column | Type | Rules |
| --- | --- | --- |
| `id` | UUID, PK | |
| `e164` | text | Required, format `+1XXXXXXXXXX`. Unique together with `carrier_id` |
| `carrier_id` | FK to `carriers` | Required |
| `customer_id` | FK to `customers`, nullable | Null means unassigned |
| `status` | enum | `active`, `pending`, `porting`, `disconnected`, `missing`, `unknown` |
| `acquired_at` | timestamp, nullable | Date we got the number, from the carrier or the purchase/port date |
| `city` | text, nullable | |
| `state` | text, nullable | Two-letter code |
| `carrier_ref` | text, nullable | The carrier's own ID for the number, if it has one |
| `carrier_data` | JSON, nullable | Latest raw carrier record, for debugging |
| `notes` | text, nullable | Current note, if any. Every note is also written to history |
| `last_seen_at` | timestamp, nullable | Last time a sync saw it at the carrier |
| `created_at`, `updated_at` | timestamp | |

Rules:

- A number that moves to a different carrier (for example, after a port out or a re-buy) gets a new row. The old row stays as `disconnected`, so history is preserved.
- `customer_id` is changed only by user actions, a completed purchase or a completed port.
- Sync may change `status`, `city`, `state`, `acquired_at`, `carrier_ref`, `carrier_data` and `last_seen_at`.
- "Unused but paid for" means `status` is `active`, `customer_id` is null, and the carrier is enabled.

Indexes:

- Unique on (`carrier_id`, `e164`)
- On `e164` (fast lookup by number, including partial match using a trigram or prefix index)
- On `customer_id`
- On `status`
- On `carrier_id`

### 3.4 `number_events` (history)

| Column | Type | Rules |
| --- | --- | --- |
| `id` | bigint or UUID, PK | Never reused |
| `number_id` | FK to `numbers` | Required |
| `customer_id` | FK to `customers`, nullable | Customer the event is about, so customer history is easy to query |
| `event_type` | enum | See list below |
| `field` | text, nullable | The field that changed, for example `customer_id`, `status` |
| `old_value` | text, nullable | Value before |
| `new_value` | text, nullable | Value after |
| `actor_type` | enum | `user` or `carrier_sync` (or `system` for scheduled jobs) |
| `actor_user_id` | FK to `users`, nullable | Set when `actor_type` is `user` |
| `sync_run_id` | FK to `sync_runs`, nullable | Set for sync-created events |
| `port_request_id` | FK to `port_requests`, nullable | Set for port-created events |
| `note` | text, nullable | Free text for note events or context |
| `metadata` | JSON, nullable | Extra context such as order ID or price |
| `occurred_at` | timestamp | Required. Set by the database |

Event types:

| Event type | When it is written |
| --- | --- |
| `imported` | Number first found by the initial import or sync |
| `bought` | Number purchased through the app |
| `ported_in` | Port-in completed |
| `assigned` | Customer set on a previously unassigned number |
| `moved` | Customer changed from one customer to another |
| `released` | Customer removed from a number (taken back) |
| `disconnected` | Number released at the carrier |
| `note_added` | A note was added |
| `carrier_changed` | The carrier changed something on its side (status, location, and so on) |
| `missing_at_carrier` | A sync no longer finds the number |
| `reappeared_at_carrier` | A number marked missing is found again |

Rules:

- **Append-only.** The app's database role has `INSERT` and `SELECT` only. A database trigger rejects `UPDATE` and `DELETE` for everyone, including admins.
- **Written in the same transaction** as the change it describes.
- **Corrections are new events.** A mistake is fixed by adding an event, not editing one.
- `old_value` and `new_value` store readable values (for example, customer names or IDs together with a snapshot name in `metadata`), so history stays understandable even if a customer is later renamed.
- Bulk actions write one event per number.

Indexes:

- On (`number_id`, `occurred_at`)
- On (`customer_id`, `occurred_at`)
- On `event_type`

### 3.5 `port_requests`

| Column | Type | Rules |
| --- | --- | --- |
| `id` | UUID, PK | |
| `customer_id` | FK to `customers` | Required. The numbers go to this customer on completion |
| `carrier_id` | FK to `carriers` | The carrier chosen for the port |
| `status` | enum | `draft`, `submitted`, `date_confirmed`, `completed`, `rejected`, `cancelled` |
| `account_name` | text | Name on the current provider's account |
| `service_address` | text (or structured fields) | Street, city, state, ZIP |
| `current_provider` | text | |
| `account_number` | text | Sensitive. Restrict access and mask in the UI where possible |
| `pin` | text | Sensitive. See security rules below |
| `carrier_port_ref` | text, nullable | The carrier's ID for this port |
| `foc_date` | date, nullable | Confirmed port date, once given |
| `rejection_reason` | text, nullable | Latest rejection reason from the carrier |
| `submitted_at`, `completed_at` | timestamp, nullable | |
| `created_by` | FK to `users` | |
| `created_at`, `updated_at` | timestamp | |

Status rules:

- Allowed moves: `draft` to `submitted` or `cancelled`; `submitted` to `date_confirmed`, `rejected` or `cancelled`; `date_confirmed` to `completed`, `rejected` or `cancelled`; `rejected` to `submitted` (after a fix) or `cancelled`.
- `completed` and `cancelled` are final.
- Every status change is also stored as a row in `port_status_history` (below), so port progress is never lost.

Sensitive data:

- Encrypt `account_number` and `pin` at rest.
- Do not write them to logs.
- Consider clearing the PIN once the port is completed or cancelled.

### 3.6 `port_request_numbers`

| Column | Type | Rules |
| --- | --- | --- |
| `id` | UUID, PK | |
| `port_request_id` | FK to `port_requests` | |
| `e164` | text | Number being ported |
| `portable` | boolean, nullable | Result of the portability check |
| `portability_note` | text, nullable | Reason if not portable |
| `number_id` | FK to `numbers`, nullable | Set when the port completes and the number is created |

Unique on (`port_request_id`, `e164`).

### 3.7 `port_status_history`

Append-only, with the same protections as `number_events`.

| Column | Type | Rules |
| --- | --- | --- |
| `id` | bigint or UUID, PK | |
| `port_request_id` | FK to `port_requests` | |
| `old_status`, `new_status` | enum | |
| `foc_date` | date, nullable | FOC date at the time |
| `rejection_reason` | text, nullable | Reason at the time |
| `actor_type` | enum | `user` or `carrier_sync` |
| `actor_user_id` | FK to `users`, nullable | |
| `occurred_at` | timestamp | |

### 3.8 `sync_runs`

| Column | Type | Rules |
| --- | --- | --- |
| `id` | UUID, PK | |
| `carrier_id` | FK to `carriers` | |
| `started_at`, `finished_at` | timestamp | |
| `status` | enum | `running`, `success`, `partial`, `failed` |
| `trigger` | enum | `schedule`, `manual`, `webhook` |
| `numbers_seen`, `numbers_added`, `numbers_changed`, `numbers_missing` | integer | Counts |
| `error` | text, nullable | Short error message |
| `triggered_by` | FK to `users`, nullable | For manual runs |

### 3.9 `webhook_events`

| Column | Type | Rules |
| --- | --- | --- |
| `id` | UUID, PK | |
| `carrier_id` | FK to `carriers` | |
| `received_at` | timestamp | |
| `payload` | JSON or text | Raw body, stored before any processing |
| `processed_at` | timestamp, nullable | |
| `result` | enum | `pending`, `processed`, `ignored`, `failed` |
| `dedupe_key` | text, nullable, unique per carrier | Makes processing idempotent |

### 3.10 `users`

| Column | Type | Rules |
| --- | --- | --- |
| `id` | UUID, PK | |
| `email` | text, unique | |
| `name` | text | |
| `role` | enum | `viewer`, `operator`, `admin` |
| `active` | boolean | Deactivated, never deleted |
| `created_at`, `updated_at` | timestamp | |

Roles:

| Role | Can do |
| --- | --- |
| `viewer` | See numbers, customers, history and ports |
| `operator` | Everything a viewer can, plus assign, move, release, buy, disconnect and manage ports |
| `admin` | Everything an operator can, plus manage users, carriers and sync. Admins still cannot edit or delete history |

## 4. Status Values

### Number status

| Value | Meaning |
| --- | --- |
| `active` | In service at the carrier |
| `pending` | Ordered, not yet active |
| `porting` | Part of an open port-in |
| `disconnected` | Released at the carrier |
| `missing` | Previously seen, now not found at the carrier. Needs review |
| `unknown` | Carrier status we do not recognize. Needs review |

### Port status

`draft`, `submitted`, `date_confirmed`, `completed`, `rejected`, `cancelled` (as defined in `requirements.md`, Feature 5).

## 5. Key Queries the Model Must Support

| Need | How the model supports it |
| --- | --- |
| Look up a number in seconds | Index on `numbers.e164`, joined to carrier and customer |
| Customer page with all their numbers | Index on `numbers.customer_id` |
| History of one number | `number_events` by `number_id`, ordered by `occurred_at` |
| History of one customer | `number_events` by `customer_id`, ordered by `occurred_at` |
| Unused numbers we pay for | `status = active` and `customer_id is null` |
| Filter by carrier, customer or status | Indexes on each column |
| Search by partial number or customer name | Trigram or prefix index on `e164`, and a search on `customers.name` |
| Open ports list | `port_requests` where status is not `completed` or `cancelled` |
| Export to CSV | The same query as the list, streamed |
| "Who had this number on a given date?" | Replay `number_events` for that number up to the date |

## 6. Business Rules and Constraints

1. A number has at most one customer at a time.
2. (`carrier_id`, `e164`) is unique.
3. `e164` must match the `+1` and ten-digit format (relax later if non-US numbers are needed).
4. Assigning, moving or releasing a number always writes a `number_events` row in the same transaction.
5. Moving a number from customer A to customer B writes one `moved` event with `old_value` A and `new_value` B.
6. A disconnected number cannot be assigned to a customer.
7. A completed port creates the numbers (or updates existing ones), assigns them to the port's customer, and writes `ported_in` events.
8. A completed purchase creates the numbers, assigns them to the chosen customer, and writes `bought` events.
9. Sync may mark a number `missing` but never deletes it and never clears `customer_id`.
10. Customers with assigned numbers cannot be deactivated until their numbers are released or moved.

## 7. Security and Data Protection

- Staff sign-in is required for all access.
- Sensitive port fields (`account_number`, `pin`) are encrypted at rest and hidden from the `viewer` role.
- Carrier credentials are never stored in these tables.
- History tables are protected at the database level (insert and select only, plus triggers that block update and delete).
- Take backups regularly, and back up history on its own schedule.
- Raw carrier payloads may contain customer data, so apply the same access rules to them.

## 8. Retention

- History (`number_events`, `port_status_history`) is kept **permanently**.
- Raw payloads (`webhook_events`, `numbers.carrier_data`) can be pruned after a set period (suggest 12 months), since the structured history already holds what matters.
- `sync_runs` can be pruned after a set period (suggest 12 months).

## 9. Migration and Future Changes

- Use versioned database migrations, so every schema change is tracked.
- `customers.external_ref` is reserved for matching Rev.io later, with no schema change needed.
- A new carrier needs only a new row in `carriers`. No schema changes.
- If more number types are needed later (toll-free, non-US), extend the `e164` format rule and add a `number_type` column.

## 10. Open Questions

1. Should a number that returns to a carrier after being disconnected reuse its old row, or get a new one? (This draft says new row.)
2. Do we need more than one level of customer (parent and sub-accounts)?
3. Which user roles does DataPhone need beyond viewer, operator and admin?
4. Should the PIN be cleared after a port completes?
5. How long must the raw carrier payloads be kept?
6. Are there non-US numbers or toll-free numbers in scope?
7. What identifies a user: email and password, or single sign-on (for example, Google)?
