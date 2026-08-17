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
\(\mathrm{rel}(c) \subseteq \{\mathrm{REFS}, \mathrm{HAS}\}\), a set of
**edge kinds**: \(\mathrm{REFS}\) is a reference edge, one symbol naming
another; \(\mathrm{HAS}\) is a containment edge, one symbol enclosing
another. The `{ }` default is both kinds, and an edge of either then
counts.

**Composition.** Execution does not return \(D(c)\); it returns the
**selection** \(N_c \subseteq D(c)\), the fixpoint of the worklist
propagation in which neighbouring substatements constrain each other
through **edge evidence**
([semantics §7](/docs/design/semantics/#selections)). Write
\(\mathrm{sym}(x)\) for the symbol an instance \(x\) belongs to, and
\(\hat{E}_{c,n}\) for the evidence relation between \(c\)'s symbols and
neighbour \(n\)'s — the edges of the kinds their nesting asks for,
oriented from \(c\)'s side, so that \(\hat{E}_{c,n}\) is the union of its
per-kind parts \(\hat{E}^{\mathrm{rel}}_{c,n}\). The **constraining
neighbours** of \(c\) — its parent, each strong child, each `use()`
provider — impose one condition each, and a row of \(c\) survives only
with evidence to every one of them; weak neighbours constrain nothing
([Weakness and Bindness](/docs/design/semantics/#weakness-and-bindness)).
This page uses one direction of that equation, once per constraining
neighbour \(n\):

$$N_c \;\subseteq\; \bigl\{\, x \in D(c) \;:\; \exists\, y \in N_n.\; \bigl(\mathrm{sym}(x),\, \mathrm{sym}(y)\bigr) \in \hat{E}_{c,n} \,\bigr\}$$

Each constraining neighbour is thus a necessary condition on \(N_c\) *on
its own*, dropping the others being a weakening — which is what lets the
refinement below pick whichever single one it likes. Evidence is matched
per symbol, since only \(\mathrm{sym}(x)\) occurs on the left; §5's roles
are lifted to symbol level for exactly that reason.

**Monotonicity.** The worklist only ever narrows: at every intermediate
stage the current selection of any substatement is a superset of \(N_c\).
That is a statement about the worklist's own stages; the probe
resolutions below are a separate sequence, and §6 argues about them by
induction on the wave that produced them.

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
set and \(\mathrm{inst}(I)\) for the instances a set \(I\) of ids names —
the coercion that lets an id list stand where §2 speaks of instances. For
an arbitrary set \(S \subseteq \mathrm{Inst}\) and one edge kind
\(\mathrm{rel}\), the **role** \(\rho_{\mathrm{rel}}(S)\) is the
semi-join image of \(S\) under that kind, taken in the direction that
puts the candidate on the left and lifted to symbol level:

$$\rho_{\mathrm{rel}}(S) \;=\; \{\, x \in \mathrm{Inst} \;:\; \mathrm{sym}(x) \in \mathrm{rel\text{-}image}(S) \,\}$$

where \(\mathrm{rel\text{-}image}(S)\) is the set of symbols joined by a
\(\mathrm{rel}\) edge to the symbol of some member of \(S\). The lifting
matters: because evidence matches per symbol (§2), roles must too, or a
refined set could drop a co-instance the worklist would have kept. Two
properties carry the soundness argument of §6.

- **Domination.** For every \(S\) and every edge kind, the instances that
  something in \(S\) evidences *through that kind* are inside the role:
  \((\hat{E}^{\mathrm{rel}}_{c,n})^{-1}[S] \subseteq \rho_{\mathrm{rel}}(S)\),
  writing
  \((\hat{E}^{\mathrm{rel}}_{c,n})^{-1}[S] = \{\, x \in \mathrm{Inst} : \exists\, y \in S.\; (\mathrm{sym}(x), \mathrm{sym}(y)) \in \hat{E}^{\mathrm{rel}}_{c,n} \,\}\)
  for the inverse image of \(S\) under evidence of that kind.
  The role reads the same edges at the same symbol level and adds none of
  the conditions a read may impose on top (direct-only containment, the
  result budget), so it can only be the more permissive of the two.
- **Monotonicity.** \(S \subseteq S'\) implies
  \(\rho_{\mathrm{rel}}(S) \subseteq \rho_{\mathrm{rel}}(S')\), the image
  being an existential over \(S\).

**Waves.** For a resolved neighbour \(n\), command \(c\) re-probes the
intersection \(D(c) \cap \rho_{\mathrm{rel}}(\mathrm{inst}(\mathrm{res}(n)))\)
— its own denotation, narrowed to what the neighbour can reach. When the
relationship carries both edge kinds, one conjunctive role would be
unsound (an edge of *either* kind is evidence), so the neighbour
contributes one probe per kind — a **branch** — and \(c\) resolves iff
every branch resolves, its set being the union of the branch results.
With one neighbour bound at a time (below), a branch *combination* is a
single branch; the machinery is the general one, taking one branch per
bound neighbour and capping the number of combinations, a candidate with
a wider fan-out staying on the predicate path. Because the branches are
unioned, a resolved set can hold more than \(k\) ids: the cap bounds each
branch, not their union. Waves iterate to a fixpoint: a substatement
resolved in wave \(i\) can serve as the binding for its other neighbours
in wave \(i+1\), so constraint flows across the tree.

**Binding choice.** Each candidate binds only its **smallest** resolved
neighbour, re-probing only when a strictly smaller binding appears: a
role's evaluation cost scales with the bound set, and a broad neighbour can
cost more to conjoin than it narrows. The refined result is a larger — still
sound — superset; the worklist narrows the rest. Any *one* constraining
neighbour may be bound: by §2 each is a condition \(N_c\) must satisfy
regardless of the others, so refining against one of several siblings
keeps every row the conjunction can keep and merely leaves the siblings'
conditions to the worklist.

## 6. Properties {#properties}

**Theorem 1 (probe exactness).** Suppose the probed predicate is
**fusable** — all of it goes into the one database statement the cap
truncates, so nothing is filtered after the limit — and the visible layer
set is the same when the probe runs and when the read runs. Then a wave-0
resolution equals \(\mathrm{ids}(D(c))\), and a refinement resolution
equals
\(\mathrm{ids}\bigl(D(c) \cap \rho_{\mathrm{rel}}(\mathrm{inst}(\mathrm{res}(n)))\bigr)\).
In both cases the cap never truncated the returned set (it fired only in
the discarded Capped case), so resolutions are deterministic and complete
for their predicate.

The first hypothesis is why eligibility asks for a single-query predicate
(§4): a limit applied *before* a later condition would return a truncated
set that the condition then thins again, exact for nothing. The second is
why probes run after layer materialisation and against the visibility the
read will use — \(\mathrm{Inst}\) must not move underneath an id list.

**Theorem 2 (soundness).** Every resolved set is a superset of the
selection: \(N_c \subseteq \mathrm{inst}(\mathrm{res}(c))\), provided each
binding it was obtained from is a constraining neighbour of §2.

That proviso is discharged by an invariant the engine maintains: an
anchored command is never weak
([semantics §9.2](/docs/design/semantics/#bindness)), and only anchored
substatements probe (§3), so every resolved neighbour is strong and hence
a conjunct of the composition in its own right. How many siblings it has
no longer enters the hypothesis — under a conjunction each of them is a
separate necessary condition, so binding one is a relaxation and never an
overreach.

*Proof sketch, by induction on the wave in which a set was resolved.*
Wave 0: \(\mathrm{res}(c) = \mathrm{ids}(D(c))\) by Theorem 1, and
\(N_c \subseteq D(c)\) by definition. Wave \(i+1\): let \(c\) resolve by
binding \(n\), itself resolved in some wave \(\le i\), and take
\(x \in N_c\). Then \(x \in D(c)\), and since \(n\) constrains \(c\), §2
gives \(y \in N_n\) with
\(\bigl(\mathrm{sym}(x), \mathrm{sym}(y)\bigr) \in \hat{E}_{c,n}\). That
relation is the union of its per-kind parts, so the witnessing edge has
some kind \(\mathrm{rel}\), and
\(x \in (\hat{E}^{\mathrm{rel}}_{c,n})^{-1}[N_n]\). The induction
hypothesis gives \(N_n \subseteq \mathrm{inst}(\mathrm{res}(n))\), so
domination followed by monotonicity (§5) yields

$$x \;\in\; (\hat{E}^{\mathrm{rel}}_{c,n})^{-1}[N_n] \;\subseteq\; \rho_{\mathrm{rel}}(N_n) \;\subseteq\; \rho_{\mathrm{rel}}\bigl(\mathrm{inst}(\mathrm{res}(n))\bigr)$$

and hence \(x \in D(c) \cap \rho_{\mathrm{rel}}(\mathrm{inst}(\mathrm{res}(n)))\),
which by Theorem 1 is what that kind's branch enumerates. So there is a
combination — with one neighbour bound, the single branch for
\(\mathrm{rel}\) — all of whose branches \(x\) satisfies, and \(c\)'s
resolved set, being the union over combinations, keeps \(x\). ∎

Soundness is exactly the contract both consumers (§7) require: a resolved
set is *exact in composition* — it may exceed \(N_c\), never undershoot it.

**Theorem 3 (termination).** The refinement loop runs at most
\(\lvert\text{substatements}\rvert\) waves. The variant is the number of
**unresolved** substatements, and what makes it fall is permanence: a
resolved substatement keeps the set it resolved to, is never re-probed,
and never returns to being capped. So a wave that resolves anything
lowers the count for good, a wave that resolves nothing ends the loop,
and the count starts at most at the number of substatements. (The
smallest-binding rule of §5 bounds how often one *candidate* re-probes
within that; it is an economy, and no part of this argument.)

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
