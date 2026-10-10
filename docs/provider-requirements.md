# Provider Requirements

This document lists what we need from each carrier, and what we must confirm, before and during integration. It supports `integration-requirements.md`.

> **Important:** The API notes below are working assumptions based on general knowledge of each carrier. Confirm every item against the official documentation and the real test accounts before building. Items marked **Confirm** are not yet verified.

## 1. Summary

| | Peerless Network | BulkVS | Bandwidth |
| --- | --- | --- | --- |
| Volume | Largest | Second | Third |
| API style (expected) | SOAP (WSDL provided) | REST / JSON | REST, mostly XML (Dashboard API) |
| Reference material | `APIService.wsdl`, `APIServicePortType.wsdl` (sent separately) | BulkVS API documentation | "Searching and ordering numbers" docs |
| Auth (expected) | **Confirm** | **Confirm** (often HTTP Basic) | **Confirm** (often HTTP Basic with account ID) |
| Build order | 1 | 2 | 3 |

## 2. What We Need From Every Carrier

Use this checklist for each carrier before development starts.

### Access

- [ ] Test (sandbox) account and credentials
- [ ] Production credentials (read-only if the carrier offers it)
- [ ] API base URLs for test and production
- [ ] Allowed IP addresses, if the carrier uses an allow list, and our server's outbound IP
- [ ] A technical contact or support channel for API questions

### Capabilities (confirm each, per carrier)

| Capability | Needed for | Peerless | BulkVS | Bandwidth |
| --- | --- | --- | --- | --- |
| List all numbers on the account | Feature 1 | ? | ? | ? |
| Get one number's details (status, city/state, date acquired) | Features 1, 3 | ? | ? | ? |
| Search available numbers (area code, city/state, pattern) | Feature 4 | ? | ? | ? |
| Price returned in search | Feature 4 | ? | ? | ? |
| Order / buy numbers | Feature 4 | ? | ? | ? |
| Disconnect / release a number | Feature 4 | ? | ? | ? |
| Check port-in eligibility | Feature 5 | ? | ? | ? |
| Create and submit a port-in | Feature 5 | ? | ? | ? |
| Port status and FOC date | Feature 5 | ? | ? | ? |
| Rejection reason on a port | Feature 5 | ? | ? | ? |
| Update and resubmit a rejected port | Feature 5 | ? | ? | ? |
| Cancel a port | Feature 5 | ? | ? | ? |
| Webhooks / callbacks | Sync | ? | ? | ? |
| Change history or audit data from the carrier | Feature 3 | ? | ? | ? |

Fill in each "?" with Yes, No or Partial as the documentation is reviewed.

### Limits and behavior

- [ ] Rate limits and request quotas
- [ ] Pagination method for large number lists
- [ ] Maximum numbers per order and per port request
- [ ] Timeouts and typical response times
- [ ] Is ordering instant, or asynchronous (needs polling)?
- [ ] Are test orders free, or do they cost money? Can they be reversed?
- [ ] Number format returned (10 digits, with or without `+1`)
- [ ] Full list of number statuses and port statuses the carrier uses

## 3. Peerless Network

**Interface:** SOAP, described by `APIService.wsdl` and `APIServicePortType.wsdl`.

### Setup steps

1. Receive the two WSDL files and test credentials.
2. Generate or load a SOAP client from the WSDL (for example, Python `zeep` or the equivalent in the chosen language).
3. Make a first authenticated call and confirm it works.
4. List the WSDL operations and map each to the adapter interface (`integration-requirements.md`, Step 2).
5. Build, then test against the sandbox.

### Specific requirements

- Handle SOAP faults and turn them into readable errors.
- Handle large responses and any paging.
- Confirm which operations exist for list, search, order, disconnect and port. Large carriers often split these across several services.
- Check whether ordering and porting are asynchronous.

### Open questions

- Which WSDL operations cover each adapter operation?
- How is authentication done (username and password in the request, header, certificate, IP allow list)?
- Can we list every number on the account, and does the response include city and state and the acquired date?
- How are port statuses and rejection reasons returned?
- Are callbacks available, or is polling the only option?

## 4. BulkVS

**Interface:** REST (expected). Use the BulkVS API documentation.

### Setup steps

1. Obtain API credentials and read the documentation.
2. Make a first authenticated read call and confirm it works.
3. Map each documented endpoint to the adapter interface.
4. Build, then test against the sandbox or a low-cost test setup.

### Specific requirements

- Handle JSON responses and HTTP error codes.
- Handle the way BulkVS identifies a number (its own record or reference, if any).
- Check whether search results include price.
- Check whether number lists, orders and ports are paged.
- Check whether webhooks exist for ports and orders.

### Open questions

- What authentication does the API use, and are credentials per account or per user?
- Does a number record include city/state and the date acquired?
- What is the port-in flow and which statuses does it return?
- Is there a sandbox, or do test actions use the live account?
- Are webhooks available?

## 5. Bandwidth

**Interface:** REST, with the Dashboard API mainly using XML (expected). Use the "searching and ordering numbers" documentation.

### Setup steps

1. Obtain account ID, API credentials and the details of the relevant site (sub-account) and location (SIP peer), if used.
2. Make a first authenticated read call and confirm it works.
3. Map each endpoint to the adapter interface.
4. Build, then test against the sandbox.

### Specific requirements

- Parse XML responses and build XML requests.
- Orders are expected to be asynchronous. Poll the order status or use callbacks.
- Check whether orders require a site and location. These need to be configured in the account.
- Check how in-account numbers are listed, and whether the list can be filtered and paged.
- Check how port-in requests and their statuses work.

### Open questions

- Which account, site and location values must be set on orders?
- How are callbacks or subscriptions configured?
- How do search, order and port statuses map to our internal ones?
- What are the rate limits?
- Is a separate sandbox available?

## 6. Information to Request From DataPhone

These unblock all three integrations:

1. Which carrier accounts and sub-accounts hold the numbers, and approximately how many numbers per carrier.
2. Whether production credentials may be used read-only for the initial import and demo.
3. Who owns each carrier relationship and can raise API issues.
4. Whether buying and disconnecting numbers in the demo should use real accounts, and the spending limit.
5. A starting customer list (CSV) and any existing number-to-customer records.
6. The outbound IP address(es) needed for carrier allow lists.
7. Hosting and where secrets should be stored.

## 7. Credential Handling Rules

- One set of credentials per carrier per environment (test, production).
- Stored in a secrets manager or environment variables, never in code or in git.
- Shared with the team through a secure channel only.
- Visible only to the server process; the UI shows "configured" or "not configured", never the secret.
- Rotated when anyone with access leaves.

## 8. Provider Readiness Checklist

A carrier is ready for integration when:

- [ ] Documentation received and read
- [ ] Test credentials work
- [ ] Capability table (Section 2) is filled in
- [ ] Open questions answered
- [ ] Status mapping table drafted
- [ ] Rate limits and ordering behavior understood
- [ ] Credentials stored securely
