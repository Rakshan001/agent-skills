---
name: site-search
description: >-
  Build or fix the search INSIDE a website or app over its own data (products, docs, posts,
  users, tickets) — right-sized engine, tested relevance, measured latency, approval before any
  change. Use when the user says "add search", "build a search engine", "search bar", "search
  page", "site search", "product search", "search as you type", "autocomplete for our products",
  "typo tolerance", "fuzzy search", "full-text search", "search is slow", "search results are
  bad", "make search like Algolia", "Postgres full-text", "SQLite FTS", "Elasticsearch vs
  Meilisearch vs Typesense", or "/site-search". NOT for: crawling or scraping other websites,
  searching the agent's own codebase or memory, LLM retrieval (RAG) pipelines, autocomplete over
  a fixed short list or a third-party API (address/places lookup), a fuzzy picker in a CLI, or a
  slow query that has nothing to do with search.
---

# Site Search — the right results, with the least necessary work

Most site search fails one of two ways: it is **naive** (`LIKE '%q%'`, load every row and filter,
sort by date) or it is **over-built** (an Elasticsearch cluster and a hand-written trie for
3,000 rows). Both usually ship with a speed claim nobody measured. This skill makes the engine
choice, the relevance and the numbers **evidence-based**.

```
LEDGER CHECK (SEARCH.md? resume) → TRACKER CHECK (optional) →
1 ASSESS: inventory entities · fields · access rules · corpus size AT PRODUCTION SCALE ·
          current search measured · latency metric DEFINED →
2 GOLDEN SET: 20–50 real queries with expected results (the relevance baseline) →
3 DESIGN: engine ladder (lowest rung that fits) · per-field text pipeline · ranking ·
          authz-in-query · sync · API · caching (only if needed) · cost →
⛔ APPROVAL GATE — nothing is edited, migrated or provisioned before a yes ⛔ →
4 BUILD unit by unit (index → query path → sync → API → UI), each tested, each ticked →
5 MEASURE: golden-set relevance + benchmark at production scale (p50/p95/p99, concurrency) →
⛔ ≥95% SKEPTIC GATE (authz holds · relevance real · every number measured) ⛔ →
REPORT (measured vs estimated) → COMMIT (auto) → ⛔ ASK BEFORE PUSH + PR ⛔
```

**Golden rules**

0. **RESUME FIRST, RECORD AS YOU GO.** Read `SEARCH.md` before anything; tick each unit as it lands.
1. **MEASURE BEFORE CHOOSING.** Corpus size, growth, query mix and the current p95 decide the
   engine. A decision made before counting the rows is a guess.
2. **CLIMB THE LADDER ONLY ON EVIDENCE.** Each rung up adds a service, a sync pipeline, memory and
   a bill. Move up only when the lower rung *measurably* misses a stated requirement.
3. **THE ENGINE ALREADY HAS THE DSA.** Inverted indexes, relevance scoring, prefix lookup,
   typo tolerance and top-K selection are built into most engines on rungs 1–3 (table 3b says
   which has what). Hand-write one only to fix a measured gap no engine closes — never to
   demonstrate the concept.
4. **RELEVANCE IS TESTED, NOT EYEBALLED.** A golden set of queries with expected results, scored
   before and after. One query proves nothing; special-casing a document to win one query is
   overfitting.
5. **PERMISSIONS ARE A QUERY FILTER, NOT A POST-FILTER.** Tenant and visibility rules go inside
   the engine query so hits, counts, facets, suggestions and snippets all respect them.
6. **EVERY NUMBER IS MEASURED OR LABELLED.** A latency without a defined metric, dataset size,
   hardware and concurrency is not a result. A placeholder (`[fill in]`) in a report is a defect.
7. ⛔ **APPROVAL BEFORE THE FIRST CODE EDIT** (only the ledger, golden set and scorer come
   earlier) — including "just build it" and "it's a small change".
8. **COMMIT AUTOMATICALLY, NEVER PUSH** — unless the project's instructions say to ask before
   committing; then ask. Push and PR only on an explicit yes.

> **Adapt to the project.** This skill is stack-agnostic. "The database", "the engine", "the test
> command" mean the project's real ones. Steps marked *(optional)* are skipped when the project
> lacks that thing — never invent infrastructure the project does not have. Engine-specific syntax,
> defaults and gotchas live in `references/engines.md`; read the section for the engine you choose
> **before** writing its queries, and confirm version-specific details against current docs.

**Autonomy contract.** Unattended, this skill reads code, queries a *local* database, writes the
ledger + golden set + scorer before approval, seeds a *local* benchmark dataset, writes code and
tests after approval, and commits. Besides the clarifying questions in Phase 1, it **stops** for:
⛔ any query against a non-local database (reads included — `EXPLAIN ANALYZE` executes the query)
· ⛔ the design approval · any non-local migration or index build · any new service, hosted plan
or recurring cost · a skeptic score below 95% it cannot close · three failed attempts at one step
· push / PR.

---

## Phase 0 — Ledger and tracker

1. **Read `SEARCH.md`** at the repo root (template in Appendix A). If it exists, continue from the
   first unticked item. Trust the code over the ledger when they disagree; fix the ledger and say so.
2. *(optional)* **Shared tracker** — if the project has one (Baton: `recall_memory "search"`,
   `list_signals`, `check_files` on the search files), check whether search work already exists,
   is stuck on a branch, or is being edited live. If so, STOP and surface it.

---

## Phase 1 — Assess (read-only)

Produce the **assessment** below in the ledger. Separate **confirmed** (seen in code/data) from
**assumed** (state the assumption and how to confirm it).

1. **Stack:** frontend, backend, database, ORM, hosting, memory limit, existing search code and
   every caller of it.
2. **Searchable inventory** — one row per entity:

   | Entity | Fields searched | Field kind (name/id/prose/tag) | Filters / sorts | Who may see it |
   |---|---|---|---|---|

   "Who may see it" is mandatory: public, per-tenant, per-role, owner-only, soft-deleted, draft.
   An entity with no stated visibility rule is a finding, not a blank.
3. **Corpus size at production scale.** Count rows (`SELECT count(*)`, grouped by tenant when
   multi-tenant). ⛔ Against a non-local database, ask first — or ask the user for the numbers. Record
   current size and expected size in 12 months. **Dev data is not the corpus** — 50 seeded rows
   predict nothing about 500,000.
4. **Current search, measured.** Run its query on representative input — locally, or on a
   non-local database only after asking, since `EXPLAIN ANALYZE` executes it; capture the plan (or
   the engine's profile) and timings. Name the bottleneck you can *see*: full scan, unbounded
   result set, N+1, in-memory filtering, missing index, no limit.
5. **Requirements, explicitly.** For each, mark *required / nice / not needed*: exact match, prefix
   (search-as-you-type), full-text, typo tolerance, phrase, filters, facets, sorting, highlighting,
   synonyms, multi-language, semantic. Do not assume semantic or personalised ranking; ask.
6. **Define the latency metric** — which of these the target applies to, in writing:

   | # | Timing | How it is measured |
   |---|---|---|
   | 1 | Engine execution | engine-reported time / `EXPLAIN ANALYZE` execution time |
   | 2 | Backend processing | server-side span around the handler, excluding network |
   | 3 | Engine/DB round trip | span around the client call |
   | 4 | API end-to-end | load tool against the endpoint, same region |
   | 5 | Browser-visible | keystroke → results painted (includes debounce + network) |

   A "10 ms" target is normally #1 or #2. Say which, and say plainly that #5 will be higher.
7. **Unanswered questions** — list them. Ask the ones that change the design now; record the rest
   as assumptions.

---

## Phase 2 — Golden query set (the relevance baseline)

Write `search-golden.json` (or the project's fixture format) with **20–50 queries**:

```json
[
  { "q": "install",        "expect_top": ["docs/getting-started/install"], "k": 3 },
  { "q": "wireles headphn", "expect_any": ["sku-1182", "sku-2290"],         "k": 5, "kind": "typo" },
  { "q": "SKU-1182",       "expect_top": ["sku-1182"],                       "k": 1, "kind": "exact-id" },
  { "q": "zzqx",           "expect_none": true,                              "kind": "no-result" }
]
```

- **Source the queries from reality**: search logs or analytics if they exist, else the top
  entities by traffic, else ask the user for 10 and derive the rest. Cover exact names, IDs/SKUs,
  prefixes, typos, multi-word, filtered queries, no-result queries, and at least one query per
  permission level (which must NOT return the protected item to the wrong viewer).
- **Hold out ~20%** of the set. Tune ranking on the rest; report the held-out score separately.
  A ranking that only wins on queries you tuned against is overfit.
- **Write the scorer**: a small test or script in the project's own test runner that runs every
  golden query through the search function (or the existing endpoint, if the logic is not yet a
  callable function — refactoring waits for approval) and checks hit@k, `expect_none` and the permission
  cases. Record its exact command in the ledger — the same command produces every later score.
- **Score** the current search now: hit@k per query and the overall pass rate. This is the
  "before" number.

---

## Phase 3 — Design

### 3a. The engine ladder — stop at the first rung that meets every *required* item

| Rung | Engine | Fits when | Costs you |
|---|---|---|---|
| 0 | **Client-side index** (Pagefind for static sites; MiniSearch / FlexSearch / Orama in the browser) | Corpus is public, small enough to ship, changes at deploy time | Payload size; zero servers |
| 1 | **Database full-text** (Postgres tsvector + GIN, `pg_trgm` for typos/prefix; SQLite FTS5; MySQL FULLTEXT; MongoDB Atlas Search on Atlas) | Data already lives there; typical catalogs and docs up to millions of rows | An index + migration; no new service; consistency is transactional |
| 2 | **Instant-search engine** (Meilisearch, Typesense; hosted Algolia) | Typo tolerance + facets + search-as-you-type quality that rung 1 measurably misses | A service, a sync pipeline, RAM (Typesense holds the index in memory), recurring cost |
| 3 | **Elasticsearch / OpenSearch** | Heavy aggregations, very large corpora, complex per-language analysis, an ops team already running it | A cluster: memory, upgrades, shards, on-call |
| 4 | **Custom index in app code** | A measured requirement none of the above meet | You own correctness, persistence, concurrency, memory |

Write one line per rung you **rejected** and why (requirement it misses, or cost it adds). Never
combine two engines without a requirement only the combination meets.

**Semantic / vector search is an add-on, not a rung.** Start lexical. Add embeddings (pgvector,
or the chosen engine's hybrid mode) only when a class of golden-set queries — intent or synonym
queries — fails lexically and a synonym list does not fix it. Document embeddings are computed
at index time; only the query embedding is computed on the request path, and its latency counts.

### 3b. Data structures — who implements what

| Concept | Where it already lives | Write it yourself only when |
|---|---|---|
| Inverted index, postings, TF/DF, doc length | Every rung 1–3 engine | Never, unless rung 4 |
| BM25 | FTS5 `bm25()`, Elasticsearch/OpenSearch (default); ParadeDB `pg_search`. **Not** Postgres `ts_rank`, Meilisearch (ranking-rule bucket sort) or Typesense (own text-match score) | Rarely — tune weights first |
| Field weighting | All of the above; Postgres via `setweight` A–D | Never |
| Prefix lookup (trie/FST) | FTS5 `prefix=`, Postgres `:*` / trigram, engine prefix search | Never for site search |
| Typo tolerance (bounded edit distance / trigram similarity) | `pg_trgm`, Meilisearch, Typesense, Elasticsearch `fuzziness`. **Not** FTS5 (trigram tokenizer is substring only) or MySQL FULLTEXT | Never compare the query against every term |
| Top-K (bounded heap) | `LIMIT` / engine top-K | Only if ranking happens in app code over a large candidate set |
| Boolean AND/OR/NOT, posting intersection | Engine query parser | Never |
| Graph / PageRank / BFS | Not part of site search | Only for a stated crawl or link-ranking requirement |
| Hash maps, caches | App layer | For metadata lookup and a cache that a measurement justifies |

Fuse.js-style fuzzy libraries **scan every record on every query** — fine for a few thousand
short items in a command palette, wrong as a site search index. Say which you are using and why.

### 3c. Text pipeline, per field

For each searched field decide: case folding, Unicode/accent normalization, tokenizer, stemming,
stop words, prefix, typo tolerance. **Identifiers, SKUs, codes, emails and product names usually
get NO stemming and NO stop-word removal** — "iPhone 15 Pro" must not become "iphon 15 pro" and
lose exact matches. Index-time and query-time processing must be identical. Check where the engine
lets you set this: some take it per field (Postgres per `to_tsvector` call, rung 2–3 engines),
some only per index (an FTS5 tokenizer applies to the whole table).

### 3d. Ranking

Field weights (name/title ≫ tags/brand ≫ description), exact match and prefix boosts, then
business signals (in stock, popularity, freshness) as **tie-breakers or bounded boosts** — a
popularity signal must not push an irrelevant item above a relevant one. Add a stable final sort
key (id) so pagination never duplicates or skips items.

### 3e. Permissions and query safety ⛔ never trade these away

- Tenant / visibility / role filters are **in the engine query**, server-side, from the session —
  never from a client parameter. Post-filtering breaks pages and leaks through `totalHits`,
  facet counts, suggestions, snippets and highlighting.
- Browser-direct engines (rung 2): the browser gets only a **search-only, scoped** key or tenant
  token with the filter baked in. The admin/master credential never leaves the server.
- Cache keys include tenant **and** permission scope. A shared cache must never serve one
  viewer's results to another.
- Escape or quote user input before it reaches query syntax (FTS5 `MATCH`, `to_tsquery`,
  Lucene `query_string`) — see `references/engines.md`. Cap query length, `limit` and page depth.
- Errors return a generic message; engine errors, internal fields and index names stay in logs.

### 3f. Keeping the index in sync

- **Rung 0:** rebuilt at deploy. **Rung 1:** generated column or trigger in the same transaction —
  no sync problem; still test update and delete.
- **Rung 2–3:** never an uncoordinated dual write as the only mechanism. Use a **transactional
  outbox** (or the queue/CDC the project already has): write the change + an outbox row in one
  transaction, a worker upserts by id **idempotently** (with an `updated_at`/version guard so an
  old event cannot overwrite a newer document), deletes are explicit events, retries back off,
  and a **reconciliation job** compares ids + versions and repairs drift. Full reindex builds a new
  index and **atomically swaps** an alias, so a failed reindex never empties live search.
- With the semantic add-on, embeddings are part of the document: re-embed on change, remove on
  delete, through the same sync path — and a changed embedding model means a full reindex.
- State what a user sees during the lag (e.g. "an edited price can show the old value for up to
  N seconds; checkout always reads the database").

### 3g. API, frontend, caching

- API (or the project's own convention — GraphQL, tRPC, server actions):
  `GET /api/search?q=&filters=&sort=&page=&limit=` and, if prefix is required,
  `GET /api/search/suggest?q=`. Validate every parameter; cap `limit`; return only the fields the
  UI renders; include `processingTimeMs` **named for the timing it is** (Phase 1 table).
- Frontend: debounce ~150 ms, cancel stale requests (`AbortController`) and ignore out-of-order
  responses, loading / empty / error states, keyboard navigation with the ARIA combobox pattern,
  URL sync for the results page.
- Caching *(optional)*: only after a benchmark shows it is needed. Define key, TTL, invalidation,
  scope isolation and stampede protection. A cache must not hide a slow query.

### 3h. The proposal

Write it to the ledger and present it: recommended rung and rejected rungs · request-flow and
indexing-flow diagrams (ASCII) · per-field pipeline · ranking · permission enforcement · sync ·
API shape · migration and rollback steps · **recurring cost and memory** (look up current prices;
never quote from memory) · the point at which this design must evolve (e.g. "past ~N rows or
M qps, move to rung 2") · the benchmark plan · prioritised build units. Any latency you expect
before measuring is written as **"estimate:"** — the proposal promises a measurement, not a number.

---

## ⛔ APPROVAL GATE ⛔

**Present the assessment, the golden-set "before" score and the design, then STOP. Until the
user explicitly approves, the only files you may write are the ledger, the golden set and its
scorer. Do not edit any other file, run any migration, build any index or provision any
service.** This applies to every case, including "just build it", a one-line query change, and a
demo tomorrow — urgency is when an unreviewed migration does the most damage. If the user requests
changes, revise and re-present. Record the approval (and what was approved) in the ledger.

---

## Phase 4 — Build, unit by unit

Order: **index/migration → query function → sync → API → UI *(optional)* → observability
*(optional)***, repeated **per entity** in the inventory — the ledger gets one row per entity per
unit, so no entity is silently dropped (work shared across entities — one federated endpoint,
one search bar — gets a row named `shared`). For each unit:

1. Write the test first — correctness, ranking, filters, **permission isolation**,
   insert/update/delete reflected in results, malicious and empty input. Skipping a test needs a
   written reason in the ledger.
2. Implement the smallest change that passes. Reuse the project's existing helpers, ORM, error
   types and logging.
3. Run the project's tests. Tick the unit in `SEARCH.md` with its commit or file list.

Migrations: avoid long table locks on large tables (Postgres: `CREATE INDEX CONCURRENTLY`, which
cannot run inside a transaction — many migration tools wrap one by default; and adding a
`STORED` generated column rewrites the table — see `references/engines.md`). Write the
down-migration, and ⛔ **ask before running anything against a non-local database.**

⛔ **Bounded retries.** If one unit fails three times, do not attempt a fourth. Three failures
mean the design assumption is wrong (wrong tokenizer, wrong engine for this query shape, wrong
sync model). STOP, write what you now believe in the ledger, and decide with the user.

---

## Phase 5 — Measure

### 5a. Relevance
Re-score the golden set: hit@k per query, overall pass rate, **held-out pass rate**, and every
permission query (must be 100%). Fix ranking with field weights and analysis choices — never by
special-casing a document, slug or id.

### 5b. Performance
1. **Dataset at production scale.** If local data is smaller than the Phase 1 figure, seed
   realistic synthetic data to that size (and to the 12-month figure).
2. **Query mix** from the golden set plus logs: exact, prefix, full-text, typo, filtered, no-result.
3. Measure **cold** (first queries after an engine/DB restart, OS caches dropped where
   possible) and **warm** (after a warm-up pass) separately.
4. **Concurrency**: at least 1 and the expected peak concurrent searches.
5. Measure the timing the target was defined on, plus #4 end-to-end. Tools: the engine's own
   timing, `EXPLAIN ANALYZE`, and a load tool the project has or can run without installing into
   the app (e.g. `autocannon`, `k6`, `oha`, `wrk`).
6. Record p50 / p95 / p99, throughput, error rate, CPU and memory, index size, indexing
   throughput, and the hardware. Save the **raw tool output** under a stated path (e.g.
   `bench/results/<date>-<change>.txt`) and the exact benchmark command in the ledger, so a third
   party can re-run it and check the numbers.

Missed the target? Profile, form one hypothesis, change one thing, re-measure. Record each attempt.

---

## ⛔ Phase 6 — ≥95% skeptic gate

1. Score your own confidence, as a percentage with one line of evidence per claim, that **(a)** no
   viewer can see a result, count, facet, suggestion or snippet they are not allowed to see,
   **(b)** the relevance scores are real and not overfit, **(c)** every number in the report was
   measured as described, and **(d)** create/update/delete are reflected in results.
2. Spawn an independent read-only reviewer (fresh context, no write tools). Give it the diff, the
   ledger, the golden set and the raw benchmark output. Instruct it to **refute** you — find a
   leaking path, a query the golden set does not cover, an unmeasured number, an untested delete —
   and to return **its own percentage for each of (a)–(d)** plus the specific defects.
3. **Final confidence = the LOWER of the two**, per claim.
4. ≥95% on all four → report. <95% → fix the named gaps, re-run the affected measurement, and
   **re-run the reviewer**. Cannot close a gap because something is genuinely ambiguous →
   ⛔ STOP and ask the user.

This gate is not waived because tests are green: green tests usually mean the tests miss the
leaking case.

---

## Phase 7 — Report, commit, ask

**Report** (also written to the ledger), with **measured** and **estimated** visibly separated:

```
Engine: <rung + engine>   Rejected: <rungs + one-line reasons>
Corpus: <N docs measured / seeded>   Hardware: <cpu, ram, where the engine ran>
Relevance: golden <before>% → <after>% · held-out <x>% · permission queries <y>/<y>
Latency (<which timing>), concurrency <c>: p50 <a> ms · p95 <b> ms · p99 <c> ms · <qps> qps
End-to-end API: p50 … p95 …   Errors: <rate>   Index: <size>, built in <time>
Target <10 ms on <timing>: MET / NOT MET (why, and the next lever)
Not measured: <anything — and why>
Security fixes found on the way: <e.g. results were not tenant-filtered>
Recurring cost: <amount + source> · Evolve when: <threshold>
```

Never send a report with a placeholder. If a number was not measured, write **not measured**.

Then **commit** only this work's files, with a conventional message (`feat(search): …`) under the
project's git identity — or, if the project's instructions say to ask before committing, ask.
⛔ **Do not push.** Ask about push **and** PR together, including the base branch. *(optional)*
Record the decision and the measured numbers to the shared tracker's memory.

---

## Red flags — stop when you catch yourself thinking these

| Thought | Reality |
|---|---|
| "Just build it — the user said so, the demo is tomorrow" | Approval gate. A five-minute review beats a migration you have to reverse during the demo. |
| "If X still outranks Y, I'll demote X's slug / boost that id" | That fixes one query and breaks unknown others. Fix weights or analysis and re-score the golden set. |
| "Re-running the one bad query proves the fix works" | One query proves one query. Score the full golden set and the held-out slice. |
| "Timings: [fill in from the run]" | A placeholder is not a result. Measure it or write "not measured". |
| "I benchmarked on my local data" (row count unknown) | Seed to the production count from Phase 1. A 50-row benchmark predicts nothing. |
| "Porter stemming everywhere is fine" | Not on names, SKUs, codes or API identifiers. Decide per field. |
| "Elasticsearch is the industry standard" | Standard for its scale. Show the measured gap at the lower rung first. |
| "I'll write an inverted index / trie / heap — the user asked for DSA" | The engine already implements them. Write the 3b table; build only for a measured gap. |
| "Filter private items after fetching" | Counts, facets, pages and suggestions still leak. Filter inside the query. |
| "It's under 10 ms" | Which timing, what corpus, what concurrency, which percentile? |
| "It's late — I'll send results now and offer to finish the checks later" | You may stop, but the ledger says *in progress* and the message lists what is unverified. Unmeasured or unchecked is never reported as done. |
| "Add Redis to make it fast" | Only after a benchmark shows the query is already right and still too slow. |

---

## Guardrails (always enforced)

- ⛔ No code edit, migration, index build or new service before the design is approved (ledger, golden set and scorer excepted).
- ⛔ Never run any query, migration or index build against a non-local database without asking.
- ⛔ Permission and tenant filters live inside the query; the admin credential never reaches the browser.
- ⛔ Never claim a latency or relevance number you did not measure; never send placeholders.
- ⛔ Never add a recurring cost (hosted plan, new server, cluster, embedding API) without the user's yes and a stated price.
- ⛔ Never special-case documents to pass the golden set.
- ⛔ Never remove or rewrite working search without stating the impact and the rollback.
- ⛔ Commit automatically (or ask, if the project says so); never push without explicit permission.

---

## Definition of done

- [ ] `SEARCH.md` read first; status, branch, scorer command and benchmark command recorded; one ticked row per entity per build unit.
- [ ] Assessment written: inventory with a visibility rule per entity, production-scale corpus size, current search measured, requirements marked, latency metric defined, confirmed vs assumed separated.
- [ ] Golden set of 20–50 real queries with held-out slice and permission queries; "before" score recorded.
- [ ] Design records the chosen rung and each rejected rung with a reason, the per-field pipeline, ranking, in-query permission enforcement, sync model, cost, and the evolve-at threshold.
- [ ] Approval recorded before the first code edit.
- [ ] Tests cover ranking, filters, permission isolation (hits, counts, facets, suggestions), insert/update/delete, malicious and empty input; the suite passes.
- [ ] Golden set re-scored (overall + held-out + permission queries at 100%).
- [ ] Benchmark at production scale: hardware, corpus, concurrency, p50/p95/p99, throughput, errors, memory — on the defined timing; raw output saved at the recorded path.
- [ ] Independent skeptic re-check run (re-run after every fix); lower score per claim taken; ≥95% on all four.
- [ ] Report has no placeholders and separates measured from estimated.
- [ ] Committed (or asked, per project rules); push and PR asked together with the base branch.

---

## Appendix A — `SEARCH.md` ledger template

```markdown
# Search ledger — <project>

Status: assessing | designing | awaiting approval | building | measuring | done
Branch: <branch>   Updated: <date>
Scorer: `<exact command>`   Benchmark: `<exact command>` → raw output in <path>

## Assessment
- Stack: …            - Corpus: <now> / <12 mo> (source: …)
- Latency metric: <timing #> target <x> ms at p95, concurrency <c>
- Requirements: exact ✓ prefix ✓ typo ✓ facets ✗ …
- Confirmed: …        - Assumed (+ how to confirm): …
- Open questions: …

| Entity | Fields | Kind | Filters/sorts | Who may see it |
|---|---|---|---|---|

## Golden set
File: search-golden.json · <n> queries · held-out <m>
Before: <x>% (held-out <y>%)

## Design
Rung: … · Rejected: … · Pipeline per field: … · Ranking: … · Permissions: … · Sync: … · Cost: …
Approved: <date, by whom, what>

## Build units (one row per entity)
| Entity | Index/migration (+down) | Query fn | Sync | API | UI (opt.) | Observability (opt.) |
|---|---|---|---|---|---|---|
| <entity> | [ ] | [ ] | [ ] | [ ] | [ ] | [ ] |

## Measurements
| Run | Change | Corpus | Conc. | p50 | p95 | p99 | Golden | Held-out |
|---|---|---|---|---|---|---|---|---|

## Skeptic (re-run after every fix)
| Claim | Self | Independent | Final (lower) |
|---|---|---|---|
| (a) no permission leak | | | |
| (b) relevance real, not overfit | | | |
| (c) every number measured | | | |
| (d) create/update/delete reflected | | | |
Gaps closed: …
```
