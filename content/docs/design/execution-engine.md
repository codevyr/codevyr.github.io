---
title: "Execution Engine"
description: "How the askl query engine evaluates queries using a monotone worklist algorithm"
weight: 100
---

The askl execution engine is a **monotone worklist propagation system**. Each substatement of a
query holds a *selection* — a set of symbols — and the engine iterates, narrowing selections and
propagating constraints between substatements until nothing more changes.

This document explains the algorithm, its data model, and why it terminates correctly.

## 1. Substatements and selections {#substatements-and-selections}

A query is a forest of *substatements* (see the [terminology](/docs/design/overview/#terminology)), each with:

- A **command** — one or more verbs (selectors) that describe what symbols to find
- A **scope** — child substatements that constrain or extend the result
- A **selection** — the current symbol set, held in the registry and updated in-place

```askl
func("processRequest") {   /* substatement A: functions named processRequest */
    "validate"             /* substatement B: symbols named validate, called by A */
}
```

Substatements are related by their nesting: A is the parent of B. The engine propagates selections
down (parent tells child which symbols to consider) and up (child constrains the parent to only
those that have matching children).

## 2. Dependency kinds {#dependency-kinds}

Each substatement has a list of *dependencies* — other substatements whose selections must exist before
it can produce useful output. There are two kinds:

| Kind | Meaning |
|------|---------|
| `Necessary` | This dep must have a selection before the substatement can produce any output |
| `Sufficient` | This dep can provide initial output; additional sufficient deps constrain further |

**Sufficient** is the common case. A child substatement depends on its parent to derive its initial
selection (e.g., look up `"validate"` only among symbols called by the selected functions). Any
one satisfied sufficient dep enables the first output; the rest narrow it.

**Necessary** applies when a substatement literally cannot compute without another substatement's result:

- `use("label")` — waits for the substatement with `label("label")` to resolve. There is nothing
  to look up until the provider has a selection.
- `forced` — waits for its parent to propagate. It derives its selection directly from the
  parent's result and has nothing to compute on its own.

## 3. Data model {#data-model}

```
StatementDependency {
    dependency:      Rc<Statement>   // the dep statement
    dependency_role: DependencyRole  // Parent | Child | User
    kind:            DependencyKind  // Necessary | Sufficient
}

ExecutionState {
    dependencies: Vec<StatementDependency>  // what this statement waits for
    dependents:   Vec<StatementDependent>   // who to notify when this statement changes
    weak:         bool                      // does not constrain its dependencies
    warnings:     Vec<Error>
}
```

The `dependents` list is the notification graph: when substatement X's selection changes, every
entry in `X.dependents` is notified so it can re-evaluate.

## 4. Background: the worklist algorithm {#worklist-algorithm}

That model is an instance of a standard technique. A
[worklist algorithm](https://en.wikipedia.org/wiki/Data-flow_analysis) comes
from dataflow analysis (Kildall 1973), and its key ingredients are:

- A **lattice** of values that can only move in one direction (here: selections — instance sets, closed per symbol — that only shrink)
- A **monotone transfer function** per node (here: constrain a substatement's selection based on its neighbours)
- A **worklist** of nodes whose outputs have changed and whose successors need re-evaluation

Because the lattice has finite height and the transfer functions are monotone (selections never grow),
the algorithm is guaranteed to terminate: each reschedule removes at least one symbol from some set,
and there are only finitely many symbols.

The closest analogy in practice is a **constraint propagation network** (as in Gecode or MiniCP):
each substatement is a constraint variable, each scope nesting is a constraint, and the engine propagates
until all constraints are simultaneously satisfied.

## 5. The pipeline {#pipeline}

`compute_nodes` runs: `build_dependency_graph` → the anchor-completeness
check → `mark_weak_statements` → `compute_roots` (which itself runs three
phases per top-level statement, statements in source order: **Phase M**
materialise layers, **Phase P** probe, **Phase R** read) →
`run_worklist`. Each statement fully finishes M, P, R before the next
starts — the statement separator (`;` or a newline) is the only time
axis a query has. The sections below cover the
graph build, the initial selections, and the worklist; probing is
formalised on the
[Cost-Based Execution](/docs/design/cost-based-execution) page.

### 5.1. Initial selections (`compute_roots`) {#initial-selections}

Phase M materialises any ephemeral layers — **one materialisation per
layer-creating statement**. A visibility snapshot is taken once, before
any of the statement's commands materialise, and every `@label` argument is
resolved up front from the completed selection of the *earlier*
statement that defines it (the parse-time ordering rule rejects
same-statement and forward references, so no mid-phase read is ever
needed). The statement's layer-bearing commands then materialise
**concurrently** against that same pre-statement snapshot: no command
sees a statement-mate's layers at materialise time — nesting does not
sequence — so the calls are independent by construction. Outcomes are
applied in pre-order (completion order is unobservable), the
statement's layers enter visibility as one atomic materialisation pushed after
the batch, and each command keeps its own layers — attribution by
layer id is what gives every command its own selection.

Phase P then probes every
eligible substatement (anchored, with a fusable single-query predicate):
a probe fetches up to `--probe-cap`+1 instance ids for the full predicate;
at or under the cap the substatement is **resolved** — the id set is its
exact denotation, its Phase-R read becomes a primary-key fetch, and the
ids feed neighbouring scopes in place of predicates the index would have
to materialise. Capped and derive-only substatements re-probe in
refinement waves under semi-join roles from resolved neighbours, to a
fixpoint. Phase R then computes the initial selections concurrently,
under the statement-complete visibility — a nested layer creator's
rows are visible to its ancestors' neighbourhood reads and vice versa,
which is what makes `search("a") { search("b") }` compose:

- `"processRequest"` → resolved by its probe (or predicate query) → initial selection
- `func`, bare type selectors, `project(...)` → pure constraints: no
  query of their own; they derive from neighbours during propagation
- `use("label")`, `forced` → produce no initial selection (they wait for propagation)

The worklist below is unchanged by probing — probes only ever hand it
smaller, exact inputs.

### 5.2. `build_dependency_graph` {#build-dependency-graph}

Wires up the `dependencies` / `dependents` edges based on query structure:

| Relationship | dep added to | role | kind |
|---|---|---|---|
| Substatement → its parent | `child.dependencies` | `Child` | `Necessary` if `forced`, else `Sufficient` |
| Substatement → label provider | `statement.dependencies` | `User` | `Necessary` |
| Parent → each child (notify) | `parent.dependents` | `Child` | — |
| Child → its parent (notify) | `child.dependents` | `Parent` | — |
| Provider → user (notify) | `provider.dependents` | `User` | — |

Note that the **parent does not hold a dependency on its children**. The parent is notified
*by* its children when they have selections, but the parent's own readiness is independent:
it can start propagating as soon as it has its own initial selection.

### 5.3. `run_worklist` {#run-worklist}

The main propagation loop:

```
seed worklist: every statement that has_selection() after compute_roots

while worklist is not empty:
    stmt = pop statement with smallest selection (fewest symbols first)
    apply update_state + filter to stmt's selection
    if stmt has no selection: skip (nothing to propagate)
    for each dependent of stmt:
        result = stmt.notify(dependent)
        if result.changed:
            worklist.schedule(dependent)
```

**Seeding.** Only substatements with an initial selection enter the worklist. Substatements like
`use("label")` or `forced` produce no selection in `compute_roots`, so they start outside
the worklist. They enter only when notified by the substatements they depend on.

**Priority.** Substatements are popped in order of selection size (fewest symbols first). This
is a heuristic: processing the most-constrained substatements first tends to propagate tight
constraints early, reducing wasted work downstream.

**Notification.** When substatement A is popped and has a selection, it notifies each entry in
`A.dependents`:

- **Child role** (parent notifying child): child derives its selection from the parent's
  current selection. Child enters the worklist if its selection changed.
- **Parent role** (child notifying parent): the engine defers until *all* children of the
  parent have selections, then merges their selections (union) and constrains the parent.
  This ensures a parent like `func { "a" ; "b" }` retains functions that call *either*
  `"a"` *or* `"b"`, not only those that call both.
- **User role** (provider notifying user): the user substatement derives from the provider's
  selection, as if the provider's symbols were its scope.

**`PropagationResult`**. `notify()` returns `PropagationResult { changed: bool }` — a clean
signal indicating whether the dependent's selection actually changed. Only changed dependents
are rescheduled, avoiding unnecessary work.

## 6. Convergence {#convergence}

The algorithm terminates because:

1. **Selections only shrink.** Derivation and constraining are monotone: they can only remove
   symbols, never add them. The lattice height is bounded by the total number of symbols.

2. **Each reschedule reflects a real change.** The worklist only adds a substatement when
   `result.changed` is true — meaning at least one symbol was removed from its selection.

3. **Unresolvable substatements never enter.** A substatement like `use("missing_label")` — where
   the label does not exist — is caught at `build_dependency_graph` time and returns an
   error immediately. A substatement whose provider has no selection simply never gets notified
   and never enters the worklist. The loop terminates naturally; it does not need to detect
   or special-case unresolvable substatements.

## 7. Example: label resolution {#label-resolution-example}

```askl
label("handlers") func("Handle") {  /* A: functions named Handle, labeled "handlers" */
    "respond"                        /* B: symbols called by Handle */
}
use("handlers") macro {              /* C: macros that reference any "handlers" function */
    func                             /* D: functions called by those macros */
}
```

Execution trace:

| Phase | Event |
|---|---|
| `compute_roots` | A gets selection: `[Handle_1, Handle_2]`. B, C, D get no selection. |
| `build_dependency_graph` | C.dependencies = `[(A, User, Necessary)]`. D.dependencies = `[(C, Child, Sufficient)]`. |
| `run_worklist` seed | A enters worklist (has selection). B does not (no selection yet). |
| Pop A | A notifies B (Child): B derives `[respond_1, respond_2]`. B enters worklist. |
| | A notifies C (User, because A is labeled "handlers"): C derives `[macro_X]`. C enters worklist. |
| Pop C (smaller) | C notifies D (Child): D derives `[init, run]`. D enters worklist. |
| Pop B | B notifies A (Parent): A is constrained to handles that actually call `respond`. |
| Pop D | D notifies C (Parent): C is constrained to macros that call the selected functions. |
| … | Continue until no selection changes. |

Each iteration removes symbols or adds nothing. The loop exits when the worklist is empty.

## 8. Weakness and bindness {#weakness-and-bindness}

Two related but distinct properties govern how substatements participate in a
query, at two different levels.

### 8.1. Weakness (command-level, compositional) {#weakness}

**Weakness answers: does this command's selection constrain its neighbours?**
A weak substatement is a display echo — it contributes nodes and edges to the
result graph, but an empty match does not eliminate its parent or children.
A strong substatement's selection participates in the composition: a parent
survives only if it actually relates to something the strong child selected.

A substatement is a *weakness candidate* when its command is **non-constraining**:
every selector is a unit verb or a bare (nameless) type selector. Filters
alone do not make a command constraining — `filter("compound_name", "x")` on
an otherwise bare substatement leaves it a candidate.

`mark_weak_statements` (between `build_dependency_graph` and `run_worklist`)
then iterates the **propagation rule** to a fixpoint. A candidate becomes
weak iff:

- its parent is weak (or it is top-level), **or**
- all of its children are weak (vacuously true for a leaf).

The consequence worth internalising: a candidate **sandwiched between a
strong parent and a strong child stays strong**. In

```askl
select filter("compound_name", "test", inherit="true") {{ "b" }}
```

the outer command is strong (`select`), the leaf `"b"` is strong, so the bare
middle scope — a candidate — satisfies neither weakening condition and
constrains: only callers of `test.b` that the outer level also relates to
survive. Drop the `select` and the outer command becomes a top-level
candidate, turns weak, weakness flows through the middle, and the same query
becomes an echo that shows *every* namespace-matching caller.

### 8.2. Bindness (component-level, outcome) {#bindness}

**Bindness answers: does this component demand instances at all?**
(A *component*: one or more statements connected by label
references — see the [terminology](/docs/design/overview/#terminology).) A binding
component wants results; a non-binding one is structure or directive.
`preamble project("ucx")` is non-binding — it configures the query. `{{}}`
is non-binding structure. Non-binding components are silently empty;
binding ones must be *satisfiable*: each needs at least one **anchor**
(a name, `search(...)`, `loc(...)`, a layer literal, or `select`), otherwise
the query is rejected with a hint.

### 8.3. `select` bridges the two levels {#select-bridges}

`select` is the user-visible verb for both properties, named for the outcome:

- at the **component** level it declares the component binding — and since it carries
  an always-true anchor, a `select`-bearing component is always satisfiable
  (an unanchored command with `select` enumerates everything its filters
  allow, bounded by the result budget);
- at its **command** it implies strength — wanting instances *here* means
  this command's selection participates in the composition, so a
  `select`-carrying command is never a weakness candidate.

Weakness otherwise keeps its defaults: anchored commands are constraining
by construction, and the propagation rule above decides the rest.

## 9. Prior art {#prior-art}

The fixpoint itself is standard. Iterating a dependency graph until
nothing changes, propagating only what changed, has the same shape as
**semi-naive evaluation** in Datalog — with the direction reversed:
Datalog's relations grow towards a least fixpoint, while selections here
only shrink, and the delta a round carries is a removal rather than an
addition. It also resembles **chaotic iteration** in abstract
interpretation, which likewise leaves the visit order free and leans on
monotonicity to make the result order-independent; §4's dataflow framing
and the constraint-propagation analogy are the same idea in two other
vocabularies. What is particular here is what sits in the lattice:
selections are instance sets closed per symbol, so a round's transfer
function is a per-symbol semi-join against a neighbour rather than a rule
body, and weakness lets a node participate in the result without
constraining anyone.
