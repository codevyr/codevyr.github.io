---
title: "Design: Layers and layer operations"
description: "The layer data model — layer kinds and lifetimes, how a query decides what it can see, the operations that create ephemeral layers, and the isolation guarantee that keeps queries from leaking rows into one another"
weight: 150
---

Every row in the askl graph — every object, symbol, instance, and reference —
belongs to exactly one **layer**. A layer is a labelled set of graph rows that a
query can choose to see or ignore. Layers are how askl keeps apart two things
that look identical from the SQL up: the **persistent index** built when you
upload a project, and the **ephemeral, per-query rows** a verb like `search()`
conjures on the fly.

This document is about that data model — what a layer is, the two kinds, how a
query decides which layers it can see, the operations that create ephemeral
layers, and how askl guarantees that one query can never see another's rows. For
how layers are *cached* — how the executor shards the content map by layer,
content-addressed hashing, and lifetime — see
[Design: Caching](/docs/design/caching).

## Why ephemeral layers exist

An ephemeral layer lets a query **modify the index** for the span of a single
command — overlaying new rows on top of the persistent index without mutating it.
That overlay is in service of one architectural goal: **keeping the entire
execution of a command at the SQL level.**

A verb like `search()` produces intermediate results (byte-range matches), and
the substatements downstream of it have to filter and compose over those results — scope
them with `project(...)`, walk their children, intersect them with other
selectors. All of that filtering and composition *already exists*, once, as SQL
over the persistent index. If those results instead lived only as Rust values
in memory, every filter and relationship would have to be reimplemented a second
time in Rust to apply to them — the same matching logic maintained in two places,
kept in sync by hand, and free to drift apart.

Ephemeral layers remove the duplication. A verb writes its results as ordinary
graph rows into a new layer, in the *same tables* as the persistent index. From
there on, downstream filtering and composition run the **same SQL** over
persistent and ephemeral rows alike — a query can't tell, and doesn't need to,
which layer a row came from. There is exactly one implementation of each filter,
and it lives in SQL. The price of the approach — intermediate results are
materialised as rows, and therefore have to be cached — is what the rest of this
document and [Design: Caching](/docs/design/caching) are about.

## What a layer is

A layer is a row in `index.layers` plus every graph row tagged with its id. The
tag is a plain `layer BIGINT` column carried by the data tables:

```
layers {
    id         BIGINT PRIMARY KEY   -- the layer's id
    parent_id  BIGINT               -- the layer this one was chained onto
    base_id    BIGINT               -- links an eph-layer shard to the root shard it caches against
    kind       TEXT                 -- 'root' | 'canary' | 'ephemeral' (coarse: not per-verb)
    hash       BYTEA UNIQUE         -- content-addressed cache key (see Caching)
    populated  BOOL                 -- two-phase-commit guard (see Caching)
    truncated  BOOL                 -- a verb limit capped this layer's contents
    last_used  TIMESTAMPTZ          -- LRU / TTL bookkeeping
}

objects           { id, project_id, ..., layer }   -- layer FK → layers(id)
symbols           { id, ..., layer }
symbol_instances  { id, ..., layer }
symbol_refs       { id, ..., layer }
```

`content_store` — the raw file bytes — is the one table with **no** layer
column: it is content-addressed by `content_hash` and shared across projects and
layers (see [Design: search()](/docs/design/search)). Everything else that a
query can select, filter, or walk carries a layer.

Because every row carries its layer, "which rows does this query see?" reduces to
"which layers are visible?", and every visibility predicate is a single
`layer = ANY($visible)` clause. Queries never reason about the *shape* of the
layer graph — only about a flat set of visible ids.

## Kinds and lifetimes

A layer has a **kind** (its structural role: `root`, `canary`, or
`ephemeral`) and, orthogonally, a **lifetime** — persistent (committed
index state) or ephemeral (query-scoped). In today's deployment the two
coincide — every persistent layer is a root, one per project — but the
model keeps them separate: incremental updates will add persistent
non-root delta layers (see the
[persistent closure](/docs/design/layer-tree/#1-objects)). The two
lifetimes compare as:

| | Persistent (today: the root) | Ephemeral |
|---|---|---|
| Created by | uploading/indexing a project | a verb, during a query |
| Lifetime | permanent (never garbage-collected) | cache entry (TTL / LRU) |
| Holds | the indexed corpus | one verb's per-query output |
| Count | one per project (`projects.root_layer_id`) today | many, transient |

Both kinds live in the *same* tables, told apart only by their `layer` tag — so a
query includes or excludes either simply through which layer ids it makes
visible. Nothing else about a row depends on which kind of layer it sits on.

> **Note:** One ephemeral layer, the **canary**, is a self-contained sentinel —
> its own project, object, symbol, and instance — used to prove that a query which
> should see *nothing* really sees nothing. It is a protected kind that garbage
> collection never touches.

## Visibility: roots and materialisations

The full structure the layers form is a **shared, content-addressed tree per
root** — root shards hang off the root, layer shards off the single layer they
shard, and only the selection shards form a query-ordered spine (see
[The Layer Tree](/docs/design/layer-tree) for the model and a worked example).
What a running query carries is much flatter: an **`EphContext`** — the set of
visible root layers plus, per root, the ordered list of ephemeral layers
materialised so far.

```
EphContext
├── roots:  [ root(projectA),  root(projectB) ]
└── layers: root A → [ A₁, A₂, A₃ ]   (ordered, most recent last)
            root B → [ B₁, B₂, B₃ ]
```

Two accessors turn this forest into the flat sets queries actually bind:

- **`visible_ids()`** = roots ∪ every ephemeral layer, flattened. This is the binding set
  for every `layer = ANY($visible)` predicate and the reference set for leak
  checks. Queries see the flat set, never the structure.
- **`root_ids()`** = just the roots — the persistent slice of visibility
  (today the *whole* persistent slice: one persistent layer per project;
  see the [persistent-prefix invariant](/docs/design/layer-tree/#1-objects)
  for the intended multi-layer closure).

A context always starts *rooted* in an explicit set of root layers (an empty set
is legal and means "no persistent data visible" — used by unit tests and the
canary). The visible set only ever grows, and only through one path.

## Materialisations: how a query accretes layers

Askl evaluates a query one top-level **statement** at a time, in source
order. A statement whose verbs
create ephemeral layers (a `search(...)`, a `loc(...)`, a `layer { … }` block)
materialises its layers — every layer-bearing command of the statement
computing against the same pre-statement visibility — then appends them
to every visible root's set as one
atomic **materialisation**:

```askl
layer { ... } ;        // statement 1 → materialisation A becomes visible on each root
search("kmalloc") { }  // statement 2 → sees A, then materialisation B joins
```

The append is **lockstep**: a materialisation must contribute a layer for *every* visible
root (one that misses a root, or names an unknown one, is a programming error
and panics). This keeps the per-root layer lists parallel, so `visible_ids()` is
always a coherent snapshot. Nesting adds to the *same* materialisation, never a new
one: `search("a") { search("b") }` is one statement, and the
inner populate does not see the outer's layers — only the next statement's
commands do.

A root's **`chain_last`** is its spine tip — the last layer of the most
recent materialisation in command pre-order — and the *next* statement's selection shards
hang off it: one per supplement-bearing command, **siblings** on the same
tip. Note what does and does
not hang there: only selection shards parent on the spine; the same materialisation's root shards
parent on the root and its layer shards on the individual layers they shard, which is
what keeps their cache keys context-free. Visibility, not parentage, is what
makes a later verb able to see an earlier statement's ephemeral rows: every
materialisation's layers are all in `visible_ids()` when the next statement's queries run.

> **Note:** The chain topology is decided entirely by the **executor**, from the
> live `EphContext` — never by the verb. A verb describes *what* rows it wants
> to materialise; the executor decides which root each layer hangs off and in
> what order. This removes a whole class of parent-disagreement bugs.

## Layer operations

Three verbs create ephemeral layers. Two are **content verbs** and one is a
**manual constructor**.

### `search()` and `loc()` — content verbs

`search("literal")` and `loc(path, line)` read the persistent corpus
(`content_store ⋈ objects`) and materialise their results as ephemeral graph
rows: one ephemeral symbol of type `SYMBOL_TYPE_CONTENT` per matching project,
and one instance per match (a byte range for `search`, a resolved line for
`loc`). The result is a first-class askl selection that downstream verbs compose
with like any other:

```askl
project("linux") search("EXPORT_SYMBOL") { }   // callees of every match
```

These verbs are **layer-agnostic**. Each provides one populate of the shape
`scan(txn, root, visible_layers, eph_branch)` (the code's `ShardedScan`);
the executor decides which layers
to run it over. Today it reads the persistent (root) layers; the same populate is
what future ephemeral *content* layers will flow through automatically. How the
executor shards and caches that populate is the subject of
[Design: Caching](/docs/design/caching).

### `layer { … }` — the manual constructor

A `layer { … }` block builds ephemeral graph rows directly from **ephemeral
operations**, without a populate:

- `ephemeral_symbol(...)` — synthesise a symbol row on the new layer;
- `ephemeral_instance(symbol_id=…, object_id=…, start=…, end=…, …)` — attach an
  instance;
- `ephemeral_ref(from_object=…, to_symbol=…, …)` — add a reference edge.

Ops may reference persistent ids or the ephemeral ids produced by *earlier
statements'* layers — a `@label` argument must name an earlier top-level
statement; same-statement references are parse errors (that is why the
block's selection shard is chained onto `chain_last`, the previous statement's tip).
The block validates every referenced id against the visible project set before
committing, so a typo'd id fails loudly instead of committing a hollow layer.

`layer { … }` is the low-level primitive the content verbs are built on
conceptually, and the seam through which synthetic or externally-supplied graph
rows can be injected into a query. It is an advanced operation; most queries
never write one directly.

## Isolation: no query sees another's rows

Ephemeral layers from unrelated queries coexist in the same tables. The
guarantee that they stay apart is a single invariant: **every row a query
returns must belong to a layer in that query's `visible_ids()`.** A row on any
other layer — persistent or ephemeral — is a *leak*.

The invariant is enforced at the type level. A selection can only be handed
downstream wrapped in a `Checked<T>`, and the only way to build one runs the leak
check first:

```rust
Checked::new(selection, eph)?   // bails (and logs) if any row's layer ∉ visible
```

A verb that returns a `Checked<Selection>` has, by construction, proven that
nothing inside it escapes the visible set. Combined with the
`layer = ANY($visible)` predicate on every read, this gives a two-sided
guarantee: the SQL only *fetches* visible rows, and the check *proves* it after
the fact.

## Lifetime and garbage collection

Persistent layers are permanent — deleting a root would cascade an entire
project's index away — so they are protected kinds, excluded from every
purge. (When persistent delta layers land, they join the protected set;
the intended invalidation rule is that committing a delta purges nothing
keyed on ids that still exist — only replacing a project's corpus does.)

Ephemeral layers are **cache entries**. Each is keyed by a content hash, its
`last_used` timestamp is bumped on every hit, and a TTL sweep drops layers that
haven't been used within a window. Re-uploading a project purges the ephemeral
cache wholesale so a subsequent query can't see stale results. Deleting a layer
cascades both to its child layers (`parent_id`) and to every data row tagged with
it (each data table's `layer` FK cascades), so a purge never leaves orphaned
objects, symbols, instances, or refs behind. The full lifetime story —
content-addressing, the two-phase `populated` guard, LRU, and TTL — is in
[Design: Caching](/docs/design/caching).

## Composition is union; masking is future

Today layers only ever **add** rows. A query's result is the union of what each
visible layer contributes, and union is associative and commutative — the order
of the chain doesn't change *what* you see, only *when* a layer became visible.
That is what lets the executor cache each layer's contribution independently (see
[Design: Caching](/docs/design/caching)).

**Masking** — a higher layer *removing* or *shadowing* a lower layer's rows — is
a deliberate non-feature for now. It would make composition order-dependent (a
fold, not a union) and is out of scope until there is a concrete need. The data
model already reserves room for it: `base_id` on an eph-layer shard records the
root shard whose rows a future mask would subtract from.
