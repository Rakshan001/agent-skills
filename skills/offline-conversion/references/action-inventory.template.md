# Offline conversion — action inventory

One row per ACTION (not per screen). Fill the last three columns before designing anything.

Feature: `<name>`  ·  Repo/path: `<where>`  ·  Date: `<YYYY-MM-DD>`

## The inventory

| # | Action | Endpoint / write | Reads | **Who else can write this during an outage?** | Class | Mechanism |
|---|---|---|---|---|---|---|
| 1 | e.g. add item to cart | `POST /cart/create` | menu, tax | another till (same table), the customer's phone | SHARED | `ADD_LINE` intent |
| 2 | e.g. print bill | `GET /cart/view` | cart | — | PRIVATE | local render from cache |
| 3 | e.g. settle | `POST /cart/tobill` | cart, split state | any till | ARBITRATED | block offline |

**Class:** PRIVATE (one writer) · SHARED (concurrent writers) · ARBITRATED (server decides using
state the client cannot see).

⛔ An empty "who else" cell is not an answer. Find out from the code. If the API permits a second
writer, the class is SHARED regardless of how unlikely it seems.

## Where is the row actually born?

The server record often does not exist when the user *thinks* they started. Selecting a table,
opening a form or navigating is usually client-side only; the row appears on the first write.

- Server record created by: `<endpoint / line>`
- Therefore, can the user be offline with local state and NO server record? **yes / no**
- If yes, every design that assumes "the record already exists" is invalid.

## Side effects keyed to "now"

| Side effect | Where | Must be dated to capture? | Does the writer accept a backdated time? |
|---|---|---|---|
| Ledger / revenue row | | yes | |
| Sequence / document number | | yes | |
| Business-day partition | | yes | |
| Push notification | | n/a — must be SUPPRESSED on replay | |
| Fulfilment / kitchen ticket | | n/a — must not re-fire work already done | |
| Cache invalidation for a closed day | | | |

## The money rule

- The function that decides when revenue books: `<file:line>`
- Does the offline path call that same function, or a copy of its conclusion? **shared / copied**
- If copied → that is the drift. Route it through the shared one.

## Existing offline mechanism being extended (if any)

- Store(s) reused: `<names>`
- **Every reader of those stores** (include templates), and whether each filters by kind:

| Reader | file:line | Filters by kind? |
|---|---|---|
| | | |

- Auth/error interceptor special-casing the first sync endpoint by literal: `<file:line>` — widened? **yes / no**

## Outcome mapping

| Server outcome | HTTP | Client renders as | Correct? |
|---|---|---|---|
| created | 200 | synced | |
| applied (revision) | 200 | synced | |
| duplicate retry | 200 | synced (no-op) | |
| stale | 200 | **not** success | |
| conflict | 200/409 | needs attention + operator action | |
| refused (validation) | 422 | dead-letter | |
