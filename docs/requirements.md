# DataPhone Number Management POC: Requirements

## Background

DataPhone is a VoIP provider in Brooklyn, NY. We sell hosted phone systems and SIP trunks. Every customer uses phone numbers (DIDs) that we buy from carriers.

Today our numbers live in three separate carrier portals, and nothing tells us in one place which customer owns which number. When a customer leaves, a number moves or a port finishes, the record lives in someone's head or an email.

We buy numbers from three carriers, listed by volume:

1. **Peerless Network** (largest)
2. **BulkVS**
3. **Bandwidth**

## Goal

Build a proof of concept (POC) that gives us one place to see, manage, buy and port in all our phone numbers across every carrier.

The POC succeeds when our team can do the following **without logging into any carrier portal**:

- Type any phone number and, within seconds, see which carrier it is with, which customer owns it and everything that happened to it.
- Pull up a customer and see all of their numbers.
- Move a number from one customer to another, with a record of who did it and when.
- Search for and buy a new number.
- Start a port-in for a new customer and follow it until it completes.
- See a list of numbers we pay a carrier for that no customer is using.

---

## Feature 1: All Numbers in One Place

Every number we hold at any carrier shows up in one list, and that list stays up to date on its own.

**What we need:**

- One list of all numbers from all carriers.
- For each number: the carrier, the customer it belongs to (if any), its status, the date we got it, and its city and state.
- Search by full or partial number, customer name, carrier or status.
- Filters for:
  - only unassigned numbers,
  - only one customer's numbers,
  - only one carrier's numbers.
- Export the list to Excel/CSV.
- The list stays up to date automatically.

## Feature 2: Assign Numbers to Customers

Every number belongs to one customer or to nobody, and our team can change that in a few clicks.

**What we need:**

- Assign one number, or many at once, to a customer.
- Move a number from one customer to another.
- Take a number back from a customer (for example, when they leave).
- A customer page that lists all of that customer's numbers.

> For the POC, a simple customer list inside the app is fine. Later it will need to match our billing system (Rev.io).

## Feature 3: Full History

Nothing that happens to a number is ever lost. We must be able to answer "who had this number, and when?" years from now.

**What we need:**

- Every change is recorded, including:
  - bought,
  - ported in,
  - assigned,
  - moved,
  - released,
  - disconnected,
  - note added,
  - changes the carrier made on its side.
- Each record shows:
  - what changed, from what to what,
  - who did it (a person or the automatic carrier update),
  - the date and time.
- History can be viewed for one number or for one customer.
- History can **never be edited or deleted**, even by an admin.

## Feature 4: Buy New Numbers

Our team can find and buy a number from inside the app, and it lands in the list already assigned to the right customer.

**What we need:**

- Search available numbers by area code, city/state or pattern (for example, ending in 1000).
- See results from all carriers side by side, with the price where the carrier provides it.
- Pick one or more numbers and buy them.
- Choose the customer at the time of purchase, so the number is assigned right away.
- Disconnect a number we no longer need, so we stop paying for it.

## Feature 5: Port In Numbers for New Customers

When a new customer brings their numbers from another provider, our team starts and tracks the whole port from inside the app.

**What we need:**

- Check first whether the numbers can be ported.
- Create a port request with the customer's details:
  - account name,
  - service address,
  - current provider,
  - account number,
  - PIN.
- Submit the port to the carrier we choose.
- A simple status that everyone understands:

  | Status | Meaning |
  | --- | --- |
  | Draft | Request created, not yet sent |
  | Submitted | Sent to the carrier |
  | Date confirmed | Carrier has given the port date |
  | Completed | Port finished |
  | Rejected | Carrier refused the port |
  | Cancelled | Port stopped |

- Show the confirmed port date (FOC) once the carrier gives it.
- If the carrier rejects the port, show the reason and let us fix and resubmit.
- A list of all open ports, so nothing gets forgotten.
- When a port completes, the numbers appear in the list already assigned to that customer.

## Must Work with Many Carriers

The app must work the same way for our team no matter which carrier a number is with. Adding a fourth carrier later should not mean rebuilding the app.

- The POC must connect to Peerless, BulkVS and Bandwidth.
- Our team should not need to know how each carrier works. The screens are the same for all of them.
- Each carrier's login details are kept safe and are not visible to regular users.

---

## Not Part of the POC

- Porting numbers out to other providers
- E911 address management
- Billing, invoicing or syncing with Rev.io
- Changing call routing in our phone system (FusionPBX)
- SMS/messaging setup
- A customer-facing portal (the POC is for DataPhone staff only)

---

## What We Want Back From You

We are leaving the technical design to you on purpose. How you approach it matters as much as the result.

### Your proposal should cover

1. How you would build it, and why you chose that approach.
2. How the app will handle three carriers that work very differently, and how a new carrier would be added later.
3. How you will keep history complete and trustworthy.
4. How you will keep our list in sync with the carriers.
5. What you would build first, and a timeline in phases.
6. The risks you see and what you would do about them.
7. The questions you need answered before you start.

### At the end of the POC we expect

- A live demo of each feature above, using our real carrier accounts.
- The source code in our GitHub.
- Short notes on how to run it and how to add a new carrier.

## Reference Material

- **Peerless Network API:** WSDL files (`APIService.wsdl` and `APIServicePortType.wsdl`), sent separately.
- **BulkVS:** API documentation.
- **Bandwidth:** documentation on searching and ordering numbers.

Test accounts and API credentials for each carrier will be provided once the proposal is approved.
