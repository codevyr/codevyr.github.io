---
title: "search(): Implementation Notes"
description: "The full body of the byte-offset helper, and the measurements behind the search() case study — including two optimisations that did not pay"
weight: 165
---

The [search() case study](/docs/design/search) makes two claims a reader
may reasonably want to check rather than take: that the reported byte
offsets are exact, and that the recheck — not the index scan, not the
offset extraction — is where the time goes. The evidence for both is
here, kept out of the case study because neither changes its argument:
one is a page of PL/pgSQL, the other a benchmark log, including two
optimisations that measurably did not pay.

## The `find_substr_byte_ranges` helper {#helper}

Reporting a match as a byte range means knowing, for every match, how
many *bytes* precede it — which a character-indexed `position()` does not
tell you in content that is not pure ASCII. Converting after the fact
would mean a second pass per match. Instead the helper walks each
candidate's content once, advancing a character cursor and a byte cursor
together, in a single `IMMUTABLE PARALLEL SAFE` function:

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

Three properties are worth pointing out:

- The cursor advances **both** `cur_char` and `cur_byte` between matches,
  using `octet_length(substring(...))` to account for multi-byte UTF-8
  characters between the previous match and the current one. This keeps
  byte positions exact in mixed ASCII / multi-byte content without a
  second pass.
- `max_n` caps the number of matches per call, which is what gives the
  caller SQL-side truncation control
  ([truncation semantics](/docs/design/search/#truncation-semantics)).
- `IMMUTABLE PARALLEL SAFE` lets the planner inline calls into a
  `CROSS JOIN LATERAL` and parallelise across candidate rows.

## Where the time goes {#performance}

On the representative deployment (94k `content_store` rows, 1.9 GB
compressed), the query `project("linux") search("mana_ib_reg_user_")`
measures at about **800 ms hot-cache**, against about 1120 ms cold. The
hot path breaks down as:

- pg_trgm GIN bitmap scan: ~5 ms
- Bitmap Heap Scan + Recheck on ~180 candidate rows: **~700 ms** — the
  dominant cost. Each recheck fetches the TOASTed `content_text`,
  decompresses it, and re-runs the `ILIKE` predicate to filter false
  positives out of the GIN candidate set.
- Function Scan on `find_substr_byte_ranges` for the ~3 rows that
  survive: ~1.5 ms

The recheck is expensive because pg_trgm is *lossy*: even after escaping
the underscore wildcards in the LIKE pattern — which by itself dropped
the candidate count from ~6000 to ~180 rows and the query time from
~3140 ms to ~880 ms — the remaining ~180 candidates all still need to be
TOAST-fetched and re-verified.

### Two optimisations that measurably don't help {#non-wins}

- **Switching TOAST compression from pglz to lz4** — expected to save
  ~200 ms of decompression on the hot path. Measured impact: **~0 ms** on
  hot-run time. Decompression turned out to be a small fraction of the
  CPU cost; the recheck is dominated by the `ILIKE` scan through ~3 MB of
  decompressed text, not by getting that text decompressed.
- **A denormalised `content_store.project_ids int[]` GIN with `BitmapAnd`
  against pg_trgm** — expected to give a ~15-25× speedup for
  small-project searches by narrowing the pre-filter to the project's
  ~1748 content rows. Deferred: the workload is `linux`-dominant, and for
  `linux`-scoped searches the intersection between the trgm bitmap and
  the project bitmap is essentially the trgm bitmap (linux owns 92k of
  the 94k content rows), so the `BitmapAnd` would buy nothing.

Both are worth recording precisely because the reasoning behind them was
plausible. The lesson they share is that the cost is in scanning the
surviving text, so the only optimisations that pay are the ones that
leave less text to scan.

### Query hygiene {#query-hygiene}

Which is also the advice to give a user:

- Longer, more distinctive queries (5+ characters) narrow the pg_trgm
  candidate set sharply — fewer recheck rows means faster searches.
- Scoping to a project via `project("...")` is a big win only when that
  project owns a small fraction of `content_store`. For the largest
  project it is a wash.
- Whole-word case-insensitive (variant 3) hits the tsvector GIN, which is
  more selective than pg_trgm for identifier-shaped queries. Prefer it
  where the semantics fit.
