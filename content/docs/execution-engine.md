---
title: "Execution Engine"
description: "How the askl query engine evaluates queries using a monotone worklist algorithm"
weight: 300
---

The askl execution engine is a **monotone worklist propagation system**. Each query statement
holds a *selection* — a set of symbols — and the engine iterates, narrowing selections and
propagating constraints between statements until nothing more changes.

This document explains the algorithm, its data model, and why it terminates correctly.

## Background: The Worklist Algorithm

A [worklist algorithm](https://en.wikipedia.org/wiki/Data-flow_analysis) is a standard technique
from dataflow analysis (Kildall 1973). The key ingredients are:

- A **lattice** of values that can only move in one direction (here: symbol sets that only shrink)
- A **monotone transfer function** per node (here: constrain a statement's selection based on its neighbours)
- A **worklist** of nodes whose outputs have changed and whose successors need re-evaluation

Because the lattice has finite height and the transfer functions are monotone (selections never grow),
the algorithm is guaranteed to terminate: each reschedule removes at least one symbol from some set,
and there are only finitely many symbols.

The closest analogy in practice is a **constraint propagation network** (as in Gecode or MiniCP):
each statement is a constraint variable, each scope nesting is a constraint, and the engine propagates
until all constraints are simultaneously satisfied.

## Statements and Selections

A query is a tree of *statements*, each with:

- A **command** — one or more verbs (selectors) that describe what symbols to find
- A **scope** — child statements that constrain or extend the result
- A **selection** — the current symbol set, held in the registry and updated in-place

```askl
func("processRequest") {   /* statement A: functions named processRequest */
    "validate"             /* statement B: symbols named validate, called by A */
}
```

Statements are related by their nesting: A is the parent of B. The engine propagates selections
down (parent tells child which symbols to consider) and up (child constrains the parent to only
those that have matching children).

## Dependency Kinds

Each statement has a list of *dependencies* — other statements whose selections must exist before
it can produce useful output. There are two kinds:

| Kind | Meaning |
|------|---------|
| `Necessary` | This dep must have a selection before the statement can produce any output |
| `Sufficient` | This dep can provide initial output; additional sufficient deps constrain further |

**Sufficient** is the common case. A child statement depends on its parent to derive its initial
selection (e.g., look up `"validate"` only among symbols called by the selected functions). Any
one satisfied sufficient dep enables the first output; the rest narrow it.

**Necessary** applies when a statement literally cannot compute without another statement's result:

- `use("label")` — waits for the statement with `label("label")` to resolve. There is nothing
  to look up until the provider has a selection.
- `forced` — waits for its parent to propagate. It derives its selection directly from the
  parent's result and has nothing to compute on its own.

## Data Model

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

The `dependents` list is the notification graph: when statement X's selection changes, every
entry in `X.dependents` is notified so it can re-evaluate.

## The Three Phases

The engine runs three phases in sequence inside `compute_nodes`:

### 1. `initialize_roots`

Runs all selectors concurrently against the database. Each statement executes its command
independently — no parent/child relationship is considered yet.

- `"processRequest"` → queries DB for functions with that name → initial selection
- `func`, `type`, bare type selectors → queries DB → initial selection
- `use("label")`, `forced` → produce no initial selection (they wait for propagation)

After this phase, root statements have selections; dependency-only statements do not.

### 2. `build_dependency_graph`

Wires up the `dependencies` / `dependents` edges based on query structure:

| Relationship | dep added to | role | kind |
|---|---|---|---|
| Statement → its parent | `child.dependencies` | `Child` | `Necessary` if `forced`, else `Sufficient` |
| Statement → label provider | `statement.dependencies` | `User` | `Necessary` |
| Parent → each child (notify) | `parent.dependents` | `Child` | — |
| Child → its parent (notify) | `child.dependents` | `Parent` | — |
| Provider → user (notify) | `provider.dependents` | `User` | — |

Note that the **parent does not hold a dependency on its children**. The parent is notified
*by* its children when they have selections, but the parent's own readiness is independent:
it can start propagating as soon as it has its own initial selection.

### 3. `run_worklist`

The main propagation loop:

```
seed worklist: every statement that has_selection() after initialize_roots

while worklist is not empty:
    stmt = pop statement with smallest selection (fewest symbols first)
    apply update_state + filter to stmt's selection
    if stmt has no selection: skip (nothing to propagate)
    for each dependent of stmt:
        result = stmt.notify(dependent)
        if result.changed:
            worklist.schedule(dependent)
```

**Seeding.** Only statements with an initial selection enter the worklist. Statements like
`use("label")` or `forced` produce no selection in `initialize_roots`, so they start outside
the worklist. They enter only when notified by the statements they depend on.

**Priority.** Statements are popped in order of selection size (fewest symbols first). This
is a heuristic: processing the most-constrained statements first tends to propagate tight
constraints early, reducing wasted work downstream.

**Notification.** When statement A is popped and has a selection, it notifies each entry in
`A.dependents`:

- **Child role** (parent notifying child): child derives its selection from the parent's
  current selection. Child enters the worklist if its selection changed.
- **Parent role** (child notifying parent): the engine defers until *all* children of the
  parent have selections, then merges their selections (union) and constrains the parent.
  This ensures a parent like `func { "a" ; "b" }` retains functions that call *either*
  `"a"` *or* `"b"`, not only those that call both.
- **User role** (provider notifying user): user statement derives from the provider's
  selection, as if the provider's symbols were its scope.

**`PropagationResult`**. `notify()` returns `PropagationResult { changed: bool }` — a clean
signal indicating whether the dependent's selection actually changed. Only changed dependents
are rescheduled, avoiding unnecessary work.

## Convergence

The algorithm terminates because:

1. **Selections only shrink.** Derivation and constraining are monotone: they can only remove
   symbols, never add them. The lattice height is bounded by the total number of symbols.

2. **Each reschedule reflects a real change.** The worklist only adds a statement when
   `result.changed` is true — meaning at least one symbol was removed from its selection.

3. **Unresolvable statements never enter.** A statement like `use("missing_label")` — where
   the label does not exist — is caught at `build_dependency_graph` time and returns an
   error immediately. A statement whose provider has no selection simply never gets notified
   and never enters the worklist. The loop terminates naturally; it does not need to detect
   or special-case unresolvable statements.

## Example: Label Resolution

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
| `initialize_roots` | A gets selection: `[Handle_1, Handle_2]`. B, C, D get no selection. |
| `build_dependency_graph` | C.dependencies = `[(A, User, Necessary)]`. D.dependencies = `[(C, Child, Sufficient)]`. |
| `run_worklist` seed | A enters worklist (has selection). B does not (no selection yet). |
| Pop A | A notifies B (Child): B derives `[respond_1, respond_2]`. B enters worklist. |
| | A notifies C (User, because A is labeled "handlers"): C derives `[macro_X]`. C enters worklist. |
| Pop C (smaller) | C notifies D (Child): D derives `[init, run]`. D enters worklist. |
| Pop B | B notifies A (Parent): A is constrained to handles that actually call `respond`. |
| Pop D | D notifies C (Parent): C is constrained to macros that call the selected functions. |
| … | Continue until no selection changes. |

Each iteration removes symbols or adds nothing. The loop exits when the worklist is empty.

## Weak Statements

A statement is **weak** if it contains only non-constraining verbs and either has no parent,
no children, or all its children are also weak. Weak statements do not constrain the
selections of their dependencies — they contribute nodes to the result graph but do not
filter what their parents or children show.

The `mark_weak_statements` phase (between `build_dependency_graph` and `run_worklist`)
computes the weak flag for all statements before propagation begins.
