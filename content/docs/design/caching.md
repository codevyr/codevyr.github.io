---
title: "Design: Caching"
description: "How askl caches expensive work so repeat and composed queries stay cheap — the two cache tiers, the content-addressed ephemeral-layer cache, and how the executor shards the content map by layer so the costly scan is cached once and reused under any context while staying correct across filter compositions and re-uploads"
weight: 175
---

Askl queries are interactive and the corpus is large — a representative
deployment holds ~1.9 GB of source content. A `search()` over a common term, or
a deep call-graph walk, is expensive; running it again a second later, or as one
branch of a bigger composed query, should be nearly free. Caching is what makes
that true, and it has to do so **without ever returning a stale or wrong
answer**.

This document explains askl's caching from the ground up: the content-addressed
store that keeps each unique file content stored and indexed exactly once
(`content_store`), and the two request-time caches layered on top of it. It then
covers, in depth, the design of the ephemeral-layer cache — how a layer becomes a
content-addressed cache entry, how the executor shards the content map by layer so the
expensive part is cached **once and reused under any context**, and the
invariants that keep the cache correct across filter compositions and re-uploads.
It assumes the [layer data model](/docs/design/layers).

## Content-addressed source content: `content_store`

The most foundational cache is the source content itself. Every file's bytes live
in `content_store`, keyed by the SHA-256 `content_hash` of those bytes — so each
unique content is stored **once**, and every `objects` row with those exact bytes
(the same file in another project, a vendored copy, an unchanged file across a
re-upload) points at the one row instead of storing its own copy.

```
content_store {
    content_hash  TEXT      PRIMARY KEY   -- SHA-256 of the raw bytes
    content       BYTEA                   -- the file bytes, stored once
    content_text  TEXT      GENERATED     -- safe-UTF-8 view, GIN-indexed (pg_trgm)
    content_tsv   TSVECTOR  GENERATED     -- tokenised view, GIN-indexed
}
```

It earns the name *cache* in two ways. First, **storage dedup**: identical
content across projects and across re-uploads costs one row, not one per object —
monorepo siblings, rebases, and re-indexing a project whose files mostly didn't
change all collapse onto existing hashes. Second, and easier to miss,
**derived-index reuse**: the two generated columns and their GIN indexes — the
tsvector tokenisation and the trigram index that `search()` scans — are computed
once per unique content. A re-upload therefore re-tokenises and re-indexes only
the files whose bytes actually changed; every unchanged file hits its existing
row with its indexes already built.

`content_store` is the content-addressed sibling of the ephemeral-layer cache:
both key on a hash and reuse an existing entry rather than redo work. The
difference is *what* they key on — `content_store` hashes source **content** to
reuse its stored bytes and derived indexes; the ephemeral-layer cache hashes a
verb's **inputs** to reuse its computed output. The
[search design](/docs/design/search) covers the storage layout, the generated
columns, and the cross-project correctness invariant in full.

## Two request-time tiers

On the query hot path — layered on top of `content_store` — askl caches at two
more levels, for two different costs:

| | In-RAM SQL result cache | Ephemeral-layer cache |
|---|---|---|
| Lives in | process memory | the database (`index.layers` + rows) |
| Keys on | the exact SQL + binds | a content hash of the command's inputs |
| Caches | the rows a read query returned | a materialised layer of graph rows |
| Bounded by | bytes (`ASKL_SQL_CACHE_BYTES`) | TTL + LRU, wholesale purge on re-upload |
| Survives | one server process | across processes (it's in the DB) |

The two are complementary. The ephemeral-layer cache avoids **re-materialising**
a command's output (re-running a `search` scan, re-inserting its instances). The
in-RAM cache avoids **re-reading** already-materialised rows for identical
read queries within a process.

## Tier 1: the in-RAM SQL result cache

The read path (`cached_load`) is a thin proxy in front of the database. For a
read query it builds the SQL and its bind parameters, hashes them into a key, and
serves the cached row set on a hit. It is a byte-bounded LRU: `SqlResultCache`
evicts least-recently-used entries once the cache exceeds `ASKL_SQL_CACHE_BYTES`,
and it is disabled entirely when that budget is `0` (`is_enabled() = max_bytes >
0`), in which case `cached_load` degrades to a plain database load.

Correctness has two parts, and — importantly — **creating or garbage-collecting
ephemeral layers does not clear this cache.**

First, the key is the hash of the *rendered SQL and its binds*. Because every
visibility predicate is a `layer = ANY($visible)` clause, the visible layer ids
are part of those binds — so a read keys on the exact set of layers it could see.
Materialising a new ephemeral layer produces a query with different binds and
therefore a different key: a miss, never a stale hit against an entry built under
a different layer set. Ephemeral layer ids are never reused, so a dead layer's
entries can never be requested again and simply age out via LRU. That is why
ephemeral-layer churn — the common case, many per query — needs no invalidation
at all.

Second, mutations to the *persistent corpus* — finalising an upload, deleting a
project — **do** clear the whole cache, on commit, via a `ClearOnDrop` guard.
(An `epoch` counter closes the race with an in-flight load: a load snapshots the
epoch before its DB read, `clear()` bumps it, and a put with a stale snapshot is
rejected — so a pre-mutation read can never land in the cache after the clear.)
This is the *only* thing that clears the cache, and it is what guarantees a
cached row set never outlives the corpus state it was read from.

## Tier 2: the ephemeral-layer cache

Every ephemeral layer is a **content-addressed cache entry**. Its `hash` column
is a function of the inputs that determine its contents, and it carries a `UNIQUE`
index. Materialising a layer is therefore a single upsert:

```
create_eph_layer(parent_id, root_shard_id, hash, kind):
    INSERT INTO layers (hash, ...) VALUES (...)
    ON CONFLICT (hash) DO UPDATE SET last_used = now()
    RETURNING id, (xmax = 0) AS created, populated
```

The upsert *is* the cache lookup. `xmax = 0` distinguishes the row we just
inserted (`created = true`, a **miss**) from one that already existed
(`created = false`, a **hit**). On a hit the expensive populate step is skipped
entirely and `last_used` is bumped — the LRU touch is free, folded into the same
statement. On a miss the caller runs the populate closure to fill the layer's
rows.

### The two-phase `populated` guard

A cache key is a promise: "a row with this hash holds exactly this content." A
half-built layer would break that promise — a concurrent reader could hit the
hash, skip the populate, and read an empty or partial layer. The `populated`
boolean closes the gap with a two-phase commit:

1. `create_eph_layer` inserts the row `populated = false`.
2. the populate closure inserts the layer's graph rows;
3. `mark_populated()` flips the flag **in the same transaction** as the rows.

Because the flag and the rows commit atomically, a layer is only ever observed
*fully alive* or *absent* — never hollow-but-populated. This is what makes the
"same hash ⇒ same content" invariant safe under concurrency and retries.

## Sharding the query by layer

A command's **content map** \(f\) ([The Command Algebra](/docs/design/command-algebra)) is **layer-agnostic** (see [Layers](/docs/design/layers)): it computes its
result over whatever layers are visible and treats all of them the same. More
than that, a *decomposable* content map — one satisfying the
[layer-decomposability axiom](/docs/design/layer-tree/#2-verb-semantics-and-the-decomposition-axiom);
content populates qualify, cross-layer operations like `layer { … }` do not —
is a **union-homomorphism**:

```
f(L₁ ∪ L₂ ∪ … ∪ Lₙ)  =  f(L₁) ∪ f(L₂) ∪ … ∪ f(Lₙ)
```

so its result over a set of layers is the union of its result over each layer on
its own. That composability is the lever the executor pulls. Because the shards
compose by union, each `f(Lₖ)` can be computed **independently** — which is
exactly what makes both **parallelism** (shards run concurrently) and
**cacheability** (each shard keyed and stored on its own) possible. This document
is about the caching half; the same property is what lets the executor fan the
work out across connections.

So instead of running one query over the whole visible set and keying the result
on that set, the executor **shards the content map by layer**: it runs `f(Lₖ)` for each
visible layer, caches each shard on its own key, and composes the shards by
union. `f(root)` and `f(eph layer)` are the *same operation* applied to different
layers — one query, sharded, not two different computations.

```
      f over the visible set  =  union of one shard per layer
                            │
   ┌───────────┬────────────┼────────────┬───────────┐
   ▼           ▼            ▼            ▼           ▼
 f(root)     f(L₁)        f(L₂)   …    f(Lₙ)
   │           │            │            │
 key:        key:         key:         key:
 (root id,   (inputs,     (inputs,     (inputs,
  inputs)     L₁)          L₂)          Lₙ)
   └───────────┴──── each shard caches on its OWN key ────┴───────┘
```

The win is that each shard's key is stable: `f(Lₖ)` is keyed on
`(verb inputs, Lₖ)` alone, so any later query whose visible set contains `Lₖ`
reuses that shard, and adding a layer re-runs only the one new shard. Keying a
single fused result on the whole visible set instead would throw the cache away
every time the set changes.

Two consequences follow from *which* layer a shard is over — the only asymmetry
between them:

- **The root shard is chain-independent.** A root layer's identity — a
  project's committed index — does not depend on any query's ephemeral chain, so
  `f(root)` is keyed by `root_shard_hash(root, command_inputs)` and nothing else.
  It is cached once and reused under every chain and every co-visible set of
  projects. This matters because, for a content populate, the root shard does all the
  expensive work (for `search`, the corpus scan). Salting per *root* (not per visible set)
  means identical inputs against different projects key different shards, and a
  project's shard is reused no matter which other projects are co-visible.
  In the code: `root_shard_populate`, keyed by `root_shard_hash(root, input_hash)`.
- **A layer shard is keyed on its layer.** `f(Lₖ)` keys on that layer's id,
  so it is reused exactly when `Lₖ` is visible again —
  extending the visible set re-materialises only the new layer's shard; every
  older shard is a hit. Layer shards are **lifetime-agnostic**: a future *persistent*
  delta layer rides the same mechanism, its layer shards simply becoming durable
  cache wins ("the root shard scans the heavy corpus" is a cost assumption — deltas
  are assumed light).

The third node kind, the **selection shard**, holds whatever does not decompose
per layer (see below); together root shard, layer shard, and selection shard are the three node
kinds of the [layer tree](/docs/design/layer-tree/#3-partitioning-the-three-node-kinds).

This decomposition is exact for **decomposable verbs under masking-free
composition** — union is associative and commutative (see
[Layers](/docs/design/layers)); the non-decomposable remainder lives in the
selection shard. Today ephemeral
layers hold no *content* for a content populate to read, so every eph-layer shard is
empty and the root shard carries the whole result; the per-layer machinery
becomes observable once ephemeral layers carry content.

> **Note:** The verb supplies only the composable query — written as if it could
> read any layer; the executor alone decides how to split it by layer id, run the
> shards, and key each one. No verb branches on "persistent vs ephemeral."

### When a shard reads more than one layer

A pure content shard `f(Lₖ)` depends only on the command's inputs and the rows on
`Lₖ`. Some operations depend on *more*: a `layer { … }` block's ops can name
specific ephemeral ids from earlier in the chain, so that shard's contents depend
on those referenced ids, not on a single layer alone. **`selection_extra`** is
that extra key material — whatever a shard reads *beyond* the one layer it is
over. It is empty for content populates (`search`, `loc`); it is non-empty only for
ops that reference ephemeral ids, which are then keyed on those ids too, so a
block can't collide with one that shares everything else but its references.

## Composition is union in two dimensions

The same union that composes *layers* also composes *selectors*. Several
selectors in one command are ORed:

```askl
search("foo") search("bar")   // union of both searches, one merged layer
```

The executor folds the selector parts into one set of shards per root: every part
runs into the same root shard, keyed on the parts in source order so two
commands that differ only in their parts key differently. Both axes — over
selectors and over layers — compose by the same associative union, which is why
the cache can key each shard independently and still reconstruct the whole by
set-union at read time.

## Filter-aware hashing

A verb's cache key has to reflect the filters in scope, or `project("X")
search(q)` would collide with an unscoped `search(q)`. It does, without
special-casing any filter type: the root-shard hash folds
`CompositeFilter::hash_into(&mut Sha256)` — the full recursive hash of the
surrounding command's And/Or/Not/Leaf filter tree. When any leaf's semantics
change (a different project name, say), the whole root-shard hash changes and the cache
falls through to a fresh population. Any new filter that constrains objects gets
this for free — no per-filter branch in the caching code.

Every hash also begins with a **domain tag** — `b"root-shard-v1"`,
`b"selection-shard-v1"`, and so on. The tags keep the different key families
disjoint from one another and from raw verb hashes, and they carry a version. On
a keying change the tag is bumped (`v1 → v2`); old entries simply never hit again
and age out via TTL, so an upgrade needs **no cache purge** — stale rows are
stranded, not aliased.

## Lifetime and invalidation

An ephemeral layer's life is bounded three ways:

- **LRU.** Every cache hit bumps `last_used` (the upsert's `ON CONFLICT DO
  UPDATE`). Hot layers stay warm for free.
- **TTL.** A sweep drops layers whose `last_used` is older than a window
  (`purge_old_eph_layers`), reclaiming space held by one-off queries.
- **Cascade.** Deleting a layer removes both its child layers
  (`parent_id ON DELETE CASCADE`) **and the data rows tagged with it** — every
  data table's `layer` FK cascades, so purging a layer drops its objects,
  symbols, instances, and refs in one step rather than leaving them dangling.
  Separately, an eph-layer shard's `root_shard_id` couples its lifetime to the root
  shard it was cached against — so it can never outlive, and mis-hit against, a
  recreated root shard of a different incarnation.

**Re-upload is the hard invalidation.** Finalising a project upload — the
transaction that flips its status to complete — also calls `purge_eph_cache`,
dropping every non-canary ephemeral layer atomically with the upload commit. This
keeps the correctness argument short: an ephemeral layer only ever reflects a
corpus state that was current when it was populated, and it is discarded the
moment that state changes. The canary sentinel is a protected kind and survives
every purge.

## Correctness invariants, summarised

- **The root shard is a function of `(root identity, verb inputs)` only** —
  its parent in the [layer tree](/docs/design/layer-tree) is
  the root, so no other ephemeral layer enters its key. This is what makes it
  reusable across every context, and the property that the per-root salt and
  the domain tag exist to protect.
- **Same hash ⇒ same content**, enforced by content-addressing plus the
  two-phase `populated` flag, so a hit can safely skip the populate.
- **A cached entry never outlives its state.** The in-RAM tier is cleared only by
  persistent-corpus mutations (upload finalise, project delete) — not by
  ephemeral-layer churn, which keys differently and needs no clear — and the
  ephemeral tier is purged on re-upload. Neither can serve rows from a corpus
  state that no longer exists.
- **Cross-project shared content stays scoped.** Deduplicated `content_store`
  rows are disambiguated by the object's project, bound into the query — see
  [Design: search()](/docs/design/search) for the invariant and its test.
