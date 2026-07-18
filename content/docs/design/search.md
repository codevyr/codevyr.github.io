---
title: "Design: search()"
description: "How the search() verb turns a literal-string query into byte-range matches over the whole indexed corpus, without regex, with byte-exact positions, and with a correctness invariant across filter compositions"
weight: 200
---

The `search()` verb is the closest askl gets to `grep` over the entire indexed corpus.
It takes a literal string and materialises, as a first-class askl selection, every
byte-range occurrence of that string in the source content of every indexed file —
subject to whatever filters (`project(...)`, etc.) are in scope. Downstream verbs
compose with it just like any other selector: `search("foo") { }` walks the children
of every match, `project("kubernetes") search("HandleFunc")` scopes to one project,
and so on.

This document explains how the verb is built. It covers the four SQL variants the
verb dispatches to, the storage layout in `content_store`, the pg_trgm / tsvector
pre-filter and the recheck pipeline, the PL/pgSQL helper that extracts byte offsets,
the ephemeral-layer cache key composition, how filter awareness threads through the
cache, and the correctness invariant for content that's deduplicated across
projects.

## Background: what the verb has to do

Given a query string `q` and a set of filters, the verb has to return every
byte-range occurrence of `q` in the source content of every indexed file, along
with the object and project each occurrence belongs to. The result flows into the
same graph model everything else uses: one ephemeral symbol per match, one
instance per byte range, an eph_layer capturing the whole batch, and standard
downstream propagation.

Three constraints shape the design:

- **No regex.** Users mostly want to search for identifiers or literal strings.
  Regex support would require an additional index-friendly matching layer and
  invites a class of user footguns (`foo.*bar` treated as regex). The verb is
  literal-only in every mode.
- **Byte-exact positions.** Downstream composition depends on byte-range instance
  offsets to attribute matches to enclosing symbols and to compare against
  indexer-produced ranges. Match positions have to be exact byte offsets in the
  original content, not character positions or approximations.
- **Cache-friendly.** The corpus is large (~1.9 GB in a representative deployment)
  and searches are interactive. Repeat calls with the same query and filter set
  must hit a cache.

## The four SQL variants

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

Every variant produces the same result shape and drops through the same
skeleton (`content_store ⋈ objects ⋈ LATERAL find_substr_byte_ranges(...)`), so
downstream code is variant-agnostic.

The split between two index types is deliberate: the fast path (variant 3) uses
`content_tsv` (a `tsvector` produced by `to_tsvector('simple', lower(...))`),
which natively understands word boundaries and gives sharp answers for the
common case of a whole-word case-insensitive search. Every other variant uses
`pg_trgm` on `content_text`, which handles arbitrary `LIKE` / `ILIKE` patterns
via trigram matching.

## Storage layout

`content_store` is deduplicated across projects: each unique file content is
stored once and referenced by its SHA-256 `content_hash`. Multiple projects can
point at the same content row (rebases, vendored copies, monorepo siblings).

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

Two generated columns keep the maintenance surface tiny. Both use
`IMMUTABLE` PL/pgSQL helpers that catch conversion errors:
`safe_convert_from(bytea)` returns NULL on invalid UTF-8, and
`safe_to_tsvector_simple(text)` returns NULL when the input exceeds PostgreSQL's
1 MiB tsvector limit. NULL values are silently dropped from the corresponding
GIN, so oversize or binary files fall out of the tsvector index (variant 3
misses them) but stay indexed by pg_trgm (variants 1, 2, 4 still find matches
inside them).

`objects` connects a project + filesystem_path to a `content_hash`:

```
objects {
    id             INT   PRIMARY KEY
    project_id     INT
    content_hash   TEXT             -- FK-like reference to content_store
    filesystem_path TEXT
    ...
}
```

The `objects` row is what carries the project identity. Content dedup means one
`content_store` row can be reachable from multiple `objects` rows — one per
project (and file path) referencing it.

## The pipeline

Every variant runs the same three-step pipeline:

```
   pg_trgm/tsvector GIN bitmap scan
              │
              ▼
     ~180 candidate ctids  (many false positives — GIN is lossy)
              │
              ▼
   Bitmap Heap Scan + Recheck
              │  fetches content_text (from TOAST if large),
              │  re-evaluates the ILIKE / tsquery predicate
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

The recheck is the load-bearing cost. GIN indexes are *lossy* — a trigram bitmap
covers "some rows match, plus false positives, indistinguishably" — so
PostgreSQL has to fetch each candidate row and re-evaluate the predicate against
its content. For a large corpus this dominates the query time: fetching ~180
TOASTed `content_text` values, decompressing them, and scanning them with
`ILIKE`. The LATERAL join to `find_substr_byte_ranges` runs only on the small
handful of rows that survive.

Two subtle correctness properties fall out of this shape:

1. **The user's query is bound as a pre-escaped LIKE pattern** (`$3 = '%…%'`
   with `\`, `%`, `_` escaped) rather than concatenated in SQL. Without escaping,
   a user query like `mana_ib_reg_user_` has PostgreSQL interpret the
   underscores as single-character wildcards, and the pg_trgm recheck candidate
   set balloons from ~180 rows to ~6000. Pre-escaping in Rust and passing as a
   bound parameter cuts the query time by ~3.5×.

2. **`find_substr_byte_ranges` uses literal `position()`**, not regex. The
   user's raw query is bound as `$1` and passed to the helper for exact
   substring matching. Even for whole-word searches, the boundary check happens
   after the fact in SQL — not inside a regex.

## The `find_substr_byte_ranges` PL/pgSQL helper

Extracting exact byte offsets requires walking the content once per candidate.
The helper does this in a single `IMMUTABLE PARALLEL SAFE` function that
maintains synchronised character and byte cursors:

```sql
CREATE FUNCTION index.find_substr_byte_ranges(
    haystack text, needle text, max_n int
) RETURNS TABLE(start_byte int, end_byte int, start_char int)
LANGUAGE plpgsql IMMUTABLE PARALLEL SAFE
AS $$
DECLARE
    cur_char int := 1;
    cur_byte int := 0;
    nlen int := length(needle);
    found int;
    gap_bytes int;
    emitted int := 0;
BEGIN
    IF nlen = 0 THEN RETURN; END IF;
    LOOP
        EXIT WHEN cur_char > length(haystack) OR emitted >= max_n;
        found := position(needle in substring(haystack from cur_char));
        EXIT WHEN found = 0;
        found := cur_char + found - 1;
        gap_bytes := octet_length(substring(haystack from cur_char for (found - cur_char)));
        cur_byte := cur_byte + gap_bytes;
        start_byte := cur_byte;
        cur_byte := cur_byte + octet_length(substring(haystack from found for nlen));
        end_byte := cur_byte;
        start_char := found;
        RETURN NEXT;
        cur_char := found + nlen;
        emitted := emitted + 1;
    END LOOP;
END;
$$;
```

Three things worth pointing out:

- The cursor advances **both** `cur_char` and `cur_byte` between matches, using
  `octet_length(substring(...))` to account for multi-byte UTF-8 characters
  between the previous match and the current one. This keeps byte positions
  exact even in mixed ASCII / multi-byte content, without needing a second
  pass.
- `max_n` caps the number of matches per call, giving the caller SQL-side
  truncation control. See the truncation section below.
- `IMMUTABLE PARALLEL SAFE` lets the planner inline calls into a `CROSS JOIN
  LATERAL` and parallelise across candidate rows.

The whole-word variants (3 and 4) don't use `\y` regex boundaries. Instead they
add an ASCII-only word-boundary check in SQL against `content_text`, using a
small helper `is_word_char(text) → boolean` that just tests the character's
codepoint against `[A-Za-z0-9_]`.

## Cache key composition

Every `search(...)` call materialises an ephemeral layer keyed by the inputs
that affect its output. The hash inputs are, in order:

- `parent_id` of the eph_layer (the layer above in the current query's chain)
- the literal bytes `"search"` — variant discriminator
- the query byte-length followed by the query bytes
- `case_sensitive` as one byte
- `whole_word` as one byte
- `limit` as a little-endian 64-bit int
- `CompositeFilter::hash_into(&mut Sha256)` — the full recursive hash of the
  surrounding command's filter set

The last input is what makes the cache filter-aware without special-casing
individual filter types. `CompositeFilter::hash_into` walks the And/Or/Not/Leaf
tree, mixing in a discriminator byte per variant and a canonical hash of each
leaf. When a leaf's semantics change (e.g., `ProjectFilterMixin`'s project
name), the whole eph_layer hash changes and the cache falls through to a fresh
population.

```
                     Hash inputs (SHA-256)
                            │
      ┌─────────────────────┴───────────────────────┐
      │                                             │
   parent_id      "search"                query   flags   limit    CompositeFilter
      │              │                      │       │       │             │
      └──────────────┴──────┬───────────────┴───────┴───────┴─────────────┘
                            ▼
                    with_eph_layer(hash, EphLayerKind::Search)
                            │
      ┌─────────────────────┴──────────────────────┐
      │                                             │
   cache HIT                                    cache MISS
      │                                             │
      │                                    populate closure runs:
      │                                    ├─ Index::search_content_matches
      │                                    ├─ group matches by project_id
      │                                    ├─ insert EphSymbolRow per project
      │                                    ├─ insert EphInstanceRow per byte range
      │                                    └─ persist eph_layers.truncated
      │                                             │
      └──────────────────┬──────────────────────────┘
                         │
                    (layer_id, created, truncated)
```

On a cache hit, the populate closure never runs; the caller reads
`eph_layers.truncated` to reconstruct any warning that was persisted with the
layer. This is what keeps `truncated` visible on repeat calls even though the
warning message itself lives in Rust.

## Ephemeral layer emission

The populate closure groups matches by `project_id` and emits:

- one `EphSymbolRow` per matching project, named `search:<query>`, with
  `symbol_type = SYMBOL_TYPE_CONTENT` (a first-class type shared with `loc()`);
- one `EphInstanceRow` per byte-range match, pointing at the corresponding
  object and covering `(start_byte, end_byte)`.

Shared content across projects surfaces cleanly. If `content_hash = "cs_shared"`
is referenced by object 100 in project X and object 200 in project Y, an
unscoped `search("token")` that matches inside `cs_shared` emits:

| Ephemeral symbol | Object | Instance range |
|---|---|---|
| `search:token` (project X) | 100 | `(start_X, end_X)` |
| `search:token` (project Y) | 200 | `(start_Y, end_Y)` |

Two projects, two symbols, two instances — even though the content itself
lives in a single `content_store` row.

## Filter composition and the correctness invariant

When a filter like `project("X")` is upstream of `search(...)`, the composite
filter's `objects_expr()` (only `ProjectFilterMixin` implements this today)
gets resolved into a small `Vec<i32>` of `visible_project_ids` and bound as
`$4`. The SQL adds one clause:

```sql
WHERE ...
  AND o.project_id = ANY($4)
```

This is the correctness invariant for cross-project shared content. Without
it, a scoped search of `cs_shared` in project X would emit *both* project X's
and project Y's objects — because both objects share the content_hash and the
JOIN would return both. With the filter, only the caller's project's objects
survive.

The test `search_cross_project_shared_content_scopes_to_filter` in
`askld/src/all_tests.rs` guards this: fixture rows put the same
`cs_shared` content_hash under two projects, then verify that
`project("search_proj_1") search("shared_token")` and
`project("search_proj_2") search("shared_token")` produce **disjoint** output
node sets. If the filter ever regresses, the test fails.

Nothing about this is `project`-specific in the code path. Any future
`FilterLeaf` that implements `objects_expr()` (e.g., a hypothetical
`filetype("...")` or `path("...")` filter) gets the same treatment for free —
its constraints flow through `CompositeFilter::compose_objects()` and become
part of the visible-project-ids resolution.

## Truncation semantics

The verb accepts a `limit=N` parameter (default `500`). Enforcement is a SQL
`LIMIT N+1` fence: if the query returns `N+1` rows, we drop the last one and
set `truncated = true`. The extra row is what distinguishes "hit the limit
exactly" from "we truncated".

`truncated` is persisted onto the `eph_layers` row (a small boolean column
added alongside `populated`), so cache hits carry the same information as
cache misses. When the caller receives a `truncated: true`, it asks the
originating selector to reconstruct the warning:

```
SearchSelector::make_truncation_warning(&self) -> Option<pest::error::Error<Rule>>
```

This is a per-selector default method — every `Selector` returns `None` by
default, and `SearchSelector` overrides it to produce a `CustomError` with the
verb's own span. Cache hits therefore surface the warning with the exact
source location of the offending `search(...)` call, even though the warning
message wasn't stored in the cache. Only the boolean fact of truncation lives
alongside the cached rows.

## Cache invalidation

The eph_layer cache is scoped to a single logical corpus state. When a project
is re-uploaded, `IndexStore::finalize_project` — the transaction that flips
`upload_status` to `Complete` — also calls `purge_eph_cache`, which drops every
non-canary `eph_layers` row atomically with the upload commit. This
guarantees that a subsequent `search(...)` sees the new state, not stale
matches from before the re-upload.

There's no per-project TTL and no versioned cache key. The finalize step is
the sole invalidation trigger. This keeps the cache correctness argument
short: the eph_layer only ever reflects a corpus state that was current when
it was populated, and it's discarded the moment that state changes.

## Performance characteristics

On the representative deployment (94k content_store rows, 1.9 GB compressed),
the query `project("linux") search("mana_ib_reg_user_")` measures at about
**800 ms hot-cache**, with a cold-cache time of about 1120 ms.

Where the time goes on the hot path:

- pg_trgm GIN bitmap scan: ~5 ms
- Bitmap Heap Scan + Recheck on ~180 candidate rows: **~700 ms** — the
  dominant cost. Each recheck fetches the TOASTed `content_text`, decompresses
  it, and re-runs the `ILIKE` predicate to filter false positives out of the
  GIN candidate set.
- Function Scan on `find_substr_byte_ranges` for the ~3 rows that survive:
  ~1.5 ms

The recheck is expensive because pg_trgm is *lossy*: even after escaping the
underscore wildcards in the LIKE pattern (which by itself dropped the
candidate count from ~6000 to ~180 rows and the query time from ~3140 ms to
~880 ms), the remaining ~180 candidates still all need to be TOAST-fetched
and re-verified.

Two attempted optimisations that measurably don't help:

- **Switching TOAST compression from pglz to lz4** — was expected to save
  ~200 ms of decompression on the hot path. Measured impact: **~0 ms** on
  hot-run time. Decompression turned out to be a small fraction of the CPU
  cost; the recheck cost is dominated by the ILIKE scan through ~3 MB of
  decompressed text, not by decompression itself.
- **A denormalised `content_store.project_ids int[]` GIN with `BitmapAnd`
  against pg_trgm** — expected to give a ~15-25× speedup for small-project
  searches by narrowing the pre-filter to the project's ~1748 content rows.
  Deferred: the workload is `linux`-dominant, and for `linux`-scoped searches
  the intersection between the trgm bitmap and the project bitmap is
  essentially the trgm bitmap (linux owns 92k of the 94k content rows), so
  BitmapAnd wouldn't help.

Recommended query hygiene:

- Longer, more distinctive queries (5+ characters) narrow the pg_trgm
  candidate set exponentially — fewer recheck rows means faster searches.
- Scoping to a small project via `project("...")` is only a big win when the
  project owns a small fraction of `content_store`. For the largest project
  it's a wash.
- Whole-word case-insensitive (variant 3) hits the tsvector GIN, which is
  more selective than pg_trgm for identifier-shaped queries. Prefer it when
  the semantics fit.

## Design decisions and known limitations

- **No regex.** Users mostly want to search for identifiers or literal strings.
  Regex support would need either a `~*` operator over `content_text` (which
  the trigram index handles poorly for non-anchored patterns) or an
  additional index. It's also the most common source of user surprise
  ("`foo.*bar` didn't match `fooXbar`") when it's supported. Literal-only is
  a smaller correctness surface and matches the modal use case.

- **Byte offsets in case-insensitive variants use lowercased `content_text`
  positions.** For ASCII content the byte positions are identical to the
  original. For Unicode where `lower()` isn't byte-length-preserving (e.g.
  `ß` → `ss`), the reported offsets are shifted relative to the original
  content by the difference in byte count of the lowered prefix. Documented
  limitation; the fix is a per-content offset map, deferred as a follow-up.

- **Oversize content silently drops from the tsvector index.**
  PostgreSQL caps tsvector documents at ~1 MiB. Files that exceed the cap
  (generated code, lockfiles, minified bundles) have `content_tsv = NULL`
  and are invisible to variant 3 — but they still appear in `content_text`
  and pg_trgm, so variants 1, 2, 4 still match inside them. Documented
  asymmetry.

- **Non-UTF-8 content is skipped from both indexes.**
  `safe_convert_from(bytea)` returns NULL when the content isn't valid UTF-8;
  the NULL propagates through both generated columns and both GINs drop
  the row. Binary blobs never surface in search results.

- **ASCII-only word boundaries.** The `is_word_char(text)` helper only
  understands `[A-Za-z0-9_]`. Non-ASCII identifiers (e.g. Cyrillic, CJK,
  Greek letters used as variable names) don't count as word characters, so
  whole-word matching around them behaves as though every non-ASCII byte is
  a boundary. Acceptable for the dominant use case; documented.

- **Cache warnings are boolean, not descriptive.** The eph_layer stores
  `truncated: bool`, not a warning message. The verb reconstructs the
  message on demand from its own span. This means (a) warnings survive
  cache hits, (b) the message wording can evolve without invalidating the
  cache, and (c) the cache doesn't need to know about `pest::error::Error`.
