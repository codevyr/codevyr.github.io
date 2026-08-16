---
title: "Layer Tree Extensions"
description: "Two generalisations of the layer tree that need no new machinery: multiple visible roots, and content on non-root layers including persistent deltas"
weight: 141
---

[Partitioning a Materialisation](/docs/design/shards) develops its model over one
project whose root is the only content-carrying layer. This page lifts
both restrictions and shows that neither costs any new machinery.

## 1. Multiple roots {#multiple-roots}

The core page fixed one project throughout; this section makes several
visible at once. With several projects visible, everything in
[shards §3](/docs/design/shards/#node-kinds)
happens **per project, in lockstep**: statement \(t\) appends one
materialisation for **every** \(R \in \mathcal{R}\) atomically. The
query's **visibility** — the allowlist it binds — is the union of the
per-project slices,

$$V_t \;=\; \bigcup_{R \,\in\, \mathcal{R}} \Lambda_t(R)$$

and because the materialisations land atomically, the per-root spines
stay parallel and \(V_t\) is always a coherent snapshot. Per-root
**salting** — folding \(h(R)\) into every root-shard key — keeps the trees'
key spaces disjoint: project A's `search("bar")` root shard and project B's
are different nodes, each cached independently of which *other*
projects happen to be visible alongside. (In the engine the two
non-root keys fold \(H(c)\) through the root shard's full root-salted
key, so they redundantly pin \(h(R)\) as well — a light layer already
determines its project.) Cross-root state never enters
any key, because no node's content is a function of another root's
content.

## 2. Content-producing layers {#content-producing-layers}

The core page also deferred two generalisations: layers other than the
root carrying corpus content, and the
[persistent delta layers](/docs/design/layers/#kinds-and-lifetimes) of
the layer data model. Neither needs new machinery.

The model treats "content" uniformly: \(C(\ell)\) may include corpus
content for **any** layer, not just roots. Today only roots carry
indexed content, so \(E_t(R)\) is empty and no layer shard is minted
at all. But nothing
in [shards §3](/docs/design/shards/#node-kinds)
assumed that. When a future verb writes content into an ephemeral
layer \(\ell\) (say, a generated or patched source overlay):

- \(\ell \in E_t(R)\) for subsequent statements, so each later command's
  populate materialises a genuine \(\mathrm{Sh}_c(\ell) = U_c(C(\ell))\);
- [the carve](/docs/design/shards/#node-kinds)
  already sums it into the materialisation's read, and already shares
  it with every other context containing \(\ell\);
- key soundness holds because \(\mathrm{id}(\ell)\) identifies content that
  — like every ephemeral layer — is immutable once populated.

The open engineering on that path is on the *producer* side (content rows
attributed to ephemeral layers, and index coverage for them), not in this
algebra.

The same reasoning covers **persistent delta layers**
([Layers](/docs/design/layers/#kinds-and-lifetimes)): a delta is a
content-carrying layer whose lifetime happens to be persistent. It joins
the closure, appears in \(E_t(R)\), and rides the **layer shard** mechanism —
a layer shard's key is layer identity plus the input hash \(H(c)\), with no
reference to lifetime, so a persistent layer shard is simply a durable cache
win. This also names the cost model honestly: "the root shard reads the root"
really means "the root shard reads the *heavy* corpus, and deltas are assumed
*light*". If a delta ever grew heavy, the remedy is compaction into the
root, not a change to this algebra.

## 3. Per-output selection shards {#per-output-shards}

Sibling selection shards exist today, at the statement level: each
selection-shard-bearing command of a statement contributes its own spine
node, all parented on the previous statement's tip, and the
deterministic tip over the siblings is already settled — the last
layer of the materialisation in command pre-order. What remains future is
splitting *within* a command, per **output**. Today a command's spine
node is one node: every output-derived piece (each `@label`'s rows)
folds into a single extra digest — a sound coarsening of the
derivation's
[output units](/docs/design/shards/#node-kinds),
free while selection-shard populates are batch inserts of precomputed rows.
If a future generative verb does heavy work per referenced selection
(deriving content from `@e`'s locations, say), the natural refinement
is one selection shard per output, keyed (spine, that output's resolved
ids). What follows is scheduling: each piece depends on one label's
read instead of all of them, so pieces materialise as their inputs
resolve — finer parallelism inside the materialisation. The structural
prerequisite — a deterministic tip over sibling selection shards — is no
longer an obstacle: the pre-order rule already provides it. Nothing in
the keying rule forbids the split; it is economics, revisited when the
cells stop being cheap.

## Where to read more

- [Partitioning a Materialisation](/docs/design/shards) — the notation, the
  decomposition axiom, the root-shard/layer-shard/selection-shard
  partition, and what its keys guarantee.
- [Layers and layer operations](/docs/design/layers) — the data model, the
  materialisation/lockstep mechanics, and the isolation guarantee.
- [Caching](/docs/design/caching) — the cache tiers, and the store, guard, and
  lifetime rules every shard rides on.
