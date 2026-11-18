# Offline conversion — verification checklist

Run these in the **real shipping artifact** (packaged desktop build, installed app, physical
device). A browser tab with "offline" ticked in devtools proves almost nothing: connectivity
detection, storage origin, service-worker behaviour and process lifecycle all differ there.

## 0. Prove the app can even SEE the outage

Before testing anything else. A probe pointed at the app's own origin (a bundled static server, a
local file) resolves happily through a total WAN outage, so the offline path never runs in the
shipped build even though it works perfectly in a dev browser.

- [ ] Pull the real network (not devtools). Confirm the app flips to offline within its probe interval.
- [ ] Confirm the probe target is a remote host the outage actually affects.
- [ ] Restore the network. Confirm it flips back and a flush starts.
- [ ] Trigger the flush from the app's own "sync now" control while offline-mode was toggled by
      hand. A shared in-flight probe can hand that caller a stale verdict, so it reports
      "unreachable" and skips the flush on a perfectly healthy connection.
- [ ] Toggle manual/offline mode off again and wait: the reachability heartbeat must RESTART, not
      terminate permanently.
- [ ] Run a second OS user session (fast user switch / RDP) with the app already open in the first.
      A per-user instance lock plus a machine-wide port means the second copy collides — confirm it
      either starts safely or fails visibly, and never on an origin that hides the first user's data.
- [ ] Confirm a captive portal / non-2xx response is not treated as online.

## 1. Capture

- [ ] Build a full record offline. Totals, tax, discounts and surcharges match what the online path
      would compute for the same input (compare against a server recompute, not against yourself).
- [ ] Every control that is ARBITRATED is visibly disabled and states why.
- [ ] An expired reference cache blocks new records with a visible reason rather than billing stale prices.
- [ ] With no cache at all, the screen says so rather than rendering empty.

## 2. Durability

- [ ] Refresh mid-record → state restored.
- [ ] **Force-quit** the app mid-record (kill the process, don't quit cleanly) → relaunch → restored.
- [ ] Fill storage toward the quota → the warning appears, and the documented behaviour at the
      critical threshold actually happens.
- [ ] Log out and back in → another tenant/account cannot see the first one's unsynced records, and
      the first one's records are still there.

## 3. Sync — the single-writer cases

- [ ] Reconnect → exactly ONE record on the server per record captured.
- [ ] The ledger/revenue row exists, for the right amount, with the right tender.
- [ ] It is dated to the **capture** instant, not the reconnect instant — check the record date, the
      sequence number's day, and the revenue date, and confirm all three agree.
- [ ] Sequence/document numbers are contiguous and belong to the sale's business day.
- [ ] Force a duplicate replay (re-arm the same key, or replay the request) → no second record, no
      second ledger row.
- [ ] Re-arm the same key with DIFFERENT content (edit the record after it synced) → the server
      refuses and the client shows it. A green success toast here is the defect this checklist
      exists for.
- [ ] Clear that alert ("Reviewed"/"Acknowledge"), then touch the record again → it does NOT come
      back as a false success, and does not silently re-send.
- [ ] Kill the app mid-flush → relaunch → the in-flight record is re-queued, not lost, not doubled.
- [ ] Let a sync fail permanently → it dead-letters visibly with a retry the operator can use.
- [ ] Expire the auth token while offline → the sync surfaces re-auth instead of logging the
      terminal out or silently discarding the queue.

## 4. Sync — the SHARED cases (the tests nobody runs)

⛔ These are the ones that catch the class of bug this skill exists for. Do not skip them.

- [ ] **Concurrent writer survives.** While client A is offline, have client B (another terminal, or
      the customer's own app) modify the same record. Reconnect A. **B's change must still be there.**
- [ ] **Two offline writers converge.** Take A and B both offline on the same record, both make
      changes, reconnect both. Nothing either of them did may vanish. If one must lose, it is
      surfaced to a human, not dropped.
- [ ] A line another client cancelled during the outage does not come back from the dead.
- [ ] A field only the other client set (attribution, membership, ownership) is not nulled by the sync.
- [ ] Work already physically completed during the outage is not re-fired on replay.
- [ ] A replay of N queued records fires ONE summary notification, not N.

## 5. Conflict handling

- [ ] Make the sync target vanish (another client settles/deletes it during the outage). The result
      is a visible conflict with an operator path forward — **not** a 500, not a silent success, and
      not an orphaned record nothing in the UI can reach.
- [ ] Confirm the money still lands somewhere reachable. "The sale is dropped" is never an acceptable
      conflict outcome.

## 6. Compatibility

- [ ] **Old client → new server.** An un-updated build syncs successfully and behaves no worse than before.
- [ ] **New client → old server.** The updated build does not dead-letter everything because the
      server does not yet return a new field. (With separately-deployed apps this WILL happen.)

## 7. Regression on the feature you copied from

If you extended an existing offline mechanism:

- [ ] The original feature still works end to end offline.
- [ ] Its "resume latest record" / "held items" / pending-count surfaces show ONLY its own kind of row.
- [ ] Its status chips and counts are unchanged with the new feature's rows present.
- [ ] A record of the new kind can never be hydrated into the old screen (try it deliberately).
