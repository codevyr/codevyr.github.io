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

**The product.** A query returns a graph: \(\mathrm{askl}(\text{query}) = \{\text{nodes}, \text{edges}\}\) — a
subgraph of the index, symbols together with the reference and
containment edges among them. Everything below exists to compute that
graph, and then to avoid computing it twice.

**A query is an ordered list of statements**, and its graph is the
union of what each statement selects. Statements are the only time
axis: they run in source order, and nesting inside one expresses
constraint, not sequence. Within a statement the unit is the
**command** — a bag of verbs folded into one filter predicate and one
combined populate ([The Command Algebra](/docs/design/command-algebra))
— and the statement's contribution is the union over its commands.

**What one command selects** is the rows matching its own filters that
*also* have evidence with its parent's and its children's selections.
Each side of that condition constrains the other, so the answer is not
read off in one pass: it is a fixpoint, reached by a worklist that
narrows every command until nothing changes
([Execution Engine](/docs/design/execution-engine)).

**A fixpoint resists caching.** A command's selection is a joint
function of everything visible — change any neighbour and it may move
— so nothing smaller than the whole query names it, and any cache
entry keyed on less would be a lie.

**So cache an upper bound instead.** Each command *reads* by a
predicate assembled from its own filters plus the constraints its
neighbours impose: a superset of the final selection, which the
worklist then narrows. That predicate changes only when the command or
its neighbourhood changes, so editing one part of a query leaves the
other parts' reads byte-identical — and identical reads are hits in
the in-RAM cache, which keys on the SQL and its binds. Making the
bound *tight* rather than merely correct is the planner's job:
capped probes measure cardinality before anything is read in full
([Cost-Based Execution](/docs/design/cost-based-execution)).

**Reads are evaluated against visible layers**, and visibility grows as
the query runs: before its reads, each statement **materialises** the
rows its content verbs produce into new layers, which the following
statements can see. The visible set is therefore part of every read's
key. It does not make the cache lie when a layer appears — it makes it
*fragment*: each distinct visibility gets its own entry, and the work
behind it is done again.

**What a materialisation contains** is where the second cache lives:
rows produced from layer content, plus rows built from earlier
statements' selections. Caching that is a different proposition from
caching a read — it saves *producing* rows, once, across queries and
processes, where the in-RAM cache saves *re-reading* rows already
produced within one process ([Caching](/docs/design/caching)).

**But production cannot be cached whole.** Key it on everything it
read — the entire visible slice — and one added ephemeral layer
changes the key, so a full-text scan over a kernel-sized corpus is
redone on account of a layer that contributed nothing to it.

**Hence the split.** A populate that satisfies a decomposability axiom
can be carved along the layers it reads, so the expensive and stable
part — the scan of the committed corpus — is keyed apart from the
cheap, volatile ones and survives their churn. Which node holds what,
what its name guarantees, and why that is the whole caching story is
[Partitioning a Materialisation](/docs/design/shards).

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

## Where to start

- **To understand what a query *means***, read
  [Execution Engine](/docs/design/execution-engine) for composition and
  weakness, then [Cost-Based Execution](/docs/design/cost-based-execution)
  for how the same meaning is computed cheaply.
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
