# Provider API Research: Peerless, BulkVS, Bandwidth

Research date: 2026-10-10. This document records what public sources say about each carrier's API, how to get credentials, and where the APIs differ from what the app needs. It updates the assumptions in `provider-requirements.md`.

**Confidence labels used below:**

- **Verified:** read directly from the carrier's own published spec or documentation.
- **Third-party:** taken from an unofficial source (for example, a community client).
- **Unverified:** not confirmed by any source. Must be checked with the carrier or a test account.

## 1. Summary

| | Peerless | BulkVS | Bandwidth |
| --- | --- | --- | --- |
| Public docs | **None found** | Yes, public OpenAPI spec | Yes, public guides and OpenAPI specs |
| API style | SOAP ("Animate API"), third-party evidence only | REST / JSON | REST, mostly **XML** (Numbers / Dashboard API), some JSON |
| Auth | customer + passcode + user ID (how they are sent: unverified) | HTTP Basic | **OAuth 2.0 client credentials** (Basic auth being retired) |
| Credentials come from | Account rep / support (unverified) | Self-service in the portal | Self-service in the Bandwidth App (admin only) |
| Sandbox | Staging host exists; access unverified | **None** | None documented |
| List all numbers | Yes (`GetTnInventoryReport`, third-party) | Yes, with full detail | Yes, but returns **numbers only**; detail needs one call per number |
| Search by city/state | Rate center and state, no city | **No** (area code + exchange only) | Yes |
| Search by pattern | Yes (`VanityDigits`) | **No** | Partly (`localVanity` + `endsIn`) |
| Price in search | Not seen | **Yes** (Mrc, Nrc, per-minute) | **No pricing endpoint found** |
| Buy | `PlaceOrder`, async | `POST /orderTn`, synchronous | `POST /orders`, async |
| Disconnect | `DisconnectOrder` | `DELETE /tnRecord` | `POST /disconnects`, async, 60-day aging |
| Port-in | Order tools, no explicit cancel/reject | `PUT /portTn`, **no cancel/reject/resubmit** | Full lifecycle, including cancel and resubmit |
| Webhooks | None seen | **Messaging only** | Yes, for orders, disconnects and port-ins |

## 2. Peerless Network

### 2.1 What we know

- **No public documentation, WSDL or developer portal was found.** Docs are probably given to customers by the account team (unverified).
- The company site mentions an "API bond available for seamless integration" but gives no detail (`https://www.peerlessnetwork.com/peerless-portal/`).
- Peerless appears to have been folded into Infobip (the portal links to an Infobip Voice US portal). Ask which platform your account is on.
- Everything technical below comes from an **unofficial Go client** generated from the WSDL (`https://github.com/ubogdan/peerless`, `https://pkg.go.dev/github.com/ubogdan/peerless`). It is marked work-in-progress and may be out of date. **Treat it as a hint, not a contract.**

### 2.2 Endpoints and style (third-party)

- Style: SOAP, called the "Animate API". The path suggests Apache Axis.
- Production: `https://animate.peerlessnetwork.com:8181/animateapi/axis/APIService`
- Staging: `https://aniweb02.peerlessnetwork.com:8181/animateapi/axis/APIService`
- The WSDL could not be downloaded: the production host refused the connection on port 8181. It is probably restricted by IP, which also matches the need for IP allow-listing.
- The WSDL file names `APIService.wsdl` and `APIServicePortType.wsdl` come from the assessment, not from any public source. They fit the Animate API but are unconfirmed.

### 2.3 Credentials

The client constructor takes **four values**:

| Value | Example in the client |
| --- | --- |
| Endpoint URL | Staging or production URL |
| `customer` | Customer (account) name |
| `passcode` | Secret passcode |
| `userid` | A user ID that looks like an email address |

**How to get them (unverified, inferred):**

1. Ask the Peerless account rep or support (phone 888-662-7757 per the site) for API credentials: customer, passcode and user ID.
2. Ask for the **WSDL files and the Animate API guide**. The assessment says the WSDLs are sent separately.
3. Ask for **IP allow-listing** of the server that will call the API.
4. Ask for a **staging account**.

How the credentials are sent (SOAP header, body fields or WS-Security) is not documented anywhere public. The WSDL will show it.

### 2.4 Operations seen in the client (names real, meaning partly inferred)

| Need | Operation | Notes |
| --- | --- | --- |
| List all numbers | `GetTnInventoryReport` | Likely the inventory report. Confirm fields |
| Number detail | `ViewNumberDetails`, `GetStatusByNumberSearch` | Returns order-style details and status |
| Search available | `SearchNumbers` with `NumberSearchParameters` | Filters: states, LATAs, locations, rate centers, NPA, NXX, quantity, consecutive, **vanity digits**, categories. **No city filter** |
| Search filters | `GetNewNumberSearchFilters`, `GetHierarchicalView` | State > LATA > rate center > NPA > NXX tree |
| Price | **None seen** | Unverified |
| Buy | `PlaceOrder` (toll-free: `PlaceTFOrder`) | Takes a purchase order number (PON). Returns an order ID, so likely async |
| Disconnect | `DisconnectOrder` (toll-free: `PlaceTFDisconnectOrder`) | Takes PON and a list of numbers with a disconnect type |
| Portability check | `PortabilityCheck` | Takes a list of numbers |
| Port-in tracking | `GetOrderStatus`, `GetOrdersByPONSearch`, `SearchOrderDetailsByOrderId`, `GetPortInRelatedOrders` | FOC date appears as a `FOCDate` field in order details |
| Port-in changes | `SupplementOrder`, `ActivateSOA`, `AddNotes`, `CreateException`, `Upload`, `Download` | Upload/download probably carry the LOA and CSR |
| Cancel / reject / resubmit | **No explicit operation seen** | May go through `SupplementOrder` or order status. Unverified |
| E911 | `ValidateE911Address` | Out of scope |

### 2.5 Other points

- **Webhooks:** none seen. Plan on polling.
- **Errors:** SOAP faults (`Fault`) and HTTP errors.
- **Rate limits:** not found.
- **Number format:** mostly strings; some result types use 64-bit integers. Normalize in the adapter.
- **Python client:** none known. Use `zeep` against the provided WSDL.

### 2.6 Impact on the app

- Peerless is the largest carrier and the **least documented**. It is the biggest schedule risk.
- Build the adapter from the WSDL as soon as it arrives. Until then, use the `FakeAdapter`.
- City/state search needs mapping from city to rate center, or filtering on our side.

## 3. BulkVS

All details below are **verified** from BulkVS's public OpenAPI spec, unless marked otherwise.

### 3.1 Documentation

| Item | Value |
| --- | --- |
| Interactive docs | `https://portal.bulkvs.com/api/v1.0/` |
| OpenAPI spec (public, no login) | `https://portal.bulkvs.com/api/v1.0/openapi` (OpenAPI 3.0, version 1.0.05) |
| Base URL | `https://portal.bulkvs.com/api/v1.0` |
| Style | REST / JSON. Keys are passed as query parameters (for example `?Number=...`) |
| Legacy | A SOAP API also exists (`https://portal.bulkvs.com/api/?wsdl`). Do not use it |
| Support | `support@bulkvs.com`, `https://support.bulkvs.com` |

Field names contain spaces and capital letters (for example `"Portout Pin"`, `"TN Details"`), so generated clients need hand fixes.

### 3.2 Credentials

- **Scheme:** HTTP Basic. An unauthenticated call returns `401` with `WWW-Authenticate: Basic realm="Restricted"`.
- **Header:** `Authorization: Basic base64(username:password)`.

**How to extract them (from the Textable setup guide and the BulkVS-DDNS README):**

1. Log in at `https://portal.bulkvs.com/login.php`.
2. Open the **API** menu, then **API Credentials**.
3. Copy the **REST API username and password**. The portal may also show a ready-made Basic auth header.
4. Do not mix these up with the SOAP credentials, which are a separate item on the same page.

**Prerequisites:**

- The account needs a **positive balance and the relevant service enabled**. Most endpoints return `403 Insufficient funds or service not enabled` otherwise. `GET /accountDetail` shows the balance and service flags.
- Unverified: whether support must enable the API, whether API calls are IP-restricted, and whether the API password differs from the portal password.

### 3.3 Endpoints

| Need | Call | Notes |
| --- | --- | --- |
| List all numbers | `GET /tnRecord` (no `Number`) | Filters: Status, Tier, Trunk Group, Activation Date, Limit and more. No page or offset parameter. Maximum `Limit` unverified |
| Number detail | `GET /tnRecord?Number=1310...` | Same shape as the list |
| Update a number | `POST /tnRecord` | `ReferenceId` is free text we can use for our own tags |
| Find stock | `GET /exchanges?Search=631` | Prefixes with quantity |
| **Search available** | `GET /orderTn?Npa=310&Nxx=906` | **NPA and NXX both required.** Returns TN, rate center, state, tier, `Mrc`, `Nrc`, per-minute rate |
| Test a purchase | `POST /orderTn?Checkonly=true` | Returns `Available` without buying |
| Buy | `POST /orderTn` | **Synchronous.** Returns the full record. Error code 7501 if unavailable |
| Disconnect | `DELETE /tnRecord?Number=...` | `409` if the number is in the wrong state |
| Portability check | `GET /validatePortability?Number=...` | Returns `Portable`, losing carrier, LNP fee, Mrc |
| Create port-in | `PUT /portTn` | Takes TN list, BTN, subscriber type, account number, PIN, name, address, requested date, LOA flag. Returns an order ID |
| Track port-in | `GET /portTn?OrderId=` (or `Number=`, `ReferenceId=`) | One parameter only. No parameters returns the last 90 days. **HTTP 300 means several matches: not an error** |
| Update port-in | `POST /portTn?OrderId=...` | Multipart: note, attachments, or `Activate` |
| Account check | `GET /accountDetail` | Balance and enabled services |

**Sample record (list/detail):**

```json
{"TN":"13109060901","Status":"Active","Portout Pin":"3591344","ReferenceId":"Customer A",
 "Routing":{"Trunk Group":"Primary"},
 "TN Details":{"Rate Center":"Los Angeles","State":"CA","Tier":"0","Activation Date":"2010-07-28 00:00:00"}}
```

**Port statuses seen in the spec:** `PENDING`, `SUBMITTED`, `EXCEPTION` (with a `Reason`), `REQUESTED_SUPP`, `FOC`, `COMPLETE`, `CANCELLED`. There is **no separate FOC date field**.

### 3.4 Gaps that affect our requirements

| Requirement | Gap | Plan |
| --- | --- | --- |
| Search by city/state | Not supported | Map city/state to NPA/NXX ourselves, or show a note in the UI |
| Search by pattern (for example ending 1000) | Not supported | Fetch results for chosen NPA/NXX and filter ourselves; only works within that stock |
| City and state on a number | Only `Rate Center` and `State` | Show rate center as the location |
| Cancel, reject, resubmit a port | **No endpoints** | Fix with a note or attachment through `POST /portTn`. Cancel is probably through support or the portal (unverified) |
| FOC date | No date field | Take `RDD` (requested date) from the order; confirm what it shows after FOC |
| Webhooks for ports or inventory | **Messaging webhooks only** | Poll `GET /portTn` and `GET /tnRecord` |
| Sandbox | **None** | Use `Checkonly=true` and read-only calls; real purchases cost real money |
| Disconnected numbers in the list | Unverified | Treat "missing from the list" as missing and flag for review |

### 3.5 Other points

- **Number format:** 11 digits with a leading 1 and no plus sign. Normalize to `+1XXXXXXXXXX`.
- **Errors:** status codes `400, 403, 404, 409, 412, 503`; some bodies carry `Code` and `Description`. The `401` body is plain text, not JSON.
- **Rate limits:** not documented.
- **Clients:** no official SDK. Several community projects exist. Generating a client from the OpenAPI spec is possible but a hand-written `httpx` client is simpler.

## 4. Bandwidth

Details come from Bandwidth's official guides and OpenAPI specs (`https://dev.bandwidth.com/spec/*.yml`).

### 4.1 Which API to use

- Inventory, ordering and porting all live in the **Numbers API** (also called the Dashboard or Iris API). It is **XML-first**, with JSON available on some endpoints.
- Docs: `https://dev.bandwidth.com/docs/numbers`
- Base URL: `https://api.bandwidth.com/api/v1` (US and Canada). V2 (`/api/v2`) adds other countries and some JSON endpoints.
- The older host `dashboard.bandwidth.com/api` still appears in the Python SDK README. Use `api.bandwidth.com`.
- The newer Voice and Messaging APIs are separate products and are not needed here.

### 4.2 Credentials: OAuth 2.0 is required

**Basic auth is being retired.** Build on OAuth from the start.

| Date | Event |
| --- | --- |
| 2026-04-01 | New integrations must use OAuth 2.0 client credentials |
| 2026-07-01 | Legacy Basic auth API users no longer updated |
| 2026-12-01 | Last day to finish migrating |
| **2026-12-02** | **Basic auth API users stop working** |

(Today is 2026-10-10, so this matters for the demo and beyond.)

**How to extract credentials:**

1. Log in at `https://app.bandwidth.com`. You need an **admin**, or a user with the **Credential access** role.
2. Go to **Account > API Credentials** and choose **Add Credential / Create**.
3. Fill in a description, the account scope, an optional secret expiry, the **account(s)** and the **roles**.
4. Save. The **Client ID and Client Secret** are shown. **The secret is shown once only**, so store it immediately in the secrets manager.
5. Up to two active secrets are allowed per credential, which allows rotation.
6. Get the **Account ID** from the app URL: `https://app.bandwidth.com/portal/r/a/{accountId}`. Every Numbers API path contains it.

**Roles:** the docs name only "Porting" and "Ordering". The exact roles needed for inventory read, ordering and porting are unverified. Give the credential roles for ordering, in-service numbers and porting.

**Token request:**

```text
POST https://api.bandwidth.com/api/v1/oauth2/token
Authorization: Basic base64(client_id:client_secret)
Content-Type: application/x-www-form-urlencoded

grant_type=client_credentials
```

The response includes `access_token` and `expires_in` (3600 in the example). Send `Authorization: Bearer <token>` on later calls. Refresh the token 5 to 10 seconds before expiry. A second token host (`id.bandwidth.com`) appeared in one search result. Use `api.bandwidth.com` as documented and confirm.

**IP allow-listing:** not mentioned in the Numbers API docs (unverified).

### 4.3 Account structure needed before ordering

Orders require a **Site** (called a sub-account) and a **Location** (SIP peer):

- Account contains Sites; each Site contains Locations.
- An order needs a `SiteId` and, optionally, a `PeerId`.
- For an inventory app, one Site and one default Location are enough. Create them first (`POST /accounts/{accountId}/sites` and `.../sippeers`), or reuse existing ones.
- Ask DataPhone which Site and Location new numbers should land in.

### 4.4 Endpoints

All paths are relative to `https://api.bandwidth.com/api/v1` (or `/api/v2` where noted), under `/accounts/{accountId}`.

| Need | Call | Notes |
| --- | --- | --- |
| List all numbers | `GET /inserviceNumbers` (v2) | Paged with `page` (an ID, not a number) and `size`; follow `links.next`. **Returns E.164 numbers only** |
| Count | `GET /inserviceNumbers/totals` | |
| Check one number | `GET /inserviceNumbers/{tn}` | 200 if in service, 404 if not |
| Other statuses | `GET /discnumbers` (disconnected), `GET /tns` | Whether `/tns` returns non-in-service numbers is unverified |
| Basic number info | `GET /tns/{tn}` (no account in path) | Status, order ID, in-service date, sub-account |
| **Number detail** | `GET /tns/{tn}/tndetails` | City, state, LATA, rate center, tier, site, location, in-service date. **One call per number** |
| Search available | `GET /availableNumbers` | Params: `areaCode`, `npaNxx`, `npaNxxx`, `city`+`state`, `rateCenter`+`state`, `zip`, `lata`, `localVanity`+`endsIn`, `tollFreeWildCardPattern`, `quantity`, `enableTNDetail`. Returns XML |
| Reserve | `POST /tnreservation` | Search results are **not held**. Semantics unread |
| Buy | `POST /orders` (XML) | Needs `SiteId`. **Async:** returns order ID, status `RECEIVED` |
| Poll order | `GET /orders/{orderId}` | `RECEIVED`, `COMPLETE`, `PARTIAL`, `FAILED`, `BACKORDERED`. Orders can return 201 with errors inside, so check the body |
| Disconnect | `POST /disconnects` (XML) | Async; max 5000 per order. Numbers sit in **aging for 60 days** (US/Canada) |
| Portability check | `POST /lnpchecker?fullcheck=true` | Returns port type, portable numbers, supported/unsupported carriers |
| Create port-in | `POST /portins` | `LnpOrder` with subscriber, address, billing number, numbers, requested FOC, and `ProcessingStatus=DRAFT` |
| Submit port-in | `PUT /portins/{orderId}` with `ProcessingStatus=SUBMITTED` | Drafts untouched for 48 hours are deleted |
| LOA / CSR | `POST /portins/{orderId}/loas`, `/csrs` | Upload documents |
| Track port-in | `GET /portins/{orderId}`, `/history`, `/tns`; list: `GET /portins` | List needs `page` and `size` (max 1000) |
| FOC date | `ActualFocDate` on the order | `GET /portins/{orderId}/availableFocDates` for allowed dates |
| Rejection | Status `EXCEPTION` with an `Errors` list | Codes in the 7000 to 7999 range |
| Fix and resubmit | `PUT /portins/{orderId}` | Moves to `REQUESTED_SUPP`, then `FOC`. Not allowed within 24 hours of FOC |
| Cancel | `DELETE /portins/{orderId}` | Submitted orders go to `REQUESTED_CANCEL`, then `CANCELLED` |

**Port-in statuses:** `DRAFT, PENDING_DOCUMENTS, MISSING_REQUIREMENTS, SUBMITTED, PENDING_CARRIER_APPROVAL, REQUESTED_SUPP, FOC, COMPLETE, EXCEPTION, SNAPBACK, REQUESTED_CANCEL, CANCELLED`.

### 4.5 Callbacks (subscriptions)

- **Subscriptions v1 (XML):** `POST /accounts/{accountId}/subscriptions` with an `OrderType` (`orders`, `disconnects`, `portins`, `portouts`, `bulkPortins`) and a callback URL with optional Basic auth.
  - Payload includes the order ID, status and, for port-ins reaching FOC, `ActualFocDate`.
  - Bandwidth first sends with no credentials; your server must answer `401` with a `WWW-Authenticate` header or the callback is **lost**.
  - **Not retried** if your server is down. Keep polling as the safety net.
  - Bandwidth recommends fetching the order after a notification, since it carries more detail.
- **Subscriptions v2 (JSON)** is recommended for new integrations. Whether it covers number orders and port-ins is unverified; list `GET /subscriptionDefinitions` with real credentials.

### 4.6 Gaps and risks

| Item | Issue | Plan |
| --- | --- | --- |
| Inventory detail | List returns numbers only | Fetch `tndetails` per new or changed number, throttled; sync does a full list then detail only for unknown numbers |
| Pricing | No endpoint found | Show "not provided" in search for Bandwidth; confirm with the account manager |
| Pattern search | No generic "ends in" | Use `localVanity` with `endsIn=true` (numeric behavior unverified) and filter ourselves |
| OAuth deadline | Basic auth stops 2026-12-02 | Use OAuth from day one; cache the token and refresh before expiry |
| XML | Most requests and responses | Parse with `lxml`; keep the XML-to-internal mapping inside the adapter |
| Sandbox | None documented | Use a separate Site and Location for tests; real orders cost money |
| Rate limits | Not published; `429` with `Retry-After` is documented | Throttle and honor `Retry-After` |
| Python SDK | `bandwidth-numbers-sdk` 1.2.0 wraps the Numbers API but its OAuth support is unclear. `bandwidth-sdk` does **not** cover the Numbers API | Write a thin `httpx` + `lxml` client, or verify the SDK first |

## 5. Comparison: How Each Requirement Is Met

| Requirement | Peerless | BulkVS | Bandwidth |
| --- | --- | --- | --- |
| One list of all numbers | `GetTnInventoryReport` | `GET /tnRecord` (complete detail) | `inserviceNumbers` + `tndetails` per number |
| City / state / date on each number | Unverified | Rate center, state, activation date | `tndetails` |
| Search by area code | Yes (NPA) | Yes (NPA + NXX required) | Yes |
| Search by city/state | Rate center and state | No | Yes |
| Search by pattern | Yes (vanity digits) | No | Partly |
| Price in search | Not seen | Yes | No |
| Buy | Async | Sync | Async |
| Disconnect | Yes | Yes | Yes (aging) |
| Portability check | Yes | Yes | Yes |
| Port status + FOC date | Order status + `FOCDate` | Status only, no FOC date field | Status + `ActualFocDate` |
| Rejection reason | Unverified | `Reason` text | `Errors` list |
| Fix and resubmit | Possibly `SupplementOrder` | Note/attachment only | `PUT /portins` |
| Cancel port | Unverified | **No endpoint** | `DELETE /portins` |
| Change notifications | None seen | None for inventory/ports | Subscriptions (not retried) |

## 6. What This Changes in Our Design

1. **Capability flags matter.** The adapter must declare what it supports, and the UI must adapt (see `integration-requirements.md`, Step 2). Examples to add: `search_by_city`, `search_by_pattern`, `search_price`, `port_cancel`, `port_foc_date`, `webhooks`, `inventory_detail_in_list`, `sandbox`.
2. **Search results differ by carrier.** The "side by side" screen must allow blank prices, and a carrier that cannot take a filter must either be skipped with a visible note or filtered on our side.
3. **Sync cost differs.** BulkVS returns everything in one call. Bandwidth needs a detail call per new number. Peerless is unknown.
4. **Polling is the baseline.** Only Bandwidth offers callbacks for orders and ports, and those are not retried. Polling must work for all three.
5. **Port actions are uneven.** For BulkVS, cancel and resubmit may have to be marked "do in the carrier portal" in the UI, with the app recording a note. This is a visible limitation for Feature 5 and should be raised with DataPhone.
6. **Buying is dangerous without a sandbox.** BulkVS has none and Bandwidth documents none. Use a confirmation step, a per-run purchase limit, and `Checkonly=true` for BulkVS during development.
7. **Credentials differ.** Peerless: customer, passcode, user ID. BulkVS: username and password. Bandwidth: account ID, client ID and secret (plus Site and Location IDs). The carriers table should store a **reference** to a secret bundle, not a fixed set of columns.
8. **Normalize every number to `+1XXXXXXXXXX`.** BulkVS returns 11 digits, Bandwidth search returns 10, Bandwidth inventory returns E.164.

## 7. Credential Extraction Checklist

| Carrier | Get it from | You will receive | Also needed |
| --- | --- | --- | --- |
| Peerless | Account rep or support (unverified) | Customer, passcode, user ID | WSDL files, API guide, IP allow-listing, staging account |
| BulkVS | Portal: **API > API Credentials** | REST username and password | Account balance and services enabled |
| Bandwidth | App: **Account > API Credentials** (admin or Credential access role) | Client ID, client secret (shown once) | Account ID, Site ID, Location ID, correct roles |

Rules for all three:

- Store only in a secrets manager or environment variables. Never in code, git or logs.
- Create **dedicated API credentials** for this app so they can be revoked on their own.
- Prefer **read-only** access first for the initial import and sync, and add ordering rights only when the demo needs them.
- Record who created each credential and when it expires.

## 8. Questions to Send to Each Carrier

**Peerless**

1. Please send the WSDL files, the API guide and a sample request for each operation we use.
2. How are `customer`, `passcode` and `userid` sent (SOAP header, body or WS-Security)?
3. Is there a staging account, and which IP addresses must be allow-listed?
4. Which operations cover pricing, cancel, reject, resubmit and FOC date for port-ins?
5. Are callbacks available? What are the rate limits?
6. Is our account now on Infobip, and does that change the API?

**BulkVS**

1. Is there a sandbox or a way to test orders and ports without cost?
2. How do we cancel a port-in or respond to a rejection through the API?
3. Is there a FOC date field, or only the requested date?
4. Do disconnected numbers stay in `GET /tnRecord`? What are the `Limit` maximum and rate limits?
5. Can search by city or pattern be supported, or by rate center?
6. Is the API password the same as the portal password, and is there any IP restriction?

**Bandwidth**

1. Which roles does the OAuth credential need for inventory read, ordering and porting?
2. Is there a sandbox or test environment for the Numbers API?
3. Is there a pricing source (an API or the rate card) for number purchases?
4. Is IP allow-listing used on the Numbers API?
5. Does Subscriptions v2 support number order and port-in events?
6. What are the Numbers API rate limits?
7. Which Site and Location should new numbers use?

## 9. Sources

**Peerless**

- `https://peerlessnetwork.com`
- `https://www.peerlessnetwork.com/peerless-portal/`
- `https://github.com/ubogdan/peerless` (unofficial)
- `https://pkg.go.dev/github.com/ubogdan/peerless` (unofficial)

**BulkVS**

- `https://portal.bulkvs.com/api/v1.0/` (docs)
- `https://portal.bulkvs.com/api/v1.0/openapi` (spec)
- `https://www.bulkvs.com/api`
- `https://www.textable.co/docs/bulkvs-setup-guide/` (credential steps, third-party)
- `https://github.com/PrPlanIT/BulkVS-DDNS` (third-party)

**Bandwidth**

- `https://dev.bandwidth.com/docs/numbers`
- `https://dev.bandwidth.com/docs/numbers/guides/searchingForNumbers`
- `https://dev.bandwidth.com/docs/numbers/guides/disconnectNumbers`
- `https://dev.bandwidth.com/docs/numbers/porting/port-ins`
- `https://dev.bandwidth.com/docs/numbers/webhooks/orderWebhook`
- `https://dev.bandwidth.com/docs/numbers/errors`
- `https://dev.bandwidth.com/docs/account/credentials/management`
- `https://dev.bandwidth.com/docs/account/credentials/migration`
- `https://dev.bandwidth.com/docs/account/structure/`
- `https://dev.bandwidth.com/docs/tools/openapi-specs`
- `https://www.bandwidth.com/support/en/articles/13004306-creating-and-managing-your-bandwidth-api-credentials`
- `https://github.com/bandwidth/python-numbers-sdk`
