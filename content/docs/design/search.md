---
title: "search()"
description: "How the search() verb turns a literal-string query into byte-range matches over the whole indexed corpus, without regex, with byte-exact positions, and with a correctness invariant across filter compositions"
weight: 160
---

The `search()` verb is the closest askl gets to `grep` over the entire indexed
corpus. It takes a literal string and materialises, as a first-class askl
selection, every byte-range occurrence of that string in the source content of
every indexed file — subject to whatever filters (`project(...)`, etc.) are in
scope. Downstream verbs compose with it like any other selector:
`search("foo") { }` walks the children of every match, and
`project("kubernetes") search("HandleFunc")` scopes to one project.

It is also where the rest of the chapter has to pay. What the earlier pages call
a populate, a shard, or a key is here a concrete SQL pipeline over 1.9 GB of
source text with a measurable cost — so this page is a case study: the contract
the verb owes its callers, the SQL that meets it, the key that makes a repeat
call free, and the invariant that survives content shared between projects.

## What the verb has to do {#the-contract}

Given a query string `q` and the filters in scope, the verb owes its caller
every byte-range occurrence of `q` in every indexed file, attributed to the
object and project it belongs to and delivered in the graph model everything
else uses: ephemeral symbols, one instance per byte range, one ephemeral layer
for the batch. Three constraints shape how it gets there:

- **No regex.** Users mostly want identifiers or literal strings. Regex support
  would need an additional index-friendly matching layer and invites a class of
  user footguns (`foo.*bar` treated as regex). The verb is literal-only in every
  mode.
- **Byte-exact positions.** Downstream composition uses byte-range offsets to
  attribute matches to enclosing symbols and to compare against
  indexer-produced ranges, so positions must be exact byte offsets in the
  original content — not character positions, not approximations.
- **Cache-friendly.** The corpus is large (~1.9 GB in a representative
  deployment) and searches are interactive. Repeat calls with the same query and
  filter set must hit a cache.

## The four SQL variants {#the-four-sql-variants}

Search modes are the product of two flags: `whole_word` (`false` = substring
match, `true` = word-boundary match) and `case` (`smart`/`sensitive`/
`insensitive`; smart-case is resolved to a concrete bool at parse time). That's
four SQL variants, each hand-built and dispatched at query-build time in Rust:

| Variant | `whole_word` | case-sensitive | Pre-filter | Notes |
|---|---|---|---|---|
| 1 | `false` | `false` | `content_text ILIKE $3` | Substring, case-insensitive |
| 2 | `false` | `true`  | `content_text LIKE $3`  | Substring, case-sensitive   |
| 3 | `true`  | `false` | `content_tsv @@ phraseto_tsquery('simple', lower($1))` | Whole-word, case-insensitive — the only tsvector path |
| 4 | `true`  | `true`  | `content_text LIKE $3` + word-boundary check | Whole-word, case-sensitive  |

All four produce the same result shape and drop through the same skeleton
(`content_store ⋈ objects ⋈ LATERAL find_substr_byte_ranges(...)`), so
downstream code is variant-agnostic. Two index types rather than one, because
the common case deserves a sharp answer: variant 3 goes to `content_tsv` (a
`tsvector` from `to_tsvector('simple', lower(...))`), which understands word
boundaries natively; the rest go to `pg_trgm` on `content_text`, which handles
arbitrary `LIKE` / `ILIKE` patterns via trigram matching — more general, and
correspondingly blunter.

## Storage layout {#storage-layout}

Both indexes sit on one table, and that table is the one place in the schema
where rows are shared rather than layered. `content_store` is deduplicated
across projects: each unique file content is stored once under the SHA-256 of
its bytes, so rebases, vendored copies, and monorepo siblings cost one row — and
one tokenisation — between them.

```
content_store {
    content_hash    TEXT         PRIMARY KEY
    content         BYTEA                       -- raw file bytes, LZ4 compressed
    content_text    TEXT         GENERATED      -- safe UTF-8 view of `content`
    content_tsv     TSVECTOR     GENERATED      -- lowercased simple-tokenised
}

INDEX content_store_text_trgm  ON content_store USING GIN (content_text gin_trgm_ops)
INDEX content_store_tsv_gin    ON content_store USING GIN (content_tsv)
```

Both generated columns are computed by `IMMUTABLE` PL/pgSQL helpers that catch
conversion errors instead of failing an upload — `safe_convert_from(bytea)`
returns NULL on invalid UTF-8, `safe_to_tsvector_simple(text)` returns NULL past
PostgreSQL's 1 MiB tsvector limit — and a NULL row is silently dropped from the
corresponding GIN, which is where two of the limitations at the end of this page
come from.

Project identity lives one table over: an `objects` row ties a `project_id` and
a `filesystem_path` to a `content_hash`, and is the join partner of every
variant. Dedup means one `content_store` row is reachable from several `objects`
rows, which is what the correctness invariant below is about.

## The pipeline {#the-pipeline}

Every variant runs the same three steps: narrow with an index, verify what the
index returned, extract offsets from the few rows that survive.

```
   pg_trgm/tsvector GIN bitmap scan
              │
              ▼
     ~180 candidate ctids  (many false positives — GIN is lossy)
              │
              ▼
   Bitmap Heap Scan + Recheck
              │  fetches content_text, re-evaluates the
              │  ILIKE / tsquery predicate
              ▼
     ~3-5 rows that actually match
              │
              ▼
   CROSS JOIN LATERAL find_substr_byte_ranges(content_text, needle, limit)
              │  walks each surviving content_text once,
              │  emits one row per byte-range match
              ▼
     (object_id, project_id, start_byte, end_byte) records
```

The middle step dominates, because GIN indexes are *lossy* — a trigram bitmap
says "some of these rows match, plus false positives, indistinguishably" — so
PostgreSQL must fetch every candidate and re-evaluate the predicate against its
content: ~700 ms of an ~800 ms hot-cache
`project("linux") search("mana_ib_reg_user_")`, against ~5 ms for the index scan
and ~1.5 ms for offset extraction
([full breakdown](/docs/design/search-appendix/#performance)).

Two properties of that shape follow, one of them worth real money.

**Bind the pattern pre-escaped.** The user's query is bound as a LIKE pattern
(`$3 = '%…%'`, with `\`, `%`, `_` escaped) rather than concatenated into the SQL.
Without escaping, a query like `mana_ib_reg_user_` has PostgreSQL read the
underscores as single-character wildcards and the candidate set balloons from
~180 rows to ~6000 — each of them then fetched and rechecked, a ~3.5× cost paid
entirely on rows that could never have matched.

**Matching is literal all the way down.** `find_substr_byte_ranges` locates
occurrences with `position()`, and it is the raw query (`$1`) that is bound to
it; the whole-word variants add an ASCII-only boundary check in SQL afterwards,
via a helper `is_word_char(text) → boolean` testing the codepoint against
`[A-Za-z0-9_]`. No `\y` boundaries, no regex anywhere. The helper walks each
surviving content once with a character cursor and a byte cursor advancing
together, which is what keeps offsets exact in multi-byte content
([body](/docs/design/search-appendix/#helper)).

## Cache key composition {#cache-key-composition}

A search is expensive enough that a second call must not repeat it — including
when it runs under a different query, with different ephemeral layers already
materialised. That much is the general machinery's doing rather than this
verb's: the expensive part of the materialisation is the **root shard**, whose
parent is the root layer, so no upstream ephemeral layer can enter its key and
one shard serves every context
([Partitioning a Materialisation](/docs/design/shards/#node-kinds)).

What is search-specific is which inputs that shard's hash folds, in order:

- the literal bytes `"search"` — the verb discriminator
- the query byte-length followed by the query bytes
- `case_sensitive` as one byte
- `whole_word` as one byte
- `limit` as a little-endian 64-bit int
- the **fused container scope**: `scope:none` when unscoped, else the scope's
  condition hash and/or its sorted resolved instance ids — a scope-fused scan
  reads the container's ranges, so the key must name them
  ([Layer Keys §3](/docs/design/layer-keys/#verb-input-hashes))
- `CompositeFilter::hash_into(&mut Sha256)` — the surrounding command's whole
  filter tree, hashed recursively
  ([Layer Keys §2](/docs/design/layer-keys/#filter-hash))

The last input is what makes the cache filter-aware with no per-filter special
case; the verb's only obligation is to fold the *whole* tree rather than the part
of it the scan believes it reads. The executor then materialises the layer per
visible root, salting the hash with the root's identity, so each project's root
shard is cached independently of which other projects are co-visible.

Bounds from outside the key belong to
[Caching](/docs/design/caching/#lifetime-and-invalidation): a cached search layer
ages out by TTL and LRU, and re-uploading a project purges the ephemeral cache in
the same transaction that commits the upload — so a search never returns matches
from a corpus state that no longer exists.

## Ephemeral layer emission {#ephemeral-layer-emission}

Matches have to become graph rows before anything downstream can compose with
them. The populate closure groups them by `project_id` and emits one
`EphSymbolRow` per matching project — named `search:<query>`, with
`symbol_type = SYMBOL_TYPE_CONTENT`, a first-class type shared with `loc()` —
and one `EphInstanceRow` per byte-range match, pointing at the corresponding
object and covering `(start_byte, end_byte)`.

Grouping by project is what makes shared content surface cleanly. If
`content_hash = "cs_shared"` is referenced by object 100 in project X and object
200 in project Y, an unscoped `search("token")` matching inside `cs_shared`
emits:

| Ephemeral symbol | Object | Instance range |
|---|---|---|
| `search:token` (project X) | 100 | `(start_X, end_X)` |
| `search:token` (project Y) | 200 | `(start_Y, end_Y)` |

Two projects, two symbols, two instances — even though the content itself lives
in a single `content_store` row.

## Filter composition and the correctness invariant {#the-correctness-invariant}

That same sharing is a hazard the moment a filter is in scope. When
`project("X")` is upstream of `search(...)`, the composite filter's
`objects_expr()` (only `ProjectFilterMixin` implements this today) resolves into
a small `Vec<i32>` of `visible_project_ids`, bound as `$4`, and the SQL adds one
clause:

```sql
WHERE ...
  AND o.project_id = ANY($4)
```

Without it a scoped search of `cs_shared` in project X would emit *both*
projects' objects, since both reach the same content row through the join. With
it, only the caller's project's objects survive — the correctness invariant for
cross-project shared content.

The test `search_cross_project_shared_content_scopes_to_filter` in
`askld/src/all_tests.rs` guards it: fixture rows put one `cs_shared`
content_hash under two projects, then verify that
`project("search_proj_1") search("shared_token")` and
`project("search_proj_2") search("shared_token")` produce **disjoint** output
node sets.

Nothing in the code path is `project`-specific. Any future `FilterLeaf`
implementing `objects_expr()` — a hypothetical `filetype("...")` or `path("...")`
— gets the same treatment for free, its constraints flowing through
`CompositeFilter::compose_objects()` into the visible-project-ids resolution.

## Truncation semantics {#truncation-semantics}

A search over a common term can match more rows than anyone wants back, so the
verb takes a `limit=N` parameter (default `500`). Enforcement is a SQL
`LIMIT N+1` fence: if the query returns `N+1` rows, we drop the last and set
`truncated = true`. The extra row is what distinguishes "hit the limit exactly"
from "we truncated".

Truncation has to survive caching, and a warning message cannot: on a cache hit
the populate closure never runs. So `truncated` is persisted onto the `layers`
row (a small boolean alongside `populated`), and the caller that reads it back
asks the originating selector to reconstruct the warning:

```
SearchSelector::make_truncation_warning(&self) -> Option<pest::error::Error<Rule>>
```

Every `Selector` returns `None` from that default method; `SearchSelector`
overrides it to produce a `CustomError` carrying the verb's own span. So a cache
hit surfaces the warning at the exact source location of the offending
`search(...)` call, though only the boolean fact was ever stored with the rows.

## Design decisions and known limitations {#design-decisions-and-known-limitations}

Several of the choices above buy a smaller implementation at a price worth
stating plainly:

- **No regex.** Supporting it would need either a `~*` operator over
  `content_text` (which the trigram index handles poorly for non-anchored
  patterns) or a second index — and regex is the most common source of user
  surprise ("`foo.*bar` didn't match `fooXbar`") where it is offered.
  Literal-only is a smaller correctness surface and matches the modal use case.

- **Byte offsets in case-insensitive variants use lowercased `content_text`
  positions.** For ASCII content these are identical to the original. Where
  `lower()` is not byte-length-preserving (`ß` → `ss`), reported offsets are
  shifted by the byte-count difference of the lowered prefix. The fix is a
  per-content offset map, deferred.

- **Oversize content silently drops from the tsvector index.** Files past
  PostgreSQL's ~1 MiB tsvector cap (generated code, lockfiles, minified bundles)
  have `content_tsv = NULL` and are invisible to variant 3 — but remain in
  `content_text` and pg_trgm, so variants 1, 2, and 4 still match inside them. A
  documented asymmetry.

- **Non-UTF-8 content is skipped from both indexes.** `safe_convert_from(bytea)`
  returns NULL on invalid UTF-8, the NULL propagates through both generated
  columns, and both GINs drop the row. Binary blobs never surface in results.

- **ASCII-only word boundaries.** `is_word_char(text)` understands
  `[A-Za-z0-9_]` and nothing else, so around non-ASCII identifiers (Cyrillic,
  CJK, Greek letters used as names) whole-word matching behaves as though every
  non-ASCII byte were a boundary. Acceptable for the dominant use case.

- **Cache warnings are boolean, not descriptive.** The ephemeral layer stores
  `truncated: bool`, never a message; the verb reconstructs the message from its
  own span. So warnings survive cache hits, the wording can evolve without
  invalidating a single entry, and the cache needs to know nothing about
  `pest::error::Error`.
