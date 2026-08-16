---
title: "Planning from Measured Cardinality"
description: "Why syntax cannot predict how much of the index a query touches, and how the engine measures before it reads: anchors, capped id probes, and semi-join refinement with its soundness argument"
weight: 115
aliases:
  - /docs/design/cost-based-execution/
---

Nothing in a query's syntax says how much of the index it will touch.
`func` and `mod("amdgpu")` have the same shape and differ by six orders
of magnitude, and which of them ought to drive a query depends on the
index it runs against rather than on how it was written. So the engine
measures first: it asks the index how large each substatement is —
cheaply, and with a bound on the effort — and lets whatever turns out to
be small drive the rest. This page gives that scheme: anchoring as static
eligibility, probes as exact bounded evaluation, refinement as semi-join
propagation, and the three properties it rests on — probe *exactness*,
refinement *soundness* against the composition semantics, and
*termination*.

## 1. Motivation {#motivation}

A query language has to decide who chooses the plan, and the cheapest
answer is to let the syntax carry it. Split a command's conditions into
*selectors*, which initiate SQL, and *filters*, which contribute WHERE
fragments; let the user pick between them by how a condition is written
(`func` versus `func("name")`, plus an explicit `filter=` toggle). The
tension is that the plan is then fixed before any cardinality is known,
and the user who now owns it does not know the cardinalities either. For

```askl
mod("amdgpu") { func { "drm_dev_enter" } }
```

such an engine cannot see that the leaf denotes a handful of rows and
that everything else should be derived from it. It resolves the bare
`func` condition into millions of ids, one SQL statement returns 2.67 M
rows (1.8 GB), and the query takes 53.7 s. Under the model below the same
query returns the identical result from kilobyte-sized, index-driven SQL
statements.

## 2. Model {#model}

**Substatements and denotations.** A query is a forest of substatements,
each carrying exactly one command; this page indexes by the command
\(c\), following [Queries and their Meaning](/docs/design/semantics).
That page defines the two objects planning starts from — a command's own
**predicate** \(P(c)\), its filters conjoined with the disjunction of its
selector branches, and its **denotation**

$$D(c) \;=\; \{\, x \in \mathrm{Inst} \;:\; x \models P(c) \,\}\,,$$

the symbol instances satisfying it, drawn from \(\mathrm{Inst}\), the
universe of instances the query can see
([semantics §6](/docs/design/semantics/#denotation)). Each nested
substatement also carries a **relationship** to its parent,
\(\mathrm{rel}(c) \subseteq \{\mathrm{REFS}, \mathrm{HAS}\}\), naming which
edge kinds relate their instances (the `{ }` default is the union
\(\mathrm{REFS} \cup \mathrm{HAS}\): either kind counts).

**Composition.** Execution does not return \(D(c)\); it returns the
**selection** \(N_c \subseteq D(c)\), the fixpoint of the worklist
propagation in which neighbouring substatements constrain each other
through **edge evidence** ([semantics §7](/docs/design/semantics/#selections)):
for a command \(c\) with neighbour \(n\),

$$N_c \;=\; \{\, x \in D(c) \;:\; \exists\, y \in N_n.\; (x,y) \in E_{\mathrm{rel}} \,\}$$

(for constraining neighbours; weak neighbours impose no such condition —
see [Weakness and Bindness](/docs/design/semantics/#weakness-and-bindness)).
Here \(E_{\mathrm{rel}}\) is the evidence relation: a REFS edge or a
containment edge, matched **per symbol** — if any instance of a symbol is
evidenced, all of that symbol's instances survive.

**Monotonicity.** The worklist only ever narrows: at every intermediate
stage the current selection of any substatement is a superset of \(N_c\).
This is the single fact every argument below leans on.

The planning problem: decide, per substatement, whether to materialise
\(D(c)\) independently, derive it from neighbours, and in what order —
using the index itself as the cardinality oracle.

## 3. Static eligibility: anchors {#anchors}

Materialising \(D(c)\) is only *meaningful* for some substatements. A
substatement is **anchored** iff its predicate contains at least one
**anchor** — a verb that can produce instances on its own, defined
together with the completeness rule it feeds in
[semantics §9.2](/docs/design/semantics/#bindness).

What matters to planning is the converse. A command of pure constraints —
bare type selectors, `project(...)`, `preamble` — has a denotation the
size of the index, never worth materialising unconstrained, so such
substatements only ever derive. Anchoring answers *may this substatement
drive?* Cost, measured next, answers *should it, and when?*

## 4. Probes {#probes}

**Definition.** A **probe** asks one question about a set \(X\) of
instances: *are there at most \(k\) of them, and if so, which?* For the
cap \(k\),

$$\mathrm{probe}(X, k) \;=\; \begin{cases} \mathrm{ids}(X) & \text{if } |X| \le k \\ \textbf{Capped} & \text{otherwise} \end{cases}$$

One database statement answers both halves: the engine fetches instance
ids for \(X\) and stops after \(k+1\) of them — a `LIMIT k+1` over an
ids-only projection. A \((k{+}1)\)-th row means **Capped** and the fetched
ids are discarded; anything short of it is the whole of \(X\).

Two deliberate choices. First, the probe is an id-fetch, **not**
`COUNT(*)`: the scan cost is identical, but a below-cap probe returns the
set *exactly* — the substatement never evaluates its predicate again.
Second, the projection is ids-only, so a capped probe's cost is bounded
at \(k+1\) id rows regardless of how wide the underlying rows are.

**Wave 0.** After layer materialisation, every eligible substatement
(anchored, with a fusable single-query predicate) probes its denotation
concurrently. Those below the cap become **resolved**, holding
\(\mathrm{ids}(D(c))\); the rest are **capped**.

## 5. Refinement {#refinement}

**Roles.** Write \(\mathrm{res}(\cdot)\) for a substatement's resolved id
set. For a resolved neighbour \(n\) of a capped (or derive-only) command
\(c\), a **role** is the semi-join image of \(\mathrm{res}(n)\) under one
relationship, lifted to symbol level:

$$\rho_{\mathrm{rel}}(\mathrm{res}(n)) \;=\; \{\, x \in \mathrm{Inst} \;:\; \mathrm{sym}(x) \in \mathrm{rel\text{-}image}(\mathrm{res}(n)) \,\}$$

Here \(\mathrm{sym}(x)\) is the symbol that instance \(x\) belongs to, and
\(\mathrm{rel\text{-}image}(X)\) is the set of symbols joined to \(X\) by
a \(\mathrm{rel}\) edge. The lifting matters: because evidence matches per
symbol (§2), roles must too, or a refined set could drop a co-instance
the worklist would have kept.

**Waves.** Command \(c\) re-probes the intersection
\(D(c) \cap \rho_{\mathrm{rel}}(\mathrm{res}(n))\) — its own denotation,
narrowed to what the neighbour can reach. When
\(\mathrm{rel} = \mathrm{REFS} \cup \mathrm{HAS}\), a single conjunctive
role would be unsound (the union semantics), so the neighbour contributes
one probe per relationship **branch**; \(c\) resolves iff every branch
combination resolves, and its set is the union of the combination results —
exact under the union semantics. Waves iterate to a fixpoint: a
substatement resolved in wave \(i\) can serve as the binding for its other
neighbours in wave \(i+1\), so constraint flows across the tree.

**Binding choice.** Each candidate binds only its **smallest** resolved
neighbour, re-probing only when a strictly smaller binding appears: a
role's evaluation cost scales with the bound set, and a broad neighbour can
cost more to conjoin than it narrows. The refined result is a larger — still
sound — superset; the worklist narrows the rest.

## 6. Properties {#properties}

**Theorem 1 (probe exactness).** A wave-0 resolution equals
\(\mathrm{ids}(D(c))\); a refinement resolution equals
\(\mathrm{ids}\bigl(D(c) \cap \rho_{\mathrm{rel}}(\mathrm{res}(n))\bigr)\).
In both cases the cap never truncated the returned set (it fired only in
the discarded Capped case), so resolutions are deterministic and complete
for their predicate.

**Theorem 2 (soundness).** Every resolved set is a superset of the
selection: \(N_c \subseteq \mathrm{res}(c)\).

*Proof sketch.* Wave 0: \(N_c \subseteq D(c)\) by definition. Refinement:
let \(x \in N_c\). Then \(x \in D(c)\), and by the composition equation of
§2 there is \(y \in N_n\) with \((x, y) \in E_{\mathrm{rel}}\). By
monotonicity \(N_n \subseteq \mathrm{res}(n)\), and the role is
at least as permissive as evidence
(\(E_{\mathrm{rel}} \subseteq \rho_{\mathrm{rel}}\) — both are
symbol-level, and \(\rho\) carries no direct-only or budget conditions), so
\(x \in \rho_{\mathrm{rel}}(\mathrm{res}(n))\), hence
\(x \in D(c) \cap \rho_{\mathrm{rel}}(\mathrm{res}(n))\). For union
relationships, \(x\) satisfies at least one branch in every combination, so
it survives the union of combination results. ∎

Soundness is exactly the contract both consumers (§7) require: a resolved
set is *exact in composition* — it may exceed \(N_c\), never undershoot it.

**Theorem 3 (termination).** Each wave either resolves at least one
substatement or is empty; candidates re-probe only on strictly
smaller bindings; hence the loop takes at most
\(\lvert\text{substatements}\rvert\) waves.

## 7. Consumers of resolved sets {#consumers}

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

## 8. Implementation notes {#implementation-notes}

The probe SQL was validated with `EXPLAIN ANALYZE` against a production
index (5.9 M symbols, 23.3 M references) before any engine work, and one
of its lessons is design rather than tuning: containment roles have to be
handed to the database as materialised subqueries, because otherwise its
planner declines to drive the probe from the tiny role set and scans the
containment tables in full instead (seconds, against milliseconds when
the small side drives) — a role only pays if it is what the read is
driven from.

Probes run in combined visibility through the in-RAM SQL cache, so repeat
and refinement probes are cache hits. All id binds are canonicalised
(sorted, deduplicated): id lists otherwise arrive in selection order,
which varies with concurrent completion order, and the cache keys on
rendered binds — non-canonical binds turned identical queries into
guaranteed misses.

## 9. Evaluation {#evaluation}

The §1 query, byte-identical output at both ends:

| Stage | Latency |
|--:|--:|
| syntax-directed baseline | 53.7 s |
| planning from measured cardinality | ≈0.3 s |

Between the two rows lie probes and refinement (§4–§5), the
smallest-neighbour binding rule that stops a broad neighbour from costing
more than it narrows, resolved ids substituted into scopes (§7), and the
two database details of §8 — materialised containment roles and canonical
id binds.

## 10. Configuration {#configuration}

`--probe-cap` / `ASKL_PROBE_CAP` (default 1000) sets the cap \(k\) of §4,
and with it the resolution threshold. `0` disables resolution outright:
every probe caps, and every read stays predicate-driven — the state the
rest of this page argues against, kept reachable so it can be measured.

## 11. Prior art {#prior-art}

The probe-and-refine loop resembles **semi-join reduction** from
distributed query processing — the SDD-1 line of work, and Bloom-filter
joins after it — where a site is shipped a projection of its neighbour so
that rows with no join partner are discarded before the expensive
operation runs. §5's roles have the same shape: a resolved neighbour's
ids are conjoined into another substatement's predicate, and the
reduction is safe precisely because it only ever over-approximates
(Theorem 2). **Sideways information passing** in Datalog evaluation and
**adaptive query processing** in database engines cover the other half —
using a partially computed result to plan the rest, rather than
committing to a plan before anything is known. What differs here is that
the oracle is the index itself rather than a statistics estimate, and
that a probe may deliberately give up: the cap turns "how many?" into
"few enough to enumerate, or not", which is the only question the
consumers in §7 ask.

## Where to read more

- [Queries and their Meaning](/docs/design/semantics) — the predicate
  and denotation this page plans for, and weakness/bindness.
- [Evaluating the Fixpoint](/docs/design/evaluation) — the worklist
  fixpoint the resolved sets are handed to.
- [Partitioning a Materialisation](/docs/design/shards) — the layer forest probes and
  reads execute against.
