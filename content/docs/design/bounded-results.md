---
title: "Bounded Results"
description: "Why no read returns too much data at any level — the two kinds of capping, the per-command result budget, and the pagination channel that recovers what a response withheld"
weight: 130
---

Every query the engine runs is bounded twice: once while *discovering*
which symbols answer the query, and once while *assembling* the response
that carries them. This page states the law that keeps the two apart,
because they fail differently: a discovery bound threatens correctness,
an assembly bound only threatens completeness of a listing. The engine
treats them accordingly — one warns and refuses to propagate, the other
marks and paginates.

The page assumes [Planning from Measured
Cardinality](/docs/design/planning): probes, anchors, and the refinement
waves. Here we care about what happens when a bound is *hit*.

## 1. The two kinds of capping {#two-kinds}

**Discovery capping is a correctness hazard.** A statement resolves by
reading rows — its own matches, or its neighbours' parents and children.
Every such read carries `LIMIT k+1` where \(k\) is the safety cap
(`--probe-cap`, shared with the probe so "resolved" and "not truncated"
name the same threshold). If the read returns \(k+1\) rows, the
statement's selection is **capped**: a truncated, order-arbitrary subset
of the true answer. Three rules follow:

1. A capped selection is *flagged* (`Selection.capped`) and the flag
   survives every path that copies or derives from it — including the
   derive path, where a receiver's own filter can shrink the re-read
   below the cap and hide the truncation.
2. A capped selection is **not a participant in composition**: it
   neither narrows its neighbours nor is narrowed by them. Constraining
   from an arbitrary subset would silently drop correct answers; the
   engine retries the read once a neighbour's constraint makes it more
   selective, and composes only then.
3. A capped statement is *said*: the result carries
   `too many results: this part of the graph is incomplete`, with advice
   and without totals — counting the true size would cost the very scan
   the cap exists to avoid.

**Assembly capping is a pagination invitation.** Once the symbol set is
settled, the response bounds what each node carries: its instance list and
its distinct out-edge listing. Hitting these caps sets *markers* —
`edges_truncated`, and in a `detail=full` response `instances_truncated`
with an exact `total_instances`; the skeleton carries only
`instance_count`, what the result holds — never the `capped` flag and
never a warning. The graph is right, a listing is merely paged, and the
[pagination endpoints](#pagination) serve the rest.

The boundary between the two is structural: assembly runs strictly after
the composition fixpoint drains, so nothing capped-for-pagination can
flow back into discovery.

## 2. The per-command result budget {#result-budget}

The response itself is bounded by the result budget
(`--max-result-symbols`, `?limit=`, or `limits(symbols=)` per command):
the maximum distinct symbols shipped **per command**. Per command, not
per response — every selector is its own request with its own budget, so
a dedicated anchor can never be evicted by a neighbouring command's
fan-out. (The global variant was tried and produced exactly that failure:
an anchored symbol whose file path sorted after a large module's derived
neighbours lost the cap lottery to them.)

Within a command, survivors are chosen deterministically by
`(path, offset, id)`; a symbol kept by *any* command survives;
`truncated` reports that some command overflowed. A \(k\)-command query
can therefore return up to \(k \cdot \text{limit}\) symbols, which
stays small because per-node payload is independently bounded (§3) and
\(k\) is the query's own size.

The budget also reaches into SQL: leaf reads of budget-bounded verbs are
bounded by a multiple of the budget (eight times it), so a bare
enumeration cannot materialise far more rows than the response could ever
keep. That bound holds only while no neighbour *consumes* the statement's
rows: a neighbour with a selection of its own clears it, because pruning
that neighbour against a truncated read would drop correct rows, while a
neighbour that merely echoes — a bare `{ }` — leaves it in place
([Evaluating the Fixpoint](/docs/design/evaluation)). A statement that
stopped at this bound without hitting the safety cap says so with its own
warning, naming the budget rather than the cap.

## 3. Render-side assembly cap {#render-caps}

One `listing` cap (`--listing-cap`, default 100; `0` disables) bounds the
rows in *every* per-node listing — the separate caps collapsed into one
number after measuring that a uniform cap is safe (a hub at listing=100 is
~380 KB). Two listings remain:

- **instances per node.** The engine's instances were already bounded by
  the safety cap; only serialization was not — a single kernel module's
  931 instances exceeded the 1 MB ceiling on their own. Real code
  instances sort before sentinel/containment ones, so a truncated list can
  never misclassify a code symbol as a directory node or drop the
  definition a renderer looks for. `total_instances` is exact for
  truncated nodes (an index count, bounded by the symbol's own size).
- **distinct out-edges per node.** Fetched discovery-side as a windowed
  SQL LIMIT (`row_number() OVER (PARTITION BY from_instance)` with a `+1`
  sentinel), sized to the largest `listing` any command requests; render
  trims each node to its own.
**Call sites per `(from, to)` pair are no longer a listing.** A client
draws one edge per pair, and a single hot pair was measured carrying 2,454
of a response's 2,898 edge entries — so `/query` now ships one edge per
pair with its `occurrence_count`, and the sites themselves are paged by
the edge endpoint. There is nothing left to cap on that path.

Objects follow the same discipline in `detail=full`: a file entry ships
only if a kept call site or a kept instance references it. The skeleton
has no objects map at all — every detail row carries its own path.

The `listing` cap (like the `symbols` budget) is set per command by the
[`limits()`](/docs/syntax#limits-verb) verb, which inherits into nested
scopes; the server flag is only the default.

## 4. The pagination channel {#pagination}

What assembly withheld, three endpoints recover per node and per edge
([HTTP API](/docs/api#references)):

- `GET /v1/symbol/{id}/references` pages the references out of a symbol
  in the engine's own `(from_instance, ref)` unit — the same containment
  rule and target choice edge discovery uses, so fetched rows are
  exactly the withheld edges. The page query is built to stream: source
  instances walked in id order, each one's references via a lateral in
  ref-id order, so the plan is an incremental sort that stops at
  `limit+1` rows — 3 ms per page on a symbol with 931 instances and a
  million references, where the naive `DISTINCT ON` form took seconds
  and a 65 MB disk sort.
- `GET /v1/symbol/{id}/instances` pages a symbol's instances in display
  order — a keyset over `(object, offset, id)`, so pages append where the
  UI shows them — with an exact total, optionally scoped by `?parent=` to
  one containment context: a symbol split across several containers is
  paged one context at a time, and the contexts partition its instances.
- `GET /v1/edge/{from}/{to}/occurrences` pages one edge's call sites in
  display order `(file, offset, ref id)`. An edge *is* a `(from, to)`
  pair, so this is its own listing rather than a filter over the whole
  references stream.

Cursors are keyset, opaque, and honest about scan position: a row the
response filters out (a self-reference has no target instance and the
engine draws no such edge) still advances the cursor, or a page boundary
landing on one would end pagination early with rows remaining.

Counts differ by kind, deliberately. An instance total is exact — one
index-driven count bounded by the symbol's own size. An edge count is
**capped** ("1000+"): exact out-edge totals for a hub node cost the
unbounded scan this whole design eliminates.

## 5. Truncation and the caches {#caches}

A capped discovery read that populates a layer persists its `truncated`
bit on the layer row, and the warning is *reconstructed* from that bit
on every cache hit — a cached truncated result warns exactly like a
fresh one (see [Search § Truncation
semantics](/docs/design/search#truncation-semantics)). The SQL result
cache is unaffected by assembly caps: pagination pages bypass it
entirely, since a page is poor cache currency.

## 6. Configuration {#configuration}

| Flag | Env | Default | Bounds |
|---|---|---|---|
| `--probe-cap` | `ASKL_PROBE_CAP` | 1000 | discovery reads and probes (§1); `0` disables resolution |
| `--max-result-symbols` | `ASKL_MAX_RESULT_SYMBOLS` | 100 | distinct symbols per command (§2) |
| `--listing-cap` | `ASKL_LISTING_CAP` | 100 | rows per per-node listing: instances, distinct out-edges (§3) |

`0` disables the corresponding cap. Per request, `?limit=` (symbols) and
`?listing=` override for one response; per command, the
[`limits(symbols=N, listing=M)`](/docs/syntax#limits-verb) verb overrides
and inherits into nested scopes.

## Where to read more {#where-to-read-more}

- [Planning from Measured Cardinality](/docs/design/planning) — the
  probes and refinement waves that decide *when* a read is attempted.
- [Evaluating the Fixpoint](/docs/design/evaluation) — the worklist a
  capped selection abstains from.
- [Layers and layer operations](/docs/design/layers) — where the
  `truncated` bit lives.
- [HTTP API](/docs/api) — the wire format of everything above.
