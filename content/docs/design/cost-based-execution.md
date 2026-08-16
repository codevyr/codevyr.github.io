---
title: "Cost-Based Execution"
description: "A formal model of query planning from measured cardinality: denotations, anchors, capped probes, and semi-join refinement with its soundness argument"
weight: 110
---

The askl engine plans query execution from **measured cardinality**, not from
syntax. This page gives the model formally: substatements as predicate
denotations, anchoring as static eligibility, probes as exact bounded
evaluation, refinement as semi-join propagation — and the properties the
scheme rests on: probe *exactness*, refinement *soundness* against the
composition semantics, and *termination*. Implementation notes and measured
results follow.

## 1. Motivation

The engine previously carried a selector/filter split that was a
hand-written query plan: selectors initiated SQL, filters contributed WHERE
fragments, and the choice between them was made by the *user*, through
syntax (`func` vs `func("name")` and an explicit `filter=` toggle). The plan
was therefore fixed before any cardinality was known. For

```askl
mod("amdgpu") { func { "drm_dev_enter" } }
```

the engine could not see that the leaf denotes a handful of rows and
everything else should be derived from it; it resolved the bare `func`
condition into millions of ids, one SQL statement returned 2.67 M rows
(1.8 GB), and the query took 53.7 s. Under the model below the same query
returns the identical result from kilobyte-sized, index-driven SQL
statements.

## 2. Model

**Substatements and denotations.** A query is a forest of substatements.
Each substatement \(s\) carries one **predicate** \(P(s)\) — the conjunction
of its filters with the disjunction of its selector branches — and
denotes

$$D(s) \;=\; \{\, x \in \mathrm{Inst} \;:\; x \models P(s) \,\}\,,$$

the set of symbol instances satisfying the predicate. Each nested
substatement also carries a **relationship** to its parent,
\(\mathrm{rel}(s) \subseteq \{\mathrm{REFS}, \mathrm{HAS}\}\), naming which
edge kinds relate their instances (the `{ }` default is the union
\(\mathrm{REFS} \cup \mathrm{HAS}\): either kind counts).

**Composition.** Execution does not return \(D(s)\); it returns the **final
selection** \(o(s) \subseteq D(s)\) — the selection as a *set* of
instances (\(o\) is the docs' symbol for a statement's output; the
referenced outputs \(O_c\) of the layer pages are earlier statements'
\(o(s')\), and the
selection as a *function* is a different object, \(\sigma\)) — the
fixpoint of the worklist
propagation in which neighbouring substatements constrain each other through
**edge evidence**: for substatements \(s\) with neighbour \(n\),

$$o(s) \;=\; \{\, x \in D(s) \;:\; \exists\, y \in o(n).\; (x,y) \in E_{\mathrm{rel}} \,\}$$

(for constraining neighbours; weak neighbours impose no such condition —
see [Weakness and Bindness](/docs/design/execution-engine/#8-weakness-and-bindness)).
Here \(E_{\mathrm{rel}}\) is the evidence relation: a REFS edge or a
containment edge, matched **per symbol** — if any instance of a symbol is
evidenced, all of that symbol's instances survive.

**Monotonicity.** The worklist only ever narrows: at every intermediate
stage the current selection of any substatement is a superset of \(o(s)\).
This is the single fact every argument below leans on.

The planning problem: decide, per substatement, whether to materialise
\(D(s)\) independently, derive it from neighbours, and in what order —
using the index itself as the cardinality oracle.

## 3. Static eligibility: anchors

Materialising \(D(s)\) is only *meaningful* for some substatements. A
substatement is **anchored** iff its predicate contains at least one
**anchor**: a name pattern, a content predicate (`search`), a location
(`loc`), a layer literal, or the explicit `select` (an always-true anchor
with budget-bounded enumeration). Type predicates, `project(...)`, and
`preamble` are pure constraints: their denotations are enormous and never
worth materialising unconstrained; such substatements only ever derive.

The **completeness rule** is the static counterpart: every **component**
(one or more statements connected by label references — see the
[terminology](/docs/design/overview/#terminology)) must
contain an anchor, or the query is rejected with a hint — a component of
pure constraints denotes nothing materialisable and used to fail silently.

Anchoring answers *may this substatement drive?* Cost, measured next,
answers *should it, and when?*

## 4. Probes

**Definition.** For a predicate \(P\) and cap \(c\), a **probe** evaluates
\(P\) projected to instance ids with `LIMIT c+1`:

$$\mathrm{probe}(P, c) \;=\; \begin{cases} \mathrm{ids}(D(P)) & \text{if } |D(P)| \le c \\ \textbf{Capped} & \text{otherwise} \end{cases}$$

Two deliberate choices. First, the probe is an id-fetch, **not**
`COUNT(*)`: the scan cost is identical, but a below-cap probe returns the
*exact denotation* — the substatement never evaluates \(P\) again. Second,
the projection is ids-only, so a capped probe's cost is bounded at \(c+1\)
id rows regardless of how wide the predicate's rows are.

**Wave 0.** After layer materialisation, every eligible substatement
(anchored, with a fusable single-query predicate) probes concurrently.
Below-cap substatements become **resolved**, holding
\(\mathrm{ids}(D(s))\); the rest are **capped**.

## 5. Refinement

**Roles.** For a resolved neighbour \(n\) of a capped (or derive-only)
substatement \(s\), a **role** is the semi-join image of \(n\)'s resolved
set under one relationship, lifted to symbol level:

$$\rho_{\mathrm{rel}}(N) \;=\; \{\, x \in \mathrm{Inst} \;:\; \mathrm{sym}(x) \in \mathrm{rel\text{-}image}(N) \,\}$$

The lifting matters: because evidence matches per symbol (§2), roles must
too, or a refined set could drop a co-instance the worklist would have
kept.

**Waves.** Substatement \(s\) re-probes \(P(s) \wedge \rho(N)\). When
\(\mathrm{rel} = \mathrm{REFS} \cup \mathrm{HAS}\), a single conjunctive
role would be unsound (the union semantics), so the neighbour contributes
one probe per relationship **branch**; \(s\) resolves iff every branch
combination resolves, and its set is the union of the combination results —
exact under the union semantics. Waves iterate to a fixpoint: a
substatement resolved in wave \(k\) can serve as the binding for its other
neighbours in wave \(k+1\), so constraint flows across the tree.

**Binding choice.** Each candidate binds only its **smallest** resolved
neighbour, re-probing only when a strictly smaller binding appears: a
role's evaluation cost scales with the bound set, and a broad neighbour can
cost more to conjoin than it narrows. The refined result is a larger — still
sound — superset; the worklist narrows the rest.

## 6. Properties

**Theorem 1 (probe exactness).** A wave-0 resolution equals
\(\mathrm{ids}(D(s))\); a refinement resolution equals
\(\mathrm{ids}(D(s) \wedge \rho(N))\). In both cases the
`LIMIT` never truncated the returned set (it fired only in the discarded
Capped case), so resolutions are deterministic and complete for their
predicate.

**Theorem 2 (soundness).** Every resolved set is a superset of the final
selection: \(o(s) \subseteq \mathrm{resolved}(s)\).

*Proof sketch.* Wave 0: \(o(s) \subseteq D(s)\) by definition. Refinement:
let \(x \in o(s)\). Then \(x \in D(s)\), and by the composition equation
there is \(y \in o(n)\) with \((x, y) \in E_{\mathrm{rel}}\). By
monotonicity \(o(n) \subseteq \mathrm{resolved}(n) = N\), and the role is
at least as permissive as evidence
(\(E_{\mathrm{rel}} \subseteq \rho_{\mathrm{rel}}\) — both are
symbol-level, and \(\rho\) carries no direct-only or budget conditions), so
\(x \in \rho(N)\), hence \(x \in D(s) \wedge \rho(N)\). For union
relationships, \(x\) satisfies at least one branch in every combination, so
it survives the union of combination results. ∎

Soundness is exactly the contract both consumers (§7) require: a resolved
set is *exact in composition* — it may exceed *o(s)*, never undershoot it.

**Theorem 3 (termination).** Each wave either resolves at least one
substatement or is empty; candidates re-probe only on strictly
smaller bindings; hence the loop takes at most
\(\lvert\text{substatements}\rvert\) waves.

## 7. Consumers of resolved sets

- **Emission.** A resolved substatement's read conjoins the id list with
  its predicate: the current-instance query becomes a primary-key fetch, while
  the predicate's contributions to the neighbourhood families remain
  byte-identical to the unprobed emission (Theorem 1 makes the conjunction
  a no-op on the current family).
- **Scopes.** The scope builders hand a resolved neighbour's ids to the
  neighbourhood and containment queries in place of a *condition* the
  index would otherwise materialise in full — by Theorem 2 the id set
  over-approximates every row the composition can keep, so nothing valid
  is scoped away. An empty resolved set still scopes (the neighbour
  matches nothing); it never widens to unscoped. This substitution is what
  removed the multi-million-row scope resolution of §1.

The worklist downstream is unchanged; probes only ever hand it smaller,
exact inputs.

## 8. Implementation notes

The probe SQL was validated with `EXPLAIN ANALYZE` against a production
index (5.9 M symbols, 23.3 M references) before the engine work, and three
times the PostgreSQL planner needed help:

- **REFS roles** work as plain `IN`-subqueries, driven through the
  `to_symbol` / `from_object` indexes.
- **HAS (containment) roles** must be `MATERIALIZED` CTEs: as
  `IN`-subqueries the planner refused to drive the probe from the tiny
  role set (1.8–10.6 s against full-table scans); materialised first, the
  outer query runs id-driven (0.5–6.9 ms).
- The **parents display query** needed a redundant semi-join —
  `sr.to_symbol IN (SELECT symbol FROM symbol_instances WHERE id = ANY(…))`,
  implied by its own joins — to stop a full reference-table hash-join
  (933 ms → 10 ms for a zero-row result).

Probes run in combined visibility through the in-RAM SQL cache, so repeat
and refinement probes are cache hits. All id binds are canonicalised
(sorted, deduplicated): id lists otherwise arrive in selection order,
which varies with concurrent completion order, and the cache keys on
rendered binds — non-canonical binds turned identical queries into
guaranteed misses.

## 9. Evaluation

The §1 query, byte-identical output at every stage:

| Stage | Latency |
|--:|--:|
| syntax-directed baseline | 53.7 s |
| probes + refinement + probe-informed scopes | 5.7 s |
| smallest-neighbour binding + scoped containment families | 1.2 s |
| canonical cache binds + parents probe route | ≈0.3 s |

## 10. Configuration and observability

- `--probe-cap` / `ASKL_PROBE_CAP` (default 1000): the resolution
  threshold. `0` disables resolution — every probe caps and reads stay
  predicate-driven.
- Every probe records a `ProbeActivation` (substatement text, resolved
  count or capped, wave number) on the execution context — the hook tests
  and diagnostics use to assert which substatements probed, when, and how
  they classified.

## Where to read more

- [Execution Engine](/docs/design/execution-engine) — the worklist
  fixpoint, dependency kinds, and weakness/bindness.
- [Partitioning a Materialisation](/docs/design/shards) — the layer forest probes and
  reads execute against.
