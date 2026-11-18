# Erasure patterns: deleting a person when the data model says "never delete"

Approved versions, activity logs, ledgers and moderation records are often append-only on
purpose; that is what makes them trustworthy. Erasure must work **around** those rows, not
through them. The design below follows EDPB guidance: personal data stays out of the immutable
layer, so deleting it leaves the immutable history intact and no longer about anyone [19], and
masking alone is not deletion [18].

## 1. Find what blocks a delete (Postgres; adapt for other databases)

```sql
-- Every foreign key that points at the user table, with its ON DELETE action
SELECT c.conrelid::regclass AS from_table,
       a.attname           AS from_column,
       c.confrelid::regclass AS to_table,
       CASE c.confdeltype WHEN 'a' THEN 'NO ACTION' WHEN 'r' THEN 'RESTRICT'
                          WHEN 'c' THEN 'CASCADE'  WHEN 'n' THEN 'SET NULL'
                          WHEN 'd' THEN 'SET DEFAULT' END AS on_delete
FROM pg_constraint c
JOIN pg_attribute a ON a.attrelid = c.conrelid AND a.attnum = ANY (c.conkey)
WHERE c.contype = 'f' AND c.confrelid = 'public.user'::regclass   -- change to your user table
ORDER BY on_delete, from_table;

-- Every trigger, so immutability and append-only rules are on the list
SELECT event_object_table AS table_name, trigger_name, action_timing, event_manipulation
FROM information_schema.triggers
WHERE trigger_schema = 'public'
ORDER BY table_name, trigger_name;

-- Rewrite rules (e.g. DO INSTEAD NOTHING on DELETE) and row-level security policies, which also
-- block or hide deletes without being triggers
SELECT tablename, rulename, definition FROM pg_rules WHERE schemaname = 'public';
SELECT tablename, policyname, cmd, qual FROM pg_policies WHERE schemaname = 'public';
```

On MySQL, read `information_schema.REFERENTIAL_CONSTRAINTS` (DELETE_RULE) and
`information_schema.TRIGGERS`; on SQLite, `PRAGMA foreign_key_list(<table>)` and
`sqlite_master WHERE type = 'trigger'`.

Repeat the first query for any profile/account table that also holds personal data. Look for
circular references (user → personal account → user), sole-owner rules that stop a user leaving
an organisation, and row-level security policies that hide rows from the erasure job.

Classify every referencing table:

| Class | Example | Erasure action |
|---|---|---|
| Owned personal data | profile, sessions, devices, OAuth links, notification prefs | Delete (CASCADE or explicit). |
| Owned content, not immutable | drafts, private notes, uploads | Delete, including object storage. |
| Shared content others depend on | published packages, comments in a thread, org resources | Keep the content; reassign authorship to the tombstone; decide (product) whether published content goes too. |
| Immutable / append-only history | approved versions, review decisions, activity log, ledger | Keep the row; it must reference only an opaque actor id. |
| Personal data inside immutable content | an email typed into a published body | Needs an append-only redaction decision and an approved exception in the immutability rule: a product and legal decision, never improvised. |

## 2. The opaque-id tombstone design

1. **Immutable rows reference an opaque actor id only** (a random UUID, not the email, handle or
   a hash of either). If they currently store names, emails, IPs or user agents, move those into
   a deletable side table keyed by the actor id, in a migration approved at the gate.
2. **Personal data lives in deletable tables** (user, profile, sessions), or is encrypted under a
   per-user key held in a deletable table; destroying the key erases the ciphertext everywhere it
   was copied, including backups (crypto-shredding).
3. **On erasure:** delete the personal rows and the per-user key; point the actor id at a
   "deleted user" tombstone (the id stays, the person is gone); delete the mapping from the person
   to the actor id. **Keep no email hash** to "block re-registration": a hash of a known email is
   still personal data. If abuse prevention truly needs it, it is a stated retention exception
   with a period, a keyed hash, and a lawyer's sign-off.
4. **Make processors erase**: email provider contacts and logs, object storage, search index,
   analytics profiles, error tracker events, CDN-cached avatars, auth provider records. Use each
   provider's delete API where one exists; record the request and response.
5. **Erasure register**: an append-only table of erasure requests (opaque request id, actor id,
   requested and completed timestamps, scope). It holds no personal data. After **any** backup
   restore, re-apply every completed erasure before the restored database serves traffic [18].
6. **Retention exceptions**, each stated in the notice and the deletion help text with its period
   and basis, then deleted by a job when it expires:

| Exception | Period | Basis | Caveat |
|---|---|---|---|
| Registration data of a deleted account (intermediary) | 180 days after deletion | IT Rules r.3 [7] | India intermediary duty; for EU users not an Art. 17(3)(b) ground (lawyer question). |
| Removed content and associated records | 180 days | IT Rules r.3 [7] | As above. |
| ICT logs | rolling 180 days | CERT-In Directions [8] | Log location question (CERT-In FAQ Q35). |
| Processing logs and associated personal data | at least 1 year | DPDP r.6(1)(e), r.8(3) [1] | From 13 May 2027; a minimum for specified purposes, not a licence to keep everything. |
| Financial / tax records | per tax law | local law | Keep minimal fields; drop profile data. |
| Legal hold | until released | legal claims, GDPR Art. 17(3)(e) [13] | Record who placed it and why. |
| Email suppression entry | while the address could be mailed | opt-out must keep working | Keep only what the provider needs; say so in the notice. |

## 3. The deletion flow (user-facing)

1. Settings → Delete account. Explain, before confirming: what is deleted, what is kept, why, and
   for how long; what happens to published or shared content; that sign-in stops immediately.
2. Re-authenticate (recent sign-in) and confirm. Offer an export first.
3. Immediately: revoke sessions, tokens and devices; block sign-in; hide the profile.
4. A job completes the erasure within the published period (state it; GDPR requires erasure
   without undue delay and an answer within one month, extendable by two months for complex
   requests [13]), then sends a confirmation to the address it is about to
   delete, or shows it in the session that requested it.
5. Staff-initiated erasure (a rights request by email) goes through the same job, with identity
   verified proportionately and the response including the r.9 contact (DPDP) [1].

Sole owners of an organisation: require transfer or deletion of the organisation first, and say
so in the confirmation step rather than failing silently.

## 4. Export (access and portability)

- Generate from the inventory: every row whose subject is the user, in JSON (machine-readable),
  plus their uploads. Include the list of processors and other fiduciaries their data was shared
  with (DPDP access right [2 s.11]).
- Exclude other people's personal data (other commenters' names in a thread: keep their opaque
  ids or "another user").
- Deliver through the signed-in session or a short-lived signed link; rate-limit; log the export
  without logging its contents.

## 5. Keyed hashing for IPs and identifiers

- Use HMAC-SHA-256 with a server-held secret, never a bare hash; for analytics or rate-limit
  windows, rotate the secret daily and discard old ones so values cannot be linked across days.
- A keyed hash is still pseudonymous personal data: it needs a purpose, a retention period and a
  row in the data map.
- Truncation (e.g. dropping the last IPv4 octet) before hashing reduces linkage further when the
  purpose allows.

## 6. Correction

Correction includes the email address: re-verify the new address, update every processor copy
(email provider contact, billing), and keep no history of old addresses beyond a stated security
retention.
