---
title: "Overview"
description: "How the askl engine works: the life of a query, the model behind each stage, and the terminology the design pages share"
weight: 90
---
These pages are long-form technical documents about *how* askl works
under the hood — algorithms, data models, correctness invariants, and
the trade-offs that shaped them. They are aimed at contributors, and at
users who want a full picture of what happens when they run a query;
the [Askl Syntax Reference](/docs/syntax) is the right first stop for
using the language.

## The life of a query

A query is a sequence of **statements**, and statements are the only
time axis the language has: they run in source order, one after
another, and nesting inside a statement expresses constraint rather
than sequence.

**Parsing** turns the text into a forest of substatements. Each one
carries a **command** — its bag of verbs, folded in source order into a
single filter predicate and a single combined populate
([Queries and their Meaning](/docs/design/semantics)). Two structural
rules are checked before anything runs: every component must contain an
anchor, so that a query cannot ask for "everything" by accident, and a
`@label` may only reference an earlier statement.

Each statement then runs three phases before the next one starts
([Evaluating the Fixpoint](/docs/design/evaluation)):

1. **Materialise.** Commands that produce content — `search`, `loc`,
   `layer { … }` — write their rows into new layers. Those layers are
   the statement's *materialisation*, and they are carved into nodes
   named by a content hash, so an identical populate from any earlier
   query is a cache hit rather than a rerun. Why the carve looks the
   way it does is [Partitioning a Materialisation](/docs/design/shards); what goes
   into the names is [Layer Keys and Hashing](/docs/design/layer-keys).
2. **Probe.** Before reading anything in full, the engine measures.
   Capped id probes find which substatements are small enough to
   enumerate exactly, and refinement waves let one selective leaf drive
   the rest of the query
   ([Cost-Based Execution](/docs/design/cost-based-execution)) — this is
   what keeps a query like `mod("amdgpu") { func { "drm_dev_enter" } }`
   from resolving millions of rows it will immediately discard.
3. **Read.** Each command reads its rows under the layers now visible,
   conjoining its filter predicate.

**Composition** finishes the job. A monotone worklist propagates
constraints between neighbouring substatements — a parent narrows its
children, children narrow their parent — until nothing changes. What
survives that fixpoint is the query's answer
([Evaluating the Fixpoint](/docs/design/evaluation)).

Underneath, two caches serve the whole pipeline: a database-backed
cache of materialised layers, shared across queries and processes, and
an in-RAM cache of read results within one process
([Caching](/docs/design/caching)).

Each of those steps exists for a reason, and the reasons compose into
one argument — from what a query returns down to why its results are
cached the way they are. That derivation, with the algebra, is
[From Result to Cache](/docs/design/derivation).

## Terminology

The design pages use these terms with fixed meanings:

- **query** — the whole parsed input: a sequence of statements.
- **statement** — one whole top-level unit of the query: a command, its
  scope, and everything nested under it. `"foo" { "bar" }` is one
  statement. Statements are the query's only time axis (`;` or a
  newline separates them; nesting never sequences), and a `@label` may
  only be referenced from a *later* statement — same-statement or
  forward references are parse errors.
- **substatement** — any command-plus-scope node within a statement, at
  any depth. By convention a statement is a substatement of itself, so
  machinery that is uniform over nodes (weakness, probes, hashes)
  is stated once, per substatement. The engine's `Statement` type
  corresponds to *substatement*.
- **command** — the verb bag of one substatement, assembled by folding
  its verbs in source order; a later verb may override an earlier
  same-tagged one ([Queries and their Meaning](/docs/design/semantics)).
  Filters, predicates, and cache hashes are per-command.
- **verb** — the generic execution unit inside a command: `search(...)`,
  `project(...)`, `func(...)`. What a verb contributes varies — a
  filter, layer content, or both.
- **filter** — a unit of filter composition: one predicate leaf of the
  command's combined predicate. Most filters are contributed by verbs;
  a bare name string contributes one without being a verb.
- **component** — one or more statements connected by label references;
  the unit of the anchor-completeness rule and of bindness.
- **selection** — the set of instances that survives the worklist
  fixpoint, written \(N_c\) for one command and \(N_s\) for a whole
  statement ([semantics](/docs/design/semantics/#selections)). The
  referenced outputs \(O_c\) of a command are earlier statements'
  selections. Distinct from the selection *function* \(\sigma_g\), which
  filters a row set.
- **materialisation** — the set of layers one layer-creating statement
  produces on one project, appended to visibility atomically
  (`push_materialisation`). Statements and their materialisations
  correspond one-to-one, so \(t\) indexes both.
- **shard** — a node holding one dependency's part of a command's
  materialisation. The **input shards** \(\mathrm{Sh}_c(\ell)\) are
  keyed by their input plus \(H(c)\): the root shard
  (\(\mathrm{Sh}_c(R)\)) over the persistent corpus, and one layer shard
  per pre-statement ephemeral layer. The **selection shard** (next entry)
  is keyed on the chain.
  Reuse scope is per kind: input shards are shared wherever their
  input is visible; the selection shard only along its chain.
- **selection shard** — the per-command node parented on the previous
  statement's tip (\(S_c\)); holds everything built on outputs — the
  materialisation's selection-dependent term — and marks the chain position
  when empty. A statement's selection-shard-bearing commands contribute *sibling*
  selection shards; the new tip is the materialisation's last layer in pre-order.
- **wave** — one probe iteration of the cost-based executor (wave 0 plus
  the refinement waves), distinct from a materialisation.

## Where to start

- **To understand what a query *means***, read
  [Queries and their Meaning](/docs/design/semantics) for composition and
  weakness, then [Evaluating the Fixpoint](/docs/design/evaluation) for
  how that meaning is computed and
  [Cost-Based Execution](/docs/design/cost-based-execution) for how it is
  computed cheaply.
- **To understand how results are stored and reused**, read
  [Layers and layer operations](/docs/design/layers) for the data model,
  then [Partitioning a Materialisation](/docs/design/shards) — it derives
  the node kinds from the caching problem — and
  [Layer Keys and Hashing](/docs/design/layer-keys) and
  [Caching](/docs/design/caching) for the mechanics.
- **To understand one verb end to end**, read
  [search()](/docs/design/search).

## Pages

**Evaluating a query**

- **[From Result to Cache](/docs/design/derivation)** — why the engine
  has the parts it has, derived from what a query returns.
- **[Queries and their Meaning](/docs/design/semantics)** — how verbs
  fold into one predicate and one populate, and why a query's answer is
  a fixpoint of mutually constraining selections.
- **[Evaluating the Fixpoint](/docs/design/evaluation)** — the phases a
  statement runs, and the monotone worklist that composes
  neighbouring substatements to a fixpoint.
- **[Cost-Based Execution](/docs/design/cost-based-execution)** —
  planning from measured cardinality: anchors, capped id probes, and
  the refinement waves that let one selective leaf drive a query.

**Storing and reusing results**

- **[Layers and layer operations](/docs/design/layers)** — the layer
  data model, how a query decides what it can see, and the isolation
  guarantee.
- **[Partitioning a Materialisation](/docs/design/shards)** — how a
  statement's materialisation is split into independently cached shards,
  and what their keys guarantee.
- **[Layer Tree Extensions](/docs/design/layer-tree-extensions)** — the
  same model with several projects visible, and with content on
  non-root layers.
- **[Layer Keys and Hashing](/docs/design/layer-keys)** — how a
  command's verbs, filters, and context become those keys.
- **[Caching](/docs/design/caching)** — the content-addressed source
  store, the two request-time tiers, and the invariants that keep a
  cached answer from outliving its state.

**One verb in depth**

- **[search()](/docs/design/search)** — turning a literal
  string into byte-range matches over the whole corpus, without regex
  and with byte-exact positions.
