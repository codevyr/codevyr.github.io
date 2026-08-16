---
title: "Caching"
description: "Two caches for the two stages of expensive work — producing rows in the database and re-reading them in memory — and the invariants that stop a cached answer from outliving the state it was computed from"
weight: 150
---

Askl queries are interactive and the corpus is large — a representative
deployment holds ~1.9 GB of source content. A `search()` over a common
term, or a deep call-graph walk, is expensive; running it again a second
later, or as one branch of a bigger composed query, should be nearly
free, and never at the price of a stale answer.

Expensive work happens in two stages, so it is cached twice. A command
**produces** rows — scanning the corpus, materialising an ephemeral
layer — and then everything downstream **reads** those rows back, usually
with the identical query. Production is cached in the database, where the
rows already are; re-reading is cached in process memory. Production is
the harder half, because an entry there is a promise that a hash names a
fixed set of rows, and most of this page is the machinery that keeps the
promise. It assumes the [layer data model](/docs/design/layers), and the
shards it stores are derived in
[Partitioning a Materialisation](/docs/design/shards).

## 1. Underneath both: content-addressed source {#content-store}

Before either tier there is the corpus. Every file's bytes live in
`content_store` keyed by the SHA-256 of those bytes, so identical content
across projects and across re-uploads costs one row — and, less obviously,
one *tokenisation*: the generated columns `search()` scans are computed
once per unique content, so a re-upload re-indexes only the files whose
bytes actually changed. It is the same bargain the tiers strike, keyed
differently — content-addressing the **source** reuses stored bytes and
derived indexes, where a layer cache hashes a verb's **inputs** to reuse
its computed output. The storage layout and the cross-project invariant
belong to [search()](/docs/design/search).

## 2. Two tiers {#two-tiers}

| | In-RAM SQL result cache | Ephemeral-layer cache |
|---|---|---|
| Lives in | process memory | the database (`index.layers` + rows) |
| Keys on | the exact SQL + binds | a content hash of the command's inputs |
| Caches | the rows a read query returned | a materialised layer of graph rows |
| Bounded by | bytes (`ASKL_SQL_CACHE_BYTES`) | TTL + LRU, wholesale purge on re-upload |
| Survives | one server process | across processes (it's in the DB) |

The stages are what keep the two from overlapping. A hit in the layer
tier means a populate never runs: no corpus scan, no rows inserted. A hit
in the in-RAM tier means an already-materialised row set is not fetched
again. Neither substitutes for the other.

## 3. Tier 1: reads, in process memory {#sql-result-cache}

The read path (`cached_load`) is a thin proxy in front of the database:
it builds a read query with its binds, hashes them into a key, and serves
the cached row set on a hit. It is a byte-bounded LRU, evicting once the
cache exceeds `ASKL_SQL_CACHE_BYTES` and degrading to a plain database
load when that budget is `0`.

What makes it correct is that **ephemeral-layer churn needs no
invalidation at all** — which matters, since there is a great deal of it.
The key is the hash of the rendered query together with its binds, and
because every visibility predicate is a `layer = ANY($visible)` clause,
the visible layer ids are among those binds: a read keys on the exact set
of layers it could see. Materialising a layer changes the binds and so
the key — a miss, never a stale hit against an entry built under a
different layer set. Layer ids are never reused, so a dead layer's
entries can never be requested again and age out through the LRU.

Mutations to the *persistent* corpus are the exception, and the only
thing that ever clears this cache: finalising an upload or deleting a
project clears it wholesale, on commit. An epoch counter closes the race
with an in-flight load — a load snapshots the epoch before its database
read, the clear bumps it, and a put carrying a stale snapshot is rejected
— so a read issued before the mutation can never land in the cache after
it. That one rule is the whole guarantee that a cached row set never
outlives the corpus state it was read from.

## 4. Tier 2: production, in the database {#layer-cache}

Every ephemeral layer is a **content-addressed cache entry**: its `hash`
column is a function of the inputs that determine its contents, and it
carries a `UNIQUE` index. Materialising is therefore a single upsert, and
the upsert *is* the lookup:

```
create_eph_layer(parent_id, root_shard_id, hash, kind):
    INSERT INTO layers (hash, ...) VALUES (...)
    ON CONFLICT (hash) DO UPDATE SET last_used = now()
    RETURNING id, (xmax = 0) AS created, populated
```

`xmax = 0` distinguishes the row this statement inserted — a **miss** —
from one that was already there, a **hit**. On a hit the expensive
populate is skipped and `last_used` is bumped by the same statement, so
the LRU touch costs nothing extra; on a miss the caller runs the populate
closure to fill the layer's rows.

An entry is not a whole command's output: a materialisation is
partitioned into shards, each keyed on the one input it reads, and it is
a *shard* that gets a hash and a layer row
([Partitioning a Materialisation](/docs/design/shards/#node-kinds)
derives the partition and its keys). Splitting is the executor's doing,
never the verb's — a verb supplies one populate, written as if it could
read any layer, and nothing in it branches on persistent versus
ephemeral.

One consequence of the partition is not about caching at all. Because the
shards compose by union, each is not merely cacheable on its own key but
*computable* on its own, so the executor fans them out across connections
and runs them concurrently: cacheability and parallelism are the same
property read twice.

### 4.1. The two-phase `populated` guard {#populated-guard}

A cache key is a promise: a row with this hash holds exactly this
content. A half-built layer would break it — a concurrent reader could
hit the hash, skip the populate, and read a partial layer. So the row is
written in two phases: `create_eph_layer` inserts it with `populated`
false, the populate closure inserts the layer's graph rows, and
`mark_populated()` flips the flag **in the same transaction** as those
rows. Flag and rows committing atomically is what makes "same hash ⇒ same
content" safe under concurrency and retries: a layer is only ever
observed fully alive or absent, never hollow-but-present.

## 5. What a key must name {#filter-aware-hashing}

Two demands land on the key. It must reflect the filters in scope, or
`project("X") search(q)` would collide with an unscoped `search(q)` —
met by folding the whole filter tree into the hash, which is the only
filter-awareness anywhere in the cache and costs no per-filter branch
([Layer Keys and Hashing](/docs/design/layer-keys/#filter-hash) has the
bytes).

It must also be versioned, which is what the **domain tag** every hash
begins with is for. Keeping key families disjoint is the obvious half;
the version is the upgrade path. A keying change bumps the tag, after
which old entries can never hit again and age out on their own — so a
change of keying scheme needs **no purge**, and the entries it obsoletes
are stranded rather than aliased.

## 6. Lifetime and invalidation {#lifetime-and-invalidation}

An ephemeral entry is meant to die, and two mechanisms retire entries by
age: every hit bumps `last_used` through the upsert's
`ON CONFLICT DO UPDATE`, so hot entries stay warm for free, and a TTL
sweep drops entries whose `last_used` has fallen outside the window. What
a delete takes with it — child layers, and every data row tagged with the
layer — belongs to the layer model
([Layers §8](/docs/design/layers/#lifetime-and-gc)). One coupling is this
page's own: a shard's `root_shard_id` ties it to the root shard it was
cached against, so it can never outlive, and then mis-hit against, a
recreated root shard of a different incarnation.

Age is not an invalidation, though, and **re-upload is**. Finalising a
project upload — the transaction that flips its status to complete — also
purges the ephemeral cache, dropping every non-canary layer atomically
with the upload commit, which keeps the correctness argument to one
sentence: an ephemeral layer only ever reflects a corpus state that was
current when it was populated, and it goes the moment that state changes.

Both tiers, then, run on one discipline: an entry names the state it was
computed from, so it can be dropped the moment that state changes and can
never quietly outlive it. The read tier gets there by keying on the layer
ids it saw and clearing on corpus mutation; the produce tier by
content-addressing its inputs, promising content only once the rows are
committed, and being purged wholesale when the corpus is replaced.
