---
title: "Layers and Layer Operations"
description: "The layer data model — layer kinds and lifetimes, how a query decides what it can see, the operations that create ephemeral layers, and the isolation guarantee that keeps queries from leaking rows into one another"
weight: 138
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
how a statement's layers are carved into independently cached shards, see
[Partitioning a Materialisation](/docs/design/shards); for the cache tiers
themselves — content-addressed hashing, the two-phase guard, and lifetime — see
[Caching](/docs/design/caching).

## 1. Why ephemeral layers exist {#why-ephemeral-layers}

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
document and [Caching](/docs/design/caching) are about.

## 2. What a layer is {#what-a-layer-is}

A layer is a row in `index.layers` plus every graph row tagged with its id. The
tag is a plain `layer BIGINT` column carried by the data tables:

```
layers {
    id         BIGINT PRIMARY KEY   -- the layer's id
    parent_id  BIGINT               -- the layer this one was chained onto
    root_shard_id BIGINT            -- ties a shard's lifetime to the root shard it was cached against
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
layers (see [search()](/docs/design/search)). Everything else that a
query can select, filter, or walk carries a layer.

Because every row carries its layer, "which rows does this query see?" reduces to
"which layers are visible?", and every visibility predicate is a single
`layer = ANY($visible)` clause. Queries never reason about the *shape* of the
layer graph — only about a flat set of visible ids.

## 3. Kinds and lifetimes {#kinds-and-lifetimes}

A layer has a **kind** (its structural role: `root`, `canary`, or
`ephemeral`) and, orthogonally, a **lifetime** — persistent (committed
index state, surviving across queries) or ephemeral (scoped to one
query). Lifetime is independent of a layer's position in the forest:
where a layer sits and how long it lives are separate facts. The two
lifetimes compare as:

| | Persistent | Ephemeral |
|---|---|---|
| Created by | uploading/indexing a project | a verb, during a query |
| Lifetime | permanent (never garbage-collected) | cache entry (TTL / LRU) |
| Holds | the indexed corpus | one verb's per-query output |
| Count | a project's persistent closure | many, transient |

Both kinds live in the *same* tables, told apart only by their `layer` tag — so a
query includes or excludes either simply through which layer ids it makes
visible. Nothing else about a row depends on which kind of layer it sits on.

**The persistent closure.** A project's persistent layers taken together are
its **persistent closure**: the **root layer** \(R_p\) — the project's
*initial* persistent layer, carrying a stored **identity hash** \(h(R_p)\)
that names the committed state it stands for
([Partitioning a Materialisation §4](/docs/design/shards/#key-trust)
makes that naming precise) — plus, in general, any further persistent
**delta layers** committed by incremental index updates.

> **Persistent-prefix invariant.** Persistent and ephemeral layers never
> interleave: per project the visible slice is always the persistent closure
> (in commit order) followed by an ephemeral suffix (the query's
> materialisations) — committed state never depends on query-scoped state.

**The canary.** One ephemeral layer is a self-contained sentinel — its own
project, object, symbol, and instance — used to prove that a query which
should see *nothing* really sees nothing.

> **Deployment status.** The above is the model; the running system
> implements a proper subset of it. A project's persistent closure is
> today just its root layer — persistent delta layers are not yet
> representable, which is why lifetime and kind still coincide and why
> `root_ids()` (§4) returns exactly one layer per project. The content
> verbs' populates are already layer-agnostic but are so far only ever
> run over the root (§6.1); when delta layers land they join the
> protected kinds of §8, and the intended invalidation rule is that
> committing a delta purges nothing keyed on ids that still exist — only
> replacing a project's corpus does. Nothing in the model depends on the
> restriction; [Layer Tree Extensions](/docs/design/layer-tree-extensions)
> covers deltas.

## 4. Visibility: roots and materialisations {#visibility}

The layers of a root form a tree, carved and parented as
[Partitioning a Materialisation](/docs/design/shards) describes. What a
running query carries is much flatter: an **`EphContext`** — the set of
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
- **`root_ids()`** = just the roots — the persistent slice of visibility,
  which is each visible project's
  [persistent closure](#kinds-and-lifetimes).

A context always starts *rooted* in an explicit set of root layers (an empty set
is legal and means "no persistent data visible" — used by unit tests and the
canary). The visible set only ever grows, and only through one path.

## 5. Materialisations: how a query accretes layers {#materialisations}

Askl evaluates a query one top-level **statement** at a time, in source
order. A statement whose verbs create ephemeral layers (a `search(...)`,
a `loc(...)`, a `layer { … }` block) materialises its layers, then
appends them to every visible root's set as one atomic
**materialisation**:

```askl
layer { ... } ;        // statement 1 → materialisation A becomes visible on each root
search("kmalloc") { }  // statement 2 → sees A, then materialisation B joins
```

Inside a statement, though, nothing is sequenced. A visibility snapshot
is taken once, before any of the statement's commands materialise, and every
`@label` argument is resolved up front from the completed selection of
the *earlier* statement that defines it — which is the whole reason for
the ordering rule of [§6.2](#manual-constructor), and the reason no
command ever has to read a statement-mate's result mid-flight. The
statement's layer-bearing commands then materialise **concurrently**
against that one pre-statement snapshot, independent by construction.
Outcomes are applied in command pre-order, so completion order is
unobservable, and each command keeps its own layers — attribution by
layer id is what gives every command its own selection.

The append is **lockstep**: a materialisation must contribute a layer for *every* visible
root (one that misses a root, or names an unknown one, is a programming error
and panics). This keeps the per-root layer lists parallel, so `visible_ids()` is
always a coherent snapshot. Nesting adds to the *same* materialisation, never a new
one: `search("a") { search("b") }` is one statement, and the
inner populate does not see the outer's layers — only the next statement's
commands do.

A root's **`tip`** is its spine tip — the last layer of the most recent
materialisation in command pre-order — and it is what the *next*
statement grows from: that statement's selection-shard-bearing commands
hang their nodes off the tip as siblings, while the rest of a
materialisation parents elsewhere in the tree
([Partitioning a Materialisation](/docs/design/shards) has the rule per
node kind). Parentage is not what makes an earlier statement's rows
readable, though — visibility is. Every materialisation's layers are in
`visible_ids()` when the next statement's queries run, whatever they hang
off.

> **Note:** The chain topology is decided entirely by the **executor**, from the
> live `EphContext` — never by the verb. A verb describes *what* rows it wants
> to materialise; the executor decides which root each layer hangs off and in
> what order. This removes a whole class of parent-disagreement bugs.

## 6. Layer operations {#layer-operations}

Three verbs create ephemeral layers. Two are **content verbs** and one is a
**manual constructor**.

### 6.1. `search()` and `loc()` — content verbs {#content-verbs}

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
the executor decides which layers to run it over, so ephemeral *content*
layers will flow through the same populate without new verb code. How the
executor splits that populate into shards is
[Partitioning a Materialisation](/docs/design/shards); how the shards are stored,
keyed, and invalidated is [Caching](/docs/design/caching).

### 6.2. `layer { … }` — the manual constructor {#manual-constructor}

A `layer { … }` block builds ephemeral graph rows directly from **ephemeral
operations**, without a populate:

- `ephemeral_symbol(...)` — synthesise a symbol row on the new layer;
- `ephemeral_instance(symbol_id=…, object_id=…, start=…, end=…, …)` — attach an
  instance;
- `ephemeral_ref(from_object=…, to_symbol=…, …)` — add a reference edge.

Ops may reference persistent ids or the ephemeral ids produced by *earlier
statements'* layers — a `@label` argument must name an earlier top-level
statement, and same-statement or forward references are parse errors.
That rule is what lets every label be resolved before materialisation
begins ([§5](#materialisations)).
The block validates every referenced id against the visible project set before
committing, so a typo'd id fails loudly instead of committing a hollow layer.

`layer { … }` is the low-level primitive the content verbs are built on
conceptually, and the seam through which synthetic or externally-supplied graph
rows can be injected into a query. It is an advanced operation; most queries
never write one directly.

## 7. Isolation: no query sees another's rows {#isolation}

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

## 8. Lifetime and garbage collection {#lifetime-and-gc}

Ephemeral layers are cache entries, so *when* one is dropped is a caching
policy — content-addressing, the two-phase `populated` guard, LRU, TTL,
and the wholesale purge on re-upload, all in
[Caching](/docs/design/caching). Two things about dropping one are
properties of the data model instead, and belong here.

**Some layers are never candidates.** Persistent layers are permanent —
deleting a root would cascade an entire project's index away — so their
kind is *protected*, excluded from every purge; the canary of §3 is
protected too, being the sentinel a purge would otherwise quietly
invalidate.

**Deleting a layer deletes its rows.** The delete cascades both to child
layers (`parent_id`) and to every data row tagged with the layer (each
data table's `layer` FK cascades), so a purge never leaves orphaned
objects, symbols, instances, or refs behind. This is the same property as
§2's: a row belongs to exactly one layer, and it has no existence apart
from it.

## 9. Composition is union; masking is future {#composition-is-union}

Today layers only ever **add** rows. A query's result is the union of what each
visible layer contributes, and union is associative and commutative — the order
of the chain doesn't change *what* you see, only *when* a layer became visible.
That is the premise the whole cache rests on: it is what lets a command's
contribution be carved per layer and each part cached independently
([Partitioning a Materialisation](/docs/design/shards/#decomposition-axiom)).
The carve is therefore exact only for **masking-free composition** — which is
what the next paragraph reserves.

**Masking** — a higher layer *removing* or *shadowing* a lower layer's rows — is
a deliberate non-feature for now. It would make composition order-dependent (a
fold, not a union) and is out of scope until there is a concrete need. Nothing in the
data model would have to change: a mask would be one more content-bearing
layer, subtracting where the others add.
