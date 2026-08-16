---
title: "Evaluating the Fixpoint"
description: "How the engine computes the fixpoint of mutually constraining selections — the pipeline, the monotone worklist, and why it terminates"
weight: 110
aliases:
  - /docs/design/execution-engine/
---

[Queries and their Meaning](/docs/design/semantics/#selections) leaves a
query's answer defined but not computed. Each command's selection is
given in terms of its neighbours' selections, which are given in terms
of theirs; the answer is the fixpoint of that mutual recursion. A
fixpoint is a specification — it says which assignment is the answer
without saying how to find it, or whether looking terminates.

This page is the procedure. The engine reaches the fixpoint by
**monotone worklist propagation**: every substatement starts at an
over-approximation of its selection, the engine narrows one
substatement at a time against its neighbours, and re-examines only
what a narrowing could have disturbed. Selections never grow — the one
property that makes the loop finite, and makes its result independent
of the order the work happened in.

## 1. The data model {#data-model}

Each substatement carries an execution state with four parts: the
**dependencies** it waits for — other substatements, each edge tagged
`Necessary` or `Sufficient`
([semantics §8](/docs/design/semantics/#dependency-kinds)) — the
**dependents** to notify when its own selection changes, whether it is
weak ([semantics §9](/docs/design/semantics/#weakness)), and any
warnings raised while it ran.

The dependents list is the **notification graph**, and it is the only
structure the loop below walks: when a substatement's selection
changes, each of its dependents is scheduled to re-evaluate. The two
lists are deliberately not mirror images of each other. A parent is
notified *by* its children but holds no dependency *on* them — its own
readiness is independent, so it can start propagating as soon as it has
an initial selection, rather than waiting for a scope that may be
waiting on it. `build_dependency_graph` wires both lists once, from the
query's nesting and its label references.

## 2. The pipeline {#pipeline}

`compute_nodes` runs five steps: build the dependency graph, check
anchor completeness, mark weak substatements, compute the initial
selections, and run the worklist. The fourth step is itself three
phases per top-level statement, statements in source order —
**materialise**, **probe**, **read** — and a statement finishes all
three before the next one starts, the statement separator being the
only time axis a query has.

The first two phases belong to other pages. **Materialise** runs the
statement's content verbs against the pre-statement visibility
snapshot and appends the resulting layers as one atomic materialisation
([Layers §5](/docs/design/layers/#materialisations)). **Probe** then
measures before anything is read in full, so that a selective leaf can
drive the rest of the query
([Cost-Based Execution](/docs/design/cost-based-execution)). Neither
changes the loop below: probing only ever hands it smaller, exact
inputs.

### 2.1. Initial selections {#initial-selections}

**Read** computes the initial selections concurrently, under the
statement-complete visibility — a nested layer creator's rows are
visible to its ancestors' neighbourhood reads and vice versa, which is
what makes `search("a") { search("b") }` compose:

- `"processRequest"` → resolved by its probe (or predicate query) → initial selection
- `func`, bare type selectors, `project(...)` → pure constraints: no
  query of their own; they derive from neighbours during propagation
- `use("label")`, `forced` → produce no initial selection (they wait for propagation)

Each initial selection is a superset of the command's final selection,
because a read applies the command's own predicate and whatever
neighbour conditions were available, but not the constraints the
fixpoint has yet to derive. That is the over-approximation the worklist
starts from.

### 2.2. `run_worklist` {#run-worklist}

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
constraints early, reducing wasted work downstream. It is free to be a heuristic — §5's
order-independence means a different order would reach the same fixpoint, only slower.

**Notification.** When substatement A is popped and has a selection, it notifies each entry in
`A.dependents`, in the role that edge was wired with:

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

## 3. Convergence {#convergence}

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

## 4. Example: label resolution {#label-resolution-example}

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

## 5. Where this sits {#prior-art}

The loop is standard, and it has four names in four literatures. Each of
them names something about it well.

As a **worklist algorithm** from dataflow analysis (Kildall 1973) it is
textbook, and the framing supplies §3's argument in the abstract: a
lattice of values that move in one direction only (here selections,
which only shrink), a monotone transfer function per node (constrain a
substatement against its neighbours), and a worklist of the nodes whose
successors need re-examining. Finite height plus monotonicity gives
termination.

As a **constraint propagation network** (Gecode, MiniCP) it is the
closest operational analogy: every substatement is a constraint
variable, every scope nesting a constraint, and propagation runs until
all constraints hold at once.

As **semi-naive evaluation** in Datalog it is the same shape with the
direction reversed — Datalog's relations grow towards a least fixpoint,
while selections here only shrink, so the delta a round carries is a
removal rather than an addition. **Chaotic iteration** in abstract
interpretation contributes the property §2.2 relies on: the visit order
is free, and monotonicity is what makes the answer independent of it.

What is particular here is what sits in the lattice. Selections are
instance sets closed per symbol, so a round's transfer function is a
per-symbol semi-join against a neighbour rather than a rule body — and
weakness ([semantics §9](/docs/design/semantics/#weakness)) lets a node
participate in the result without constraining anyone, which none of
the four vocabularies has a word for.

## Where to read more

- [Queries and their Meaning](/docs/design/semantics) — what the
  fixpoint computed here means, and where the initial selections'
  predicate comes from.
- [Cost-Based Execution](/docs/design/cost-based-execution) — the probe
  phase, and how it shrinks the loop's inputs.
- [Layers and layer operations](/docs/design/layers) — the materialise
  phase, and the visibility each read runs under.
