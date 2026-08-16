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
([The Command Algebra](/docs/design/command-algebra)). Two structural
rules are checked before anything runs: every component must contain an
anchor, so that a query cannot ask for "everything" by accident, and a
`@label` may only reference an earlier statement.

Each statement then runs three phases before the next one starts
([Execution Engine](/docs/design/execution-engine)):

1. **Materialise.** Commands that produce content — `search`, `loc`,
   `layer { … }` — write their rows into new layers. Those layers are
   the statement's *materialisation*, and they are carved into nodes
   named by a content hash, so an identical populate from any earlier
   query is a cache hit rather than a rerun. Why the carve looks the
   way it does is [The Layer Tree](/docs/design/layer-tree); what goes
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
([Execution Engine](/docs/design/execution-engine)).

Underneath, two caches serve the whole pipeline: a database-backed
cache of materialised layers, shared across queries and processes, and
an in-RAM cache of read results within one process
([Caching](/docs/design/caching)).

## Where to start

- **To understand what a query *means***, read
  [Execution Engine](/docs/design/execution-engine) for composition and
  weakness, then [Cost-Based Execution](/docs/design/cost-based-execution)
  for how the same meaning is computed cheaply.
- **To understand how results are stored and reused**, read
  [The Layer Tree](/docs/design/layer-tree) first — it derives the node
  kinds from the caching problem — then
  [Layer Keys and Hashing](/docs/design/layer-keys) and
  [Caching](/docs/design/caching) for the mechanics.
- **To understand one verb end to end**, read
  [Design: search()](/docs/design/search).

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
  same-tagged one ([The Command Algebra](/docs/design/command-algebra)).
  Filters, predicates, and cache hashes are per-command.
- **verb** — the generic execution unit inside a command: `search(...)`,
  `project(...)`, `func(...)`. What a verb contributes varies — a
  filter, layer content, or both.
- **filter** — a unit of filter composition: one predicate leaf of the
  command's combined predicate. Most filters are contributed by verbs;
  a bare name string contributes one without being a verb.
- **component** — one or more statements connected by label references;
  the unit of the anchor-completeness rule and of bindness.
- **selection** — the set of instances a statement's read produces
  (the worklist fixpoint), written \(o(s)\): a statement's *output*.
  The referenced outputs \(O_c\) of a command are earlier statements' \(o(s')\).
  Distinct from the selection *function* \(\sigma_g\), which filters a
  row set.
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

## Pages

**Evaluating a query**

- **[Execution Engine](/docs/design/execution-engine)** — the phases a
  statement runs, and the monotone worklist that composes
  neighbouring substatements to a fixpoint.
- **[Cost-Based Execution](/docs/design/cost-based-execution)** —
  planning from measured cardinality: anchors, capped id probes, and
  the refinement waves that let one selective leaf drive a query.
- **[The Command Algebra](/docs/design/command-algebra)** — how verbs
  fold into a command, filters compose into one predicate, and content
  verbs union into one populate.

**Storing and reusing results**

- **[The Layer Tree](/docs/design/layer-tree)** — what a statement
  materialises, how it is carved into cached nodes, and what their
  keys guarantee.
- **[Layer Tree Extensions](/docs/design/layer-tree-extensions)** — the
  same model with several projects visible, and with content on
  non-root layers.
- **[Layer Keys and Hashing](/docs/design/layer-keys)** — how a
  command's verbs, filters, and context become those keys.
- **[Layers and layer operations](/docs/design/layers)** — the layer
  data model, how a query decides what it can see, and the isolation
  guarantee.
- **[Caching](/docs/design/caching)** — the two cache tiers and how the
  executor splits a populate so the expensive part is paid once.

**One verb in depth**

- **[Design: search()](/docs/design/search)** — turning a literal
  string into byte-range matches over the whole corpus, without regex
  and with byte-exact positions.
