---
name: privacy-compliance
description: >-
  Take a web app or codebase from "we have some legal pages" to an evidenced privacy gap report
  and, after approval, working controls (privacy notice, cookie policy, consent, age gate,
  account deletion that survives immutable data, export, retention jobs, grievance page, breach
  runbook) proven by drills. India (DPDP Act + Rules 2025, IT Intermediary Rules, CERT-In, SPDI),
  EU/UK (GDPR, ePrivacy, PECR) and US (CCPA/CPRA, COPPA, CAN-SPAM). Use when
  the user says "privacy policy", "cookie policy", "cookie banner", "do we need consent", "GDPR",
  "DPDP", "DPDPA", "IT Rules", "grievance officer", "CERT-In", "CCPA", "COPPA", "data deletion",
  "delete my account", "right to be forgotten", "erasure", "data export", "DSAR", "data
  retention", "data map", "records of processing", "sub-processors", "breach notification",
  "security.txt", "DMCA agent", "are we compliant", "privacy audit", "legal pages before launch",
  or "/privacy-compliance". Not legal advice and never certifies anything as compliant. NOT for a
  general security review or pentest, reviewing or negotiating a single contract or DPA, or a
  one-off legal question with no codebase to inspect.
---

# Privacy compliance: from legal pages to evidenced controls

An agent asked to "add a privacy policy" writes a plausible page that does not match the code: it
omits the session IP column, names no processors, promises a deletion the schema cannot perform,
and calls an unsalted hash "anonymous". This skill inventories what the code actually collects,
maps it to the law that applies, ranks the gaps, and builds controls only after approval, then
proves each control with a drill.

```
LEDGER CHECK (PRIVACY-LEDGER.md: resume?) + NOT-LEGAL-ADVICE statement →
1 INVENTORY from the code (schema, forms, cookies/storage, logs, scripts, processors, uploads) →
2 APPLICABILITY (where users are, thresholds, intermediary?, children?, dates) →
3 GAP REPORT (launch-blocking · deadline · scale) incl. what blocks erasure →
⛔ 4 APPROVAL GATE: no code, schema, page or config edit before this ⛔ →
5 BUILD unit by unit (pages · consent + age gate · rights · retention · email · logs · runbook) →
6 DRILLS (consent-declined network capture · delete-a-user sweep · export check · timers) →
⛔ 7 SKEPTIC ≥95% (lower score counts) ⛔ → DEFINITION OF DONE → COMMIT (auto) → ⛔ ASK BEFORE PUSH ⛔
```

**Golden rules**

0. **Not legal advice, and never "compliant".** Say so at the start of every run and at the top of
   every report. The output is a gap report plus evidence. Never write "compliant", "GDPR-ready",
   "safe" or "secure" about the product, in pages, commits or summaries; describe the control
   ("account deletion removes email, name and avatar within 30 days; see drill D2").
1. **The code is the source of truth, not the existing policy.** Inventory first. A notice that
   does not describe what the code does is a misrepresentation, however well written.
2. **Dates move.** Commencement dates and deadlines live in `references/regimes.md` with a
   "last verified" line. If that date is more than 90 days old, or a deadline is within 90 days,
   re-verify against the primary source before stating it to the user.
3. ⛔ **Nothing is edited before the approval gate (Phase 4)**, including "just the privacy page".
   Phases 0–3 write only `PRIVACY-LEDGER.md`.
4. ⛔ **Never invent legal facts about the user's company**: legal entity name, address,
   grievance officer, representative, governing law, DMCA agent, processor contracts. Ask. A
   page with a guessed or placeholder officer is worse than no page, because it looks finished.
5. **Consent only where the law needs it.** Strictly necessary storage and contract-based
   processing need none; non-exempt trackers need opt-in. A banner for exempt cookies adds
   friction and no protection.
6. **Erasure is designed, not bolted on.** Personal data stays out of immutable rows; deletion
   removes the mapping. Soft-delete flags and unsalted hashes are not erasure.
7. **A control without a drill is a claim.** Phase 6 evidence is what the report cites.
8. **Commit automatically, never push.** Ask about push and PR together, with the base branch.
   If the project forbids agent commits, leave the changes in the working tree and say so.

> **Adapt to the project.** Wherever this says "the schema", "the email provider", "object
> storage" or "the scheduler", substitute the project's real one. Anything marked *(optional)* is
> skipped when the project lacks it. Never add infrastructure (a consent platform, a queue, a
> new processor) that is not already there without listing it in the plan for approval.

**Autonomy contract.** Unattended, this skill reads the code, writes the ledger, drafts the gap
report, and (after approval) builds and drills the approved units. It stops for exactly: the
approval gate; any fact in rule 4; a gap whose fix is a product or legal decision (redacting
content inside immutable versions, refusing an erasure, keeping data for a retention duty); a
schema migration on a database it cannot roll back; restoring a backup or calling a live
processor delete API during a drill; a skeptic score below 95 it cannot close; and push/PR.

---

## Phase 0 — Ledger check, then the disclaimer

1. Read `PRIVACY-LEDGER.md` at the repo root (template in Appendix A). If it exists, resume at the
   first unticked unit; trust the code over the ledger when they disagree, fix the ledger, and say
   so. With a shared tracker *(optional)*, recall prior privacy decisions and check that no live
   session is editing the files you will touch.
2. Tell the user, before anything else: *"This is an engineering gap analysis, not legal advice.
   It will not make the product 'compliant'. It produces a gap report with evidence, a list of
   questions for a lawyer, and (with your approval) controls with drills that show they work."*
3. Record the run date and the `last verified` date of `references/regimes.md` in the ledger.

## Phase 1 — Data inventory from the code

Read `references/inventory.md` for the search recipes per stack. Build the data map **from code
and config, not from memory or from the existing policy**. Every row cites `file:line`.

| Field | Purpose | Source (form / API / cookie / derived) | Store (table.column, bucket, log, provider) | Processor | Retention (actual, from code) | Lawful basis (draft) | Exported? | Erased? |
|---|---|---|---|---|---|---|---|---|

Cover, and write "none found" for any that is empty:

- **Schema and migrations**: every column holding a person's data, including IPs, user agents,
  free-text fields, JSON blobs, hashes of identifiers, and referrer or device ids.
- **Forms and APIs**: every input that reaches storage; sign-up, profile, onboarding, feedback,
  support, comments, uploads.
- **Cookies and browser storage**: every cookie (name, lifetime, path, first/third party) and
  every `localStorage` / `sessionStorage` / IndexedDB key; service workers and caches.
- **Logs**: application logs, request logs, error tracking, audit/activity tables, staff action
  logs; what they contain and where they are retained (including the host's own log retention).
- **Analytics and third-party scripts**: every script, pixel, font, embed, avatar host and CSP
  `connect-src` / `img-src` entry; what the browser sends to each.
- **Processors**: hosting, database, object storage, email, auth/OAuth providers, payments,
  analytics, error tracking, search, AI APIs. Note each one's region.
- **Email**: transactional vs promotional, unsubscribe handling, suppression lists.
- **Immutable and append-only data**: every trigger, rule or policy that forbids `UPDATE` /
  `DELETE`, and every foreign key to the user table with its `ON DELETE` action. This list feeds
  the erasure design (`references/erasure-patterns.md` §1 has the queries).

Write the map into the ledger. ⛔ Do not proceed with an empty row you did not search for.

## Phase 2 — Which regimes apply

Ask, one question at a time, with the recommendation marked, and record the answers:

1. Where are users, and where is the company established? *(Recommended: assume India + EU/UK +
   US users for a public English site until the user says otherwise; record it as an assumption
   for the lawyer.)*
2. Does the product host content users upload or publish? If yes, it is likely an intermediary
   under the IT Act, and the IT Rules duties apply now. *(Recommended: if any user content is
   public, treat it as an intermediary.)*
3. Could children use it? *(Recommended: 18+ accounts with an age declaration at sign-up.)*
4. Any ads, ad pixels, sold or shared data, or payments? Revenue and California user counts (for
   CCPA thresholds)? *(Recommended: if unknown, record CCPA as "not shown to apply" and add the
   thresholds to the lawyer questions; never mark it "does not apply" without numbers.)*
5. Promotional email? A mobile app in Google Play? Google sign-in (OAuth verification rules)?
   *(Recommended: answer from the code; ask only what the code cannot show.)*

Then fill the applicability table from `references/regimes.md`: regime · in force on which date
· why it applies or not · threshold evidence. Copy the open lawyer questions that apply.

## Phase 3 — Gap report, ranked

One row per gap, cited to both the obligation and the code:

| # | Gap | Obligation (regime + article/rule) | Evidence (`file:line` or "absent") | Rank | Fix | Owner (code / lawyer / ops) | Blocks erasure? |
|---|---|---|---|---|---|---|---|

- **Rank L (launch-blocking):** in force now, or required by a platform you depend on (Google
  OAuth, Play). Examples: IT Rules grievance officer, CERT-In 6-hour runbook, privacy notice that
  matches the inventory, account deletion, non-exempt trackers firing without consent.
- **Rank D (deadline):** in force on a future date in `references/regimes.md` (DPDP operative
  Rules from 13 May 2027; the Act's sections ~May 2027, secondary). Put the date in the row.
- **Rank S (scale):** triggered by a threshold (CCPA, SSMI, SDF designation).
- **What blocks erasure** gets its own sub-list: each immutable trigger, each `NO ACTION` foreign
  key, circular references, personal data inside immutable content bodies, unswept stores
  (object storage, search index, email provider, logs, backups).
- **Inaccuracies** in existing pages are gaps too (a policy that says "we do not store IPs" while
  a sessions table does).

Close the report with: the open lawyer questions, the facts needed from the user (rule 4), and the
proposed build units in order.

## Phase 4 — ⛔ Approval gate ⛔

> ⛔ **Present the gap report and the build plan, and STOP. Do not edit any code, schema, page,
> config or email template until the user explicitly approves.** This applies to every unit,
> including a one-paragraph page change. If they approve a subset, build only that subset.

The plan lists, per unit: files to change, migrations (with rollback), new jobs, any new
dependency or processor, the drill that will prove it, and the facts still needed from the user.

## Phase 5 — Build, unit by unit

Tick each unit in the ledger **as it lands**. Edit only the approved files; a new file or
dependency means re-plan and re-approve.

1. **Pages** — follow `references/pages-checklist.md`: privacy notice (required contents per
   regime), cookie policy, terms and content policy, grievance and contact page, sub-processor
   list, DMCA page *(if US-hosted UGC)*, `/.well-known/security.txt`, account deletion and export
   help text stating what is kept, why and for how long. Every statement in the notice traces to
   an inventory row; every inventory row appears in the notice. Version and date each page.
2. **Consent and age gate** — consent UI only for non-exempt storage and scripts; nothing
   non-exempt loads before an affirmative choice; withdrawing is as easy as giving; the choice is
   recorded (what, when, notice version). Age declaration at sign-up, recorded, with the
   consequence of a "no" defined. Honour Global Privacy Control *(if CCPA applies)*.
3. **Rights** — export, correction (including email), and deletion per
   `references/erasure-patterns.md`: opaque actor ids in immutable rows, personal data in
   deletable tables or under a per-user key, a "deleted user" tombstone, the mapping deleted, no
   email hash kept, every processor told to erase, an erasure register re-applied after any backup
   restore, and retention exceptions stated with their period and legal basis. ⛔ Before writing
   the erasure migration, run the Phase 7 skeptic on the **design** (FK/trigger list, the class
   of every referencing table, the migration and its rollback); below 95, fix it before coding.
4. **Retention jobs** — one scheduled job per data class in the inventory, with the period from
   the retention schedule; never shorter than a statutory minimum, never "forever".
5. **Email** — unsubscribe link and `List-Unsubscribe` headers on non-transactional mail, honoured
   within 10 business days; postal address and ad label where CAN-SPAM requires them.
6. **Logs and IPs** — replace unsalted hashes of IPs or emails with a keyed hash (HMAC with a
   server-held secret, rotated for analytics so days cannot be linked); redact personal data from
   logs; set log retention to the schedule; sync clocks to a traceable NTP source.
7. **Breach runbook** — `references/breach-runbook.md`, filled in with the user's real contacts.

⛔ If the same unit fails three times, stop: three failures means the inventory or the erasure
design is wrong, not the code. Say what you now believe and decide with the user.

## Phase 6 — Drills (evidence, not assertions)

Run every drill whose unit was built; a drill is skipped only when its unit was not approved, and
the ledger says so. Paste the command and its output (or screenshot) into the ledger.

- **D1 consent declined:** fresh browser profile, decline (or no choice), load every template,
  signed out **and** signed in; capture all network requests and storage writes. Pass = every
  stored key, and every request (first- or third-party) that reads or writes device storage or
  sends data to analytics, ads or embeds, is on the exempt list in the cookie policy. Then accept, withdraw, reload: the non-exempt requests must stop again.
- **D2 delete a test user:** create a user that touches every inventory row (profile, upload,
  comment, email, session, device, OAuth link, export), plus a second user who replies to its
  content. Delete the first, wait for the jobs, then search **every store** for its email, name,
  handle, IP, user agent, device or machine id, OAuth provider ids, avatar URLs, and the hashed
  forms of its IP and email as the code computes them: database (all text and JSON columns),
  session, cache and rate-limit stores, object storage prefixes and shared blob stores, search
  index, auth provider, email provider contacts and send logs, logs, error tracker, analytics.
  Pass = only the stated exceptions remain, each with its period; the opaque actor id survives
  only as the tombstone; the second user's content is intact.
- **D2b backup restore** ⛔ ask first: restore into an isolated scratch database that serves no
  traffic, re-apply the erasure register, repeat the D2 search, then destroy the scratch copy.
  Processor delete calls in D2 use the provider's test mode; where none exists, ⛔ ask before
  calling the live API.
- **D3 export:** request an export for the D2 user before deleting; validate it against a schema
  generated from the inventory. Pass = every row whose subject is the user is present,
  machine-readable, and nothing about other people leaks in.
- **D4 timers:** submit a grievance and a rights request; confirm the acknowledgement arrives
  within the published period and the request is tracked against each deadline that applies
  (24 h acknowledge / 7 days resolve under the IT Rules; one month under GDPR; at most 90 days
  published under DPDP).
- **D5 retention:** seed rows older and younger than each period; run the job. Pass = old rows
  gone, rows inside a statutory minimum kept.
- **D6 static checks:** no unsalted hash of IP or email remains (greps in
  `references/inventory.md` §10); `security.txt` has `Contact` and a future `Expires`; every page
  linked from the footer; a mail sample shows the unsubscribe header.
- **D7 consent and age records:** answer "no" to the age question: no account row is created.
  Accept and then withdraw consent: both records exist, with time and notice version.
- **D8 breach tabletop:** run the rehearsal in `references/breach-runbook.md`; record timings
  and which contacts answered.

A failed drill sends you back to Phase 5 for that unit; never edit the drill to pass.

## Phase 7 — ⛔ Skeptic gate, done, commit ⛔

1. Score your confidence (0–100) that: every inventory row is accurate and covered by the notice;
   every built unit passed its drill; no statement claims compliance or safety; every date matches
   `references/regimes.md`; and D2 leaves no personal data outside the stated exceptions.
2. Spawn an independent read-only reviewer (fresh context, no edit tools). Give it the ledger,
   the diff, the drill evidence and the reference files, and instruct it to **refute**: find a
   store D2 did not search, a tracker D1 missed, a notice claim the code contradicts, a date that
   is wrong, a placeholder presented as fact.
3. **Final confidence = the lower of the two.** ≥95 → continue. Below 95 → fix the named gaps and
   re-run the affected drills. Cannot close because it is a legal or product decision → ⛔ stop and
   put it to the user. "Everything is green, the skeptic is ceremony" is exactly when it pays.
4. Tick the definition of done, then commit only this work (conventional message, project git
   author, body naming the gaps closed and the drills that prove them). ⛔ **Do not push.** Ask
   about push and PR together, with the base branch.

---

## Rationalisations

| Thought | Reality |
|---|---|
| "Our analytics is cookieless, so no consent is needed" | EDPB Guidelines 2/2023 (¶33, 50–51): a script that makes the browser send data back is in Art. 5(3) scope with or without cookies. Check the exemption per country (CNIL audience measurement, UK statistics exception) and give an opt-out. |
| "We hash the IP, so it is anonymous" | An unsalted SHA-256 of an IPv4 address is reversed by hashing all 2^32 addresses. A keyed hash is still pseudonymous personal data; it needs a retention period. |
| "`deleted_at` is set, so the user is deleted" | Masking or flagging is not erasure (EDPB 2025 erasure report). The data is still in the row, the backups and the processors. |
| "The audit log and versions are immutable, so we cannot delete" | Keep personal data out of the immutable rows; delete the mapping and the history stays intact (EDPB 02/2025). |
| "DPDP says keep logs a year, so we keep everything a year" | DPDP r.8(3) is a minimum for specified purposes, in force with the Rules from 13 May 2027; for EU users an Indian duty is not an Art. 17(3)(b) ground. State each exception; ask the lawyer. |
| "DPDP is not in force until 2027, India can wait" | IT Rules, CERT-In Directions and SPDI Rules are in force now. |
| "We already have a privacy policy" | Check it against the inventory row by row; real audits find omitted IPs, missing processors and promises the schema contradicts. |
| "We are too small for GDPR" | Art. 3(2) has no size threshold; the Art. 30 exemption is lost when processing is not occasional. CCPA does have thresholds: check them, do not assume either way. |
| "Put a cookie banner on everything to be safe" | Consent for exempt storage is noise. Build consent only for non-exempt storage and scripts. |
| "`DELETE FROM users` will do" | `NO ACTION` foreign keys fail it, cascades delete other people's content, and immutability triggers block it. Run the `references/erasure-patterns.md` §1 queries first. |
| "Put a placeholder grievance officer and fix it later" | A placeholder that ships looks finished. Ask for the real name and contact; block the unit until you have it. |
| "The code looks right, the drill is overkill" | The drill is the evidence the report cites. Without it the control is a claim. |
| "Say we are GDPR compliant on the page" | Never. Describe what the controls do. |

## Guardrails

- ⛔ Not legal advice; never claim the product is compliant, safe or secure. Stated in Phase 0,
  in every report, and checked by the skeptic.
- ⛔ No edit before the Phase 4 approval; only approved units and files after it.
- ⛔ Never invent the company's legal facts; ask.
- ⛔ Never weaken an immutability trigger or append-only rule without an approved, recorded
  decision; erasure works around immutable rows, not through them.
- ⛔ Never delete production data during a drill; drills use test users, an isolated scratch
  database for backup restores (asked first, destroyed after), and the provider's test mode where
  one exists (asked first where none does).
- ⛔ Never copy production personal data into the ledger, a test fixture or a report.
- ⛔ Never push; ask about push and PR together.

## Definition of done

- [ ] Ledger read first; resumed rather than restarted; not-legal-advice statement given.
- [ ] Data map covers schema, forms, cookies/storage, logs, scripts, processors, email, uploads and
      immutable data, each row cited to `file:line` or marked "none found".
- [ ] Applicability table filled, with dates from `references/regimes.md` and its verified date
      checked (re-verified if older than 90 days or a deadline is within 90 days).
- [ ] Gap report ranked L / D / S, with an erasure-blockers list and lawyer questions.
- [ ] The user approved the plan before any edit; only approved units were built.
- [ ] Every notice statement traces to an inventory row and every row appears in the notice.
- [ ] Drills D1–D8 run for every built unit (a skipped drill names its unapproved unit), with
      commands and output in the ledger; all pass.
- [ ] The erasure design passed the skeptic before its migration was written.
- [ ] No page, commit or report claims compliance or safety; no placeholder legal facts shipped.
- [ ] Skeptic score ≥95, lower score taken; gaps it named fixed or put to the user.
- [ ] Committed (only this work, project author); push and PR asked, not done.

---

## Appendix A — `PRIVACY-LEDGER.md` template

```markdown
# Privacy ledger
> Engineering gap analysis, not legal advice. Nothing here certifies compliance.

- Run date: <date> · regimes.md last verified: <date> · Status: inventory | applicability | gaps | approved | building | drills | done
- Assumptions (for the lawyer): <where users are, intermediary?, children?>

## Data map
| Field | Purpose | Source | Store | Processor | Retention | Basis | Exported? | Erased? |

## Applicability
| Regime | In force | Applies? | Why / threshold evidence |

## Gaps
| # | Gap | Obligation | Evidence | Rank | Fix | Owner | Blocks erasure? | Status |

## Facts needed from the user
- [ ] Legal entity + address · [ ] Grievance officer · [ ] EU/UK representative · [ ] Governing law · [ ] DMCA agent

## Lawyer questions
1. ...

## Build units (approved <date>)
- [ ] U1 <unit> — files · drill · rollback

## Drill evidence
| Drill | Date | Command | Result |

## Skeptic
| Round | Self | Skeptic | Final | Gaps named |
```
