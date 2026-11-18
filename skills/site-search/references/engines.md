# Engine reference — syntax, defaults and gotchas

Read only the section for the engine you chose. **Freshness:** written October 2026. Defaults
and syntax change between versions — confirm anything version-specific against the docs for the
version the project actually runs before relying on it.

---

## Postgres (rung 1)

**Full-text column + index**

```sql
ALTER TABLE product ADD COLUMN search tsvector GENERATED ALWAYS AS (
  setweight(to_tsvector('simple',  coalesce(name, '')),        'A') ||
  setweight(to_tsvector('simple',  coalesce(brand, '')),       'B') ||
  setweight(to_tsvector('english', coalesce(description, '')), 'C')
) STORED;                                    -- Postgres 12+
CREATE INDEX CONCURRENTLY product_search_idx ON product USING gin (search);
```

- **Locks.** Adding a `STORED` generated column rewrites the whole table under an `ACCESS
  EXCLUSIVE` lock. That's fine for small tables. For large ones, add a plain `tsvector` column,
  keep it current with a trigger, and backfill it in batches.
- **`CREATE INDEX CONCURRENTLY` cannot run inside a transaction**, and many migration tools wrap
  each migration in one by default. Use the tool's opt-out, for example Django `atomic = False`
  with `AddIndexConcurrently`, or Rails `disable_ddl_transaction!` with
  `algorithm: :concurrently`. For other tools, check their docs or run the statement as a
  separate non-transactional step. A failed concurrent build leaves an `INVALID` index: drop it
  and retry.

- `'simple'` = no stemming, no stop words → right for names, brands, codes. `'english'` stems →
  only for prose. Pick per field (SKILL.md §3c).
- Query with `websearch_to_tsquery('english', $1)` or `plainto_tsquery` — both accept raw user
  text. **`to_tsquery` raises a syntax error on raw input**; build it only from sanitized tokens
  (e.g. prefix `'wire':*`).
- `ts_rank` / `ts_rank_cd` use the A–D weights but **are not BM25** (no corpus-wide IDF). If BM25
  is a hard requirement, evaluate the ParadeDB `pg_search` extension — and check that the host
  allows installing it.
- `unaccent()` is not immutable, so it cannot go straight into a generated column or index
  expression; wrap it in an immutable function that names the dictionary explicitly.

**Typo tolerance and substring/prefix with `pg_trgm`**

```sql
CREATE EXTENSION IF NOT EXISTS pg_trgm;
CREATE INDEX CONCURRENTLY product_name_trgm ON product USING gin (name gin_trgm_ops);
-- multi-tenant: CREATE EXTENSION btree_gin; then gin ("tenantId", name gin_trgm_ops)
SELECT id, name, word_similarity($1, name) AS score
FROM product
WHERE "tenantId" = $2 AND $1 <% name          -- word_similarity: query inside a longer name
ORDER BY score DESC, id DESC
LIMIT 20;
```

- A trigram GIN index also accelerates `ILIKE '%q%'`, the usual quick fix for an ORM
  substring filter that does a full scan. The index must match the SQL the ORM actually emits.
  Prisma `mode: 'insensitive'` emits `ILIKE` on the bare column. Django `icontains` emits
  `UPPER(col) LIKE UPPER(...)`, so it needs the index on `UPPER(name) gin_trgm_ops`. Confirm
  with `EXPLAIN`.
- Escape `%`, `_` and `\` in user input before putting it inside a `LIKE`/`ILIKE` pattern.
- `similarity` (`%`) compares whole strings and penalises long names; `word_similarity` (`<%`)
  suits "query is part of the name". Thresholds: `pg_trgm.similarity_threshold` (default 0.3) and
  `pg_trgm.word_similarity_threshold`.
- Queries under 3 characters produce few trigrams and match poorly. Route them to a prefix
  query: `lower(name) LIKE lower($1) || '%'` (wildcards escaped) on a btree index over
  `lower(name) text_pattern_ops`, or a tsquery with `:*`. A `text_pattern_ops` index serves
  `LIKE`, not `ILIKE`.

**General**
- ORMs: use the parameterized raw-query form (Prisma `$queryRaw` tagged template, Django
  `params=`, SQLAlchemy `text()` with bind params). Never string-concatenate user input.
- Large `OFFSET` gets slower with depth, so cap page depth or use keyset pagination:
  `WHERE (score, id) < ($last_score, $last_id)`. This works only when every `ORDER BY` column
  sorts the same direction (`score DESC, id DESC`). With mixed directions, split it into
  `score < $s OR (score = $s AND id > $i)`. Equality on a computed float is fragile: select
  the score rounded (e.g. `round(score::numeric, 6)`) and paginate on that same rounded value.
- `count(*) OVER ()` for totals costs a full match; for big result sets cap it ("1,000+").
- Verify the plan: `EXPLAIN (ANALYZE, BUFFERS)` must show the GIN index, not a Seq Scan.

---

## SQLite FTS5 (rung 1)

```sql
CREATE VIRTUAL TABLE pages_fts USING fts5(
  title, body,
  content='pages', content_rowid='id',            -- external content: no duplicate text
  tokenize='unicode61 remove_diacritics 2',       -- tokenizer is per TABLE (see below)
  prefix='2 3'                                    -- fast 2- and 3-char prefix queries
);
-- keep it in sync (same transaction as the write):
CREATE TRIGGER pages_ai AFTER INSERT ON pages BEGIN
  INSERT INTO pages_fts(rowid, title, body) VALUES (new.id, new.title, new.body);
END;
CREATE TRIGGER pages_ad AFTER DELETE ON pages BEGIN
  INSERT INTO pages_fts(pages_fts, rowid, title, body) VALUES ('delete', old.id, old.title, old.body);
END;
CREATE TRIGGER pages_au AFTER UPDATE ON pages BEGIN
  INSERT INTO pages_fts(pages_fts, rowid, title, body) VALUES ('delete', old.id, old.title, old.body);
  INSERT INTO pages_fts(rowid, title, body) VALUES (new.id, new.title, new.body);
END;
-- backfill / repair: INSERT INTO pages_fts(pages_fts) VALUES ('rebuild');
```

- **The tokenizer applies to every column of the table.** `porter` stems titles, names and codes
  as well as prose. To stem only prose, use two FTS tables (stemmed body, unstemmed title) and
  combine their scores, or skip stemming and rely on prefix queries. Test both against the golden
  set.
- **External-content deletes must pass the OLD column values** via the special `'delete'`
  command. A plain `DELETE FROM pages_fts` or wrong old values silently corrupts the index.
- `bm25(pages_fts, 10.0, 1.0)` weights columns in declaration order; **lower (more negative) is
  better**, so `ORDER BY bm25(...)` ascending.
- **Escape `MATCH` input**: quote every token (`"tok"`, doubling any inner `"`), join with spaces
  (implicit AND), and add `*` after the closing quote for prefix: `"wire"*`. Raw `"`, `-`, `:`,
  `(`, `NEAR`, `AND`/`OR`/`NOT` otherwise raise errors or change the query's meaning.
- Filter visibility in the same query: `JOIN pages p ON p.id = pages_fts.rowid WHERE pages_fts
  MATCH :q AND (p.is_private = 0 OR :is_staff = 1)`, where `:is_staff` comes from the session,
  never from the request. `snippet()` / `highlight()` then only run on rows the
  viewer may see.
- `tokenize='trigram'` (SQLite 3.34+) gives substring matching, not edit-distance typo
  tolerance. True typo tolerance needs a spellfix-style candidate step or a higher rung.

---

## MySQL / MariaDB FULLTEXT (rung 1)

- InnoDB `FULLTEXT(name, description)`; `MATCH(...) AGAINST(? IN NATURAL LANGUAGE MODE)`.
- `IN BOOLEAN MODE` gives prefix (`wire*`) but treats `+ - < > ( ) ~ * " @` as operators —
  escape or strip them from user input.
- Minimum token length (`innodb_ft_min_token_size`, default 3) and the built-in stopword list
  silently drop short terms such as "tv" and "go". Check them before you blame relevance.
- Use the `ngram` parser for CJK text.

---

## MongoDB (rung 1)

- Classic `$text` index: one per collection, basic stemming, no typo tolerance, no prefix.
- Atlas Search (`$search`, Lucene-based) adds fuzzy, autocomplete and facets. Check whether it is
  available for the deployment you run (Atlas, or the self-managed search process) before
  designing around it.

---

## Meilisearch (rung 2)

- Typo tolerance on by default: 1 typo from 5 characters, 2 typos from 9 (`typoTolerance`). Turn
  it off per attribute for SKUs and codes (`disableOnAttributes`).
- `searchableAttributes` order **is** field weight. `filterableAttributes` / `sortableAttributes`
  must be declared before you can filter or sort, and changing them triggers a reindex.
- `pagination.maxTotalHits` defaults to **1000**; deeper results are not reachable.
- Document writes are **asynchronous tasks**: check the task status, and treat a failed task as a
  sync failure for the outbox to retry.
- `POST /documents` replaces whole documents; `PUT /documents` merges fields. Pick deliberately.
- Multi-tenant browser search: issue **tenant tokens** (signed from a search key, with the tenant
  filter embedded). The master key and admin keys stay on the server.
- Zero-downtime reindex: build `products_v2`, then `swap-indexes` with `products`.
- `processingTimeMs` = engine time (timing #1), not request time.

---

## Typesense (rung 2)

- **The index lives in RAM.** Size memory from the measured dataset before choosing it on a
  small VPS.
- `query_by` + `query_by_weights` set field weights. `num_typos` (default 2) and `prefix`
  (default true) apply per field. Disable typos on identifier fields.
- Browser search: **scoped search keys** generated server-side from a search-only key with a
  `filter_by` embedded. Never ship the admin key.
- Reindex: new collection, then repoint the **collection alias**.
- `search_time_ms` = engine time.

---

## Elasticsearch / OpenSearch (rung 3)

- On user input use `multi_match` or `simple_query_string`. **`query_string` on raw input**
  raises errors and allows leading wildcards and regex that can exhaust the cluster.
- `index.max_result_window` defaults to 10,000. Deep pagination uses `search_after` with a
  point-in-time. Both engines support this; OpenSearch added it in 2.4.
- Prefix and autocomplete: the `search_as_you_type` field type, edge n-grams, or the completion
  suggester. Typos: `fuzziness: "AUTO"`.
- Reindex through **aliases**: write `products_v2`, then swap the alias atomically.
- Near-real-time: writes become searchable after the refresh interval (default 1s). On a shard
  that has had no searches for a while, Elasticsearch skips scheduled refreshes unless
  `refresh_interval` is set explicitly. The next search then waits for a refresh, so the first
  query after an idle period is slower. Account for this in cold-run measurements.
- Document-level security may need a paid tier or plugin. Otherwise put the permission filter in
  every query's `bool.filter`.
- `took` = engine time in ms. JVM heap is usually ≤50% of RAM and under ~31 GB.

---

## Hosted (Algolia and similar)

- Billed per search request and per record. Search-as-you-type multiplies requests, so debounce
  and price it at the expected keystroke volume. Look up the current price; don't quote one
  from memory.
- Multi-tenant browser search uses **secured, filter-embedded search keys** generated server-side.

---

## Client-side (rung 0)

| Library | Shape | Note |
|---|---|---|
| Pagefind | Static index built after the site build, loaded in chunks | Best fit for static docs/blogs; scales to large sites cheaply |
| MiniSearch | In-memory inverted index, prefix + fuzzy, add/remove | Small dynamic corpora in the browser or a Node process |
| FlexSearch | In-memory, very fast, more configuration | Watch memory with large corpora |
| Orama | Full-text (+ vectors) in browser/edge | Check bundle size |
| Lunr | Index is immutable once built | Rebuild to change anything |
| Fuse.js | **Linear scan per query, not an index** | Command palettes of a few thousand items; not site search |

⛔ Never put a private or tenant-scoped document in an index that ships to the browser. Every
visitor can download it.

---

## Load tools (timing #4)

`npx autocannon -c 20 -d 30 "http://localhost:3000/api/search?q=wireless"` · `k6 run script.js`
· `oha` · `wrk`. Run the load tool on a different machine from the server, or at least note that
it shared one. Cycle the query mix rather than hammering a single cached query.
