---
name: offline-conversion
description: >-
  Make an existing online-only feature work OFFLINE — a POS screen, an order flow, a form, a
  dashboard — without inventing money bugs. The rule this skill exists to enforce: classify the
  RESOURCE before choosing the mechanism. A resource only one client writes (a private order, a
  draft) can be captured locally and replayed as a snapshot. A resource many clients write
  concurrently (a shared cart, a table, a seat, an inventory count) CANNOT — replaying a snapshot
  over it silently deletes other writers' changes. A resource the server arbitrates (payment
  capture, approval, split division, allocation) cannot be decided offline at all and must be
  blocked with a visible reason. Enumerates every action of the target feature, classifies each,
  and only then designs: reference cache + TTL, durable outbox, idempotency by UNIQUE CONSTRAINT
  (never check-then-act), provisional local ids reconciled on sync, and conflict outcomes that
  never drop a sale. Verifies the ledger/side-effect parity a naive port always misses — revenue
  booked at the same step as online, notifications suppressed on replay, timestamps backdated to
  capture. Requires a ≥95% skeptic-corroborated design gate BEFORE any code, because the failure
  mode here is not a crash, it is money that silently never lands. Use whenever the user says
  "make X work offline", "offline mode", "offline-first", "convert X to offline like Y", "works
  without internet", "sync when reconnected", "outage", "flaky wifi", or is porting an existing
  offline feature to a second area of the app.
---

# Offline conversion

Take a feature that assumes the network and make it survive an outage — without losing money,
losing other people's writes, or lying to the operator about what was saved.

```
INVENTORY every action → CLASSIFY each resource (private / shared / server-arbitrated) →
PICK the mechanism per class → DESIGN the sync contract → ⛔ ≥95% SKEPTIC GATE ⛔ →
⛔ APPROVAL ⛔ → build reference cache → durable store → outbox → replay → side-effect parity →
verify in the REAL shipping artifact (the .exe / the device, not just a browser tab)
```

## Why this skill exists

The tempting move is to copy an offline feature that already works elsewhere in the app. That is
the single most reliable way to ship a money bug, because **the mechanism that made it safe there
was a property of the resource, not of the code.** An order that belongs to one till replays
safely. A table that four tills and a customer's phone all write to does not. Same app, same
outbox, opposite outcome.

Everything below exists because a real conversion attempt got this wrong three times in a row.

---

## Phase 1 — Inventory every action (no design yet)

List every action the feature performs, one row each. Not screens — **actions**.

For each: what it reads, what it writes, which endpoint, and **who else can write the same thing**
while this client is offline. That last column is the whole skill.

Sources for "who else": other terminals/devices, the customer's own app, background jobs, webhooks,
admin tools, cron. If you cannot answer it from the code, go and find out — an assumption here is
the bug.

⛔ Do not proceed until every action has an answer in the last column.

## Phase 2 — Classify each resource

| Class | Test | Mechanism | Example |
|---|---|---|---|
| **PRIVATE** | Exactly one client can write it for the duration of the outage | Snapshot capture + idempotent replay | A takeaway order being rung up at one till; a draft; a local form |
| **SHARED** | Two or more writers can touch it concurrently | **Append-only intents, never snapshots** — and the intents must be commutative | A dine-in table's cart; a collaborative doc; an inventory count |
| **ARBITRATED** | The server decides, using state the client cannot see | **Block offline, show the reason** | Payment capture, approval workflows, allocation, split division, anything with a server-side lock |

Three traps:

- **"Nobody else touches it in practice."** Practice is not a constraint. If the API permits a
  second writer, classify it SHARED.
- **A screen is not a resource.** One screen usually spans all three classes. Classify per action.
- **Opening/selecting is not creating.** Check where the server row is actually born. If it is born
  on first write, then "the row always exists by the time we go offline" is false, and any design
  resting on it collapses.

## Phase 3 — Mechanism per class

### PRIVATE — snapshot + idempotent replay
Capture the whole object locally, replay once on reconnect. Requires:
- A **provisional local id** (`OFF-<device>-<seq>`) reconciled to the server id on sync.
- **Idempotency enforced by a UNIQUE CONSTRAINT**, not a check-then-act read. A `findOne` then
  `create` is a race, and worse, it usually becomes *create-or-return*: the second call silently
  returns the stored object and discards the payload it was sent.
- **A revision, if the object can change after its first sync.** Create-or-return plus a client
  that re-sends the same key with new content equals silent data loss with a green success toast.
  Either carry a monotonic revision the server applies, or mint a new key per committed change.
- The server recomputing money authoritatively as the verification of the client's local maths.

### SHARED — append-only, commutative intents
Sync *what the user did*, not *what the object now looks like*. Each intent gets its own
idempotency key.

**Commutative is the hard requirement, and it is where naive op-logs fail.** "Append-only
transport" is not the same as "append-only semantics". Check every intent:
- `ADD(item)` — commutative. Fine.
- `SET_QUANTITY(line, 3)` — **not** commutative. It is last-writer-wins over an absolute value from
  a stale local view, and it will overwrite a concurrent change. Use `ADJUST_QUANTITY(+n)`.
- `DELETE(line)` — needs a tombstone, not a destructive delete, or it removes lines the offline
  client never knew existed.
- Anything setting a scalar on the shared parent (discount, customer, note) is last-writer-wins;
  decide explicitly whether that is acceptable and say so.

Also: intents created offline reference **local** ids. The server must return a local→server id
map, and later intents must resolve through it. Persist that map in the same transaction that marks
the intents applied, or a crash mid-sync strands them.

### ARBITRATED — block, and say why
Disable the control and show the reason in the UI, next to the thing that is disabled. Never let it
appear to work. A one-line honest message ("Settling needs a connection — the order is saved and
will sync") beats an action that silently queues and reconciles wrong.

## Phase 4 — Design the sync contract

Write it down before coding:
- **Identity:** what makes two syncs "the same"? Where is that key stored, and is it also the local
  primary key? (If you make the key per-revision, you have changed the primary key — say where the
  stable identity now lives.)
- **The unique index** that enforces it. Partial, so rows that legitimately have no key are exempt.
- **Outcomes**, and how each is rendered: created / applied / duplicate-retry / stale / conflict /
  refused. ⛔ **A "duplicate" or "conflict" response is NOT a success.** Check the client's outcome
  classifier — if it maps every HTTP 200 to success, a refusal will show the operator a green tick.
- **Conflict policy:** never drop the sale, never create an orphan nothing can reach, always
  surface it to a human with enough context to act. "Dead-letter it" is only a policy if you also
  define what the operator does next.
- **How do you TELL a retry from an edit?** "Same key" is not enough. If the server cannot compare
  what was sent against what it stored, it must either discard the edit (silent money loss) or
  apply it (double-billing). Store a **content fingerprint** alongside the key at create time, in
  the same transaction. Hash only the fields that decide the record — deliberately exclude anything
  the client re-stamps on every rebuild (a capture timestamp is the classic one; include it and
  every honest retry becomes a false conflict). Normalise money EXACTLY as the pricing code does,
  or a separately-built artifact will dead-letter correct records. Records written before the field
  existed have no fingerprint: classify them as duplicates so the existing backlog does not
  dead-letter on upgrade.
- **"Acknowledged" must not mean "deleted."** Whatever button clears the operator's alert has to
  STAMP the local row, never remove it. That row holds the only local evidence of what the server
  already has; delete it and the next action mints a fresh one with no evidence, which against an
  older server reports success for a write that never landed — the original bug, one tap after the
  operator thought they had handled it.
- **Backward compatibility, both directions.** Old client → new server, and new client → old
  server. The second is the one people forget, and with a separately-deployed app (a packaged
  desktop build, an app-store release) it is guaranteed to happen.

## Phase 5 — Side-effect parity (what naive ports always miss)

Going offline changes *when* things happen, and every side effect keyed to "now" is a defect:

- **Money.** Find the exact rule that decides when a sale books to the ledger, and make the offline
  path use the *same shared function* — not a copy of its conclusion. A private copy of a business
  rule is the drift that costs real revenue, and it is invisible until someone reconciles.
- **Timestamps.** Sale date, sequence numbers, business-day partitions and revenue dates must come
  from the **capture** instant, not the reconnect instant. Verify the id generator and the ledger
  writer both accept a backdated time. If one does and the other does not, the two records will
  disagree forever.
- **Notifications and kitchen/fulfilment tickets.** Replaying N queued actions must not fire N
  pushes, or re-fire work already physically done during the outage. Look for an existing
  suppression flag; if the path you are adding has no equivalent, that is a defect, not a detail.
- **Clock trust.** Clamp an absurd client clock on both ends, and fall back to server time.
- **Caches and aggregates.** A backdated write mutates a closed day. Anything caching that day
  needs invalidating.

## Phase 6 — The offline data itself

- **Reference cache** (menu, prices, tax, config) with a **TTL**, tenant/account-scoped, primed
  while online and delta-synced on reconnect using a **server-stamped** watermark (never the client
  clock, and stamp it *before* reading so a racing edit re-sends rather than vanishes).
- **Expiry is a hard stop:** stale prices are wrong prices. Refuse to take orders on an expired
  catalogue and say so, rather than billing yesterday's tax.
- **Commit at the moment of the user action.** Never defer a durable write to unload/quit — a
  packaged app can be force-killed seconds after quit.
- **Settle promises on the transaction, not the request.** In IndexedDB, `onsuccess` fires before
  commit; a rollback then reports a lost write as saved, and an abort leaves the promise pending
  forever, wedging every later write.
- **Storage origin is part of your data's identity.** If the shipping artifact serves the app from
  a host/port that can change, the store moves with it and every unsynced record disappears.
- **One opener per shared local database.** If several services open the same store, they WILL
  drift to different schema versions. The low version then fails permanently — and if the opener
  caches its promise, the rejection is permanent for the whole session, so the queue holding
  unsynced records becomes unreachable with no error. Funnel every consumer through one module;
  handle `onblocked` (another window otherwise stalls the upgrade forever with an unsettled
  promise); on a version-too-low error, reopen and ADOPT what is on disk rather than locking the
  operator out after a rollback; and never cache a failed open.
- **Check what your retry "force" flag resets.** A poll that force-flushes on every tick can reset
  each row's next-attempt time, silently turning an exponential-backoff ladder into dead code that
  hammers a failing endpoint. Force on reconnect and on an explicit user action; never on the timer.
- **Scope by tenant/account**, and be deliberate about legacy rows written before scoping existed —
  hiding them can hide unsynced money.

## Phase 7 — Don't corrupt the feature you copied from

If you are extending an existing offline mechanism to a second area, the shared store now holds two
kinds of row. **Find every reader.** Any query that filters by status/tenant but not by kind will
now see foreign rows — and a "latest active record" lookup will happily hydrate the wrong one into
the wrong screen and sync it to the wrong endpoint.

Grep for every consumer of the store, the outbox, and the status chips — templates included. This
is the step that gets skipped, and it is invisible in review because each individual query still
looks correct.

Also check any auth/error interceptor that special-cases the *first* sync endpoint by string
literal. A second endpoint will not match it, and the usual symptom is the terminal logging itself
out mid-shift.

## Phase 8 — ⛔ Design gate: ≥95%, skeptic-corroborated ⛔

Before writing code, hand the written design to an **independent read-only agent** with fresh
context and instruct it to REFUTE it: find the action it misclassified, the concurrent writer it
forgot, the response it treats as success, the side effect keyed to `now`. Final confidence is the
**lower** of your score and the skeptic's.

- **≥95%** → present the design for approval, then build.
- **<95%** → fix and re-score. **If it fails three times, the architecture is wrong, not the plan.**
  Stop, and take the fundamentals to the user — by then the failures will be pointing at the same
  thing, and that thing is usually a resource you classified as PRIVATE that is actually SHARED.

## Phase 9 — Verify where it actually ships

Testing in a browser tab with devtools offline proves very little.

- Run it in the **real artifact** — the packaged desktop build, the installed app, the device.
  Connectivity detection especially: a probe against the app's own origin can resolve happily
  through a total WAN outage, so the offline path never runs at all in the shipped build.
- **Kill the app mid-order**, relaunch, confirm restore.
- **Reconnect and confirm exactly one** of everything: one order, one ledger row, correct date,
  correct sequence number.
- **Force a duplicate replay** and confirm no second row.
- For SHARED resources: have a **second writer change the same object during the outage**, then
  sync, and confirm their change survived. This is the test that catches the bug this skill exists
  for, and it is the one nobody runs.
- Verify the operator-visible states: offline badge, pending count, failed/retry, conflict.

## Red flags

| Thought | Reality |
|---|---|
| "Copy how the other screen does it offline" | The other screen's resource was private. Classify first. |
| "Append-only, so concurrent writes are safe" | Only if the ops are commutative. `SET x = 3` is not. |
| "The server dedupes, so replay is safe" | Create-or-return **discards** the new payload. That is data loss with a success response. |
| "It returned 200, it synced" | Duplicate / stale / conflict are all 200 in most APIs. |
| "Nobody else edits that record" | If the API allows it, it happens. |
| "We'll reconcile the ledger later" | Nobody reconciles later. Book it at the same step as online, or it is gone. |
| "It works offline in Chrome" | The shipped build detects connectivity differently. Test the artifact. |
| "Just cache the API responses" | A service worker caches *responses*. Offline writes need a durable store and an outbox. |

## Definition of done

- [ ] Every action inventoried, with its concurrent writers named.
- [ ] Every resource classified PRIVATE / SHARED / ARBITRATED, and the mechanism follows the class.
- [ ] SHARED intents verified commutative (no absolute-value setters, deletes are tombstones).
- [ ] Idempotency enforced by a unique constraint; post-sync changes have a defined path.
- [ ] Every response outcome mapped; duplicate/stale/conflict never render as success.
- [ ] Money booked by the same shared rule as the online path — not a copy of it.
- [ ] Timestamps, sequence numbers and revenue dated to capture, verified end to end.
- [ ] Replay suppresses notifications and already-completed fulfilment work.
- [ ] Reference cache scoped + TTL'd; expiry blocks ordering with a visible reason.
- [ ] Every reader of a shared offline store filters by kind (if extending an existing mechanism).
- [ ] Design scored ≥95% with an independent skeptic BEFORE code; approved by the user.
- [ ] Verified in the real shipping artifact, including a concurrent-writer test and a forced replay.
