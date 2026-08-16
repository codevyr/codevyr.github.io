---
title: "Overview"
description: "What askl does, what a query returns, the three questions its design has to answer, and how to read the rest of the chapter"
weight: 90
---
askl answers questions about code: which functions call this one, where
this string occurs, what a module contains. A query names patterns and
nests them; the engine resolves the names against an index of one or
more uploaded projects and returns a **graph** — the symbols that
matched, and the reference and containment edges among them.

These pages are long-form technical documents about *how* that happens
— algorithms, data models, correctness invariants, and the trade-offs
that shaped them. They are aimed at contributors, and at users who want
a full picture of what happens when they run a query; the
[Askl Syntax Reference](/docs/syntax) is the right first stop for using
the language.

## The life of a query {#the-life-of-a-query}

Before any of the design questions, it helps to know what actually
runs. A query is a sequence of **statements**, and statements are the
only time axis the language has: they run in source order, one after
another, and nesting inside a statement expresses constraint rather
than sequence.

**Parsing** turns the text into a forest of substatements. Each one
carries a **command** — its bag of verbs, folded in source order into a
single filter predicate and a single combined populate
([Queries and their Meaning](/docs/design/semantics)). Two structural
rules are checked before anything runs: every binding component must
contain an anchor, so that a query cannot ask for "everything" by
accident, and a `@label` may only reference an earlier statement.

Each statement then runs three phases before the next one starts
([Evaluating the Fixpoint](/docs/design/evaluation)):

1. **Materialise.** Commands that produce content — `search`, `loc`,
   `layer { … }` — write their rows into new layers, which become
   visible to every later statement. Those layers are the statement's
   *materialisation*, and each is stored and named separately
   ([Partitioning a Materialisation](/docs/design/shards) has the split,
   [Layer Keys and Hashing](/docs/design/layer-keys) the names).
2. **Probe.** Before reading anything in full, the engine measures.
   Capped id probes find which substatements are small enough to
   enumerate exactly, and refinement waves let one selective leaf drive
   the rest of the query
   ([Planning from Measured Cardinality](/docs/design/planning)) — this is
   what keeps a query like `mod("amdgpu") { func { "drm_dev_enter" } }`
   from resolving millions of rows it will immediately discard.
3. **Read.** Each command reads its rows under the layers now visible,
   conjoining its filter predicate.

**Composition** finishes the job. A monotone worklist propagates
constraints between neighbouring substatements — a parent narrows its
children, children narrow their parent — until nothing changes; the
instances still standing, and the edges among them, are emitted as the
result graph ([Evaluating the Fixpoint](/docs/design/evaluation)).

## Three questions {#three-questions}

Every part of that pipeline is there for a reason, and the reasons are
answers to three questions that a graph-returning query language cannot
avoid. Each one creates the next.

1. **What does a query mean?** Nesting is constraint, not sequence, so
   neighbouring parts of a query constrain each other and no part can
   be evaluated on its own. The answer is a fixpoint
   ([Queries and their Meaning](/docs/design/semantics)).
2. **How is it evaluated without being slow?** The fixpoint has to be
   computed over an index of millions of rows, and syntax cannot
   predict how many rows a pattern will match — so the engine measures
   before it reads ([Evaluating the Fixpoint](/docs/design/evaluation),
   [Planning from Measured Cardinality](/docs/design/planning)).
3. **How is work reused across queries?** An interactive session reruns
   near-identical queries, so the expensive parts must survive from one
   to the next. Reads are one kind of work and the rows a command
   *produces* are another, and the second cannot be reused until it has
   been partitioned along what it reads. That partition is the
   chapter's main contribution
   ([Partitioning a Materialisation](/docs/design/shards)).

[From Result to Cache](/docs/design/derivation) is the single argument
that runs through all three, from what a query returns down to why its
results are stored the way they are; the pages after it develop one
part each.

## Terminology {#terminology}

The pages share a vocabulary and a notation, both used with fixed
meanings. This section is the reference for both; each term names the
page that owns it.

**Query structure.**

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
- **filter** — a constraint on rows that a command carries alongside its
  content verbs: `project("linux")`, a type constraint, `ignore(...)`.
  Most filters are contributed by verbs; a bare name string contributes
  one without being a verb. A command's filters are combined into a
  single predicate \(F_c\), a tree of leaves under And/Or/Not nodes, and
  "the command's filter" means that composite
  ([Layer Keys §1](/docs/design/layer-keys/#filter-predicate)).
- **component** — one or more statements connected by label references;
  the unit of the anchor rule and of bindness.
- **anchor** — a verb that can produce instances on its own: a name
  pattern, a name filter, `search(...)`, `loc(...)`, a layer literal, or
  `select`. Everything else only narrows what an anchor produced, which
  is why a binding component without one is rejected rather than
  answered ([semantics §9.2](/docs/design/semantics/#bindness)).
- **weakness** — whether a command's selection constrains its
  neighbours. A weak substatement is a display echo: it contributes
  nodes and edges, but an empty match does not eliminate its parent or
  children ([semantics §9.1](/docs/design/semantics/#weakness)).
- **bindness** — whether a *component* demands instances at all. A
  binding component wants results and must be satisfiable; a
  non-binding one is structure or directive, and is silently empty
  ([semantics §9.2](/docs/design/semantics/#bindness)).

**What a query means.**

- **denotation** — \(D(c)\), what command \(c\)'s own predicate
  \(P(c)\) — its filters and its selector branches — picks out of the
  visible instances. It is computable from the command alone, which is
  what distinguishes it from a selection
  ([semantics §6](/docs/design/semantics/#denotation)).
- **selection** — \(N_c\), the instances a command holds once the
  worklist fixpoint has run, closed per symbol; \(N_s\) for a whole
  statement ([semantics §7](/docs/design/semantics/#selections)). The
  referenced outputs \(O_c\) of a command are earlier statements'
  selections. Distinct from the selection *function* \(\sigma_{F_c}\),
  which filters a row set.
- **populate** — the function a content verb contributes: given a slice
  of layer content, it returns the rows that verb writes for just that
  slice ([semantics §1](/docs/design/semantics/#what-a-verb-denotes)).
  The engine fills a layer's rows by running populates; \(U_c\) is a
  command's populates unioned.
- **wave** — one probe iteration of the cost-based executor (wave 0 plus
  the refinement waves), distinct from a materialisation.

**Storage and reuse.**

- **layer** — a labelled set of graph rows: one `index.layers` row plus
  every object, symbol, instance, and reference tagged with its id.
  Every row belongs to exactly one layer, and a query sees a flat set of
  visible layer ids rather than any structure over them
  ([Layers §2](/docs/design/layers/#what-a-layer-is)).
- **root layer** — \(R\), a project's initial persistent layer, carrying
  a stored identity hash \(h(R)\) that names the committed index state
  it stands for. With any later persistent delta layers it forms the
  project's *persistent closure*
  ([Layers §3](/docs/design/layers/#kinds-and-lifetimes)).
- **ephemeral layer** — a layer materialised by a verb during a query,
  holding that command's output. It lives in the same tables as the
  persistent index and is read by the same SQL, but its lifetime is that
  of a cache entry — TTL and LRU, not permanence
  ([Layers §3](/docs/design/layers/#kinds-and-lifetimes),
  [Caching](/docs/design/caching)).
- **materialisation** — the set of layers one layer-creating statement
  produces on one project, appended to visibility atomically
  (`push_materialisation`). Statements and their materialisations
  correspond one-to-one, so \(t\) indexes both.
- **shard** — a node holding one dependency's part of a command's
  materialisation, stored and keyed on its own. There are three kinds,
  derived in
  [Partitioning a Materialisation §3](/docs/design/shards/#node-kinds);
  they are the engine's `ShardRole::Root`, `Layer`, and `Selection`.
- **root shard** — \(\mathrm{Sh}_c(R)\), the part of the materialisation
  the command's populate produces from the root layer's content: the
  expensive scan of the committed corpus. Its key names \(h(R)\) and
  \(H(c)\) and nothing about the query's ephemeral context, so it is
  shared by every query that runs the same command against the same
  corpus.
- **layer shard** — \(\mathrm{Sh}_c(\ell)\), the same populate over one
  *content-bearing light layer* — a visible non-root layer whose rows a
  populate reads, one of the set \(E_t(R)\) — keyed by that layer's id.
  Root and layer shards together are the **input shards**: the shards
  whose parent is the very input they read.
- **selection shard** — \(S_c\), the per-command node parented on the
  previous statement's tip; it holds everything the command builds from
  earlier statements' selections, and marks the chain position when
  empty. A statement's selection-shard-bearing commands contribute
  *sibling* selection shards.
- **tip** — \(\mathrm{tip}_t(R)\), the last layer of statement \(t\)'s
  materialisation in command pre-order: the layer the next statement's
  selection shards hang off ([Layers §5](/docs/design/layers/#materialisations)).
- **spine** — the chain of tips, one per statement — the only lineage in
  the layer forest that encodes query history
  ([shards §5](/docs/design/shards/#worked-example)).
- **command hash** — \(H(c)\), a canonical hash summarising a command,
  so that equal hashes mean semantically identical commands. It
  names every input the command's populates read *except* the layer
  content they are aimed at, which the node keys name instead
  ([Layer Keys §4](/docs/design/layer-keys/#command-hash)).

## The pages, in reading order {#the-pages}

The chapter is written to be read in order — each page's problem is
created by the one before it — but every page opens by stating that
problem, so it can also be entered directly. Read the first four for
what a query *means* and how it is evaluated; the next four for how
results are stored and reused, which is where the design is least
obvious; the last for one verb end to end, in SQL and in milliseconds.

**Evaluating a query**

- **[From Result to Cache](/docs/design/derivation)** — the chapter in
  one argument: why the engine has the parts it has, derived from what
  a query returns.
- **[Queries and their Meaning](/docs/design/semantics)** — how a bag of
  verbs folds into one predicate and one populate, and why a query's
  answer is a fixpoint of mutually constraining selections.
- **[Evaluating the Fixpoint](/docs/design/evaluation)** — the phases a
  statement runs, the monotone worklist that composes neighbouring
  substatements, and why it terminates.
- **[Planning from Measured Cardinality](/docs/design/planning)** —
  anchors, capped id probes, and the refinement waves that let one
  selective leaf drive a query.

**Storing and reusing results**

- **[Layers and layer operations](/docs/design/layers)** — why
  intermediate results are rows at all, how a query decides what it can
  see, and the isolation guarantee.
- **[Partitioning a Materialisation](/docs/design/shards)** — the
  contribution: production carved along its dependencies, so a volatile
  input cannot invalidate expensive work.
- **[Layer Keys and Hashing](/docs/design/layer-keys)** — one rule at
  byte level: name what a populate reads, and nothing more.
- **[Caching](/docs/design/caching)** — the two tiers, and the
  invariants that keep a cached answer from outliving the state it was
  computed from.

**One verb in depth**

- **[search()](/docs/design/search)** — turning a literal string into
  byte-range matches over the whole corpus, without regex and with
  byte-exact positions: the abstractions above cashed out against real
  SQL and real numbers. Its helper's full body and the measurements
  behind its claims are in
  [search(): Implementation Notes](/docs/design/search-appendix).
