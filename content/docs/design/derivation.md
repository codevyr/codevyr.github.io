---
title: "From Result to Cache"
description: "Deriving the engine's structure from what a query returns: selections as fixpoints, upper bounds as cache keys, and why production has to be partitioned"
weight: 95
---

The design pages each answer a *how*. This page answers the *why* they
share: starting from what a query returns and asking, at each step,
what could be reused, it arrives at the two caches and at the shape of
the layer storage. Nothing here is new machinery — every result has a
page that develops it — but the order is the order the problems
actually arise in.

## 1. The product {#product}

A query \(Q\) returns a graph: symbols, and the reference and
containment edges among them.

$$\mathrm{askl}(Q) \;=\; (N, E)$$

\(E\) is derived — the index's edges restricted to the nodes that
survive — so the whole question is which nodes \(N\) a query selects,
and how to avoid computing them twice.

## 2. Nodes decompose along the syntax {#decomposition}

A query is an ordered list of statements, and its nodes are the union
of what the statements select. A statement is a tree of substatements,
each carrying one command, and its nodes are the union over those
commands:

$$N \;=\; \bigcup_{s \,\in\, Q} N_s, \qquad N_s \;=\; \bigcup_{c \,\in\, \mathrm{cmds}(s)} N_c$$

The unions are plain, but the *order* of \(Q\) is not decorative:
statements are the only time axis the language has, and §6 is where
that matters. Within a statement, nesting expresses constraint rather
than sequence — which is exactly what the next step is about.

## 3. What one command selects {#selection}

Write \(D(c)\) for the rows matching command \(c\)'s own filters —
its **denotation**, computable from the command alone. Selection is
narrower than that: a row survives only if it also has evidence with
what \(c\)'s neighbours selected. For every constraining neighbour
\(n\) of \(c\) (its parent, and each of its children):

$$N_c \;=\; \{\, x \in D(c) \;:\; \forall n.\ \exists\, y \in N_n.\ (x, y) \in E_{\mathrm{rel}} \,\}$$

Each \(N_c\) is defined in terms of its neighbours' \(N_n\), which are
defined in terms of theirs. The system is mutually recursive, and its
answer is a **fixpoint**: start from the denotations and narrow, a
command at a time, until nothing changes. That loop is the
[execution engine](/docs/design/execution-engine); it terminates
because every step only removes rows.

## 4. A fixpoint resists caching {#fixpoint-resists-caching}

The natural thing to cache is \(N_c\) — a command's answer. It is also
the one thing that cannot be named cheaply. \(N_c\) is a joint
function of the whole system: add a filter to a *sibling's* child and
the fixpoint may move, so any honest key for \(N_c\) would have to
name the entire query. A cache keyed on less would serve one query's
answer to another.

So selections are not cached. Something upstream of them is.

## 5. Cache an upper bound instead {#upper-bound}

Before the fixpoint runs, each command *reads*: it asks the database
for a superset of its selection, using a predicate \(P(c)\) built from
its own filters plus whatever its neighbours can contribute as
conditions rather than as results.

$$N_c \;\subseteq\; \llbracket P(c) \rrbracket \;\subseteq\; D(c)$$

This is the object worth caching, for a reason the fixpoint lacks:
\(P(c)\) mentions only \(c\) and its immediate neighbourhood. Edit one
statement of a long query and every other command's read is
*byte-identical* — same SQL, same binds — so it hits the in-RAM result
cache untouched. Two consequences follow. Interactive editing gets
cheap, which is what the cache is for; and it pays to make \(P(c)\)
*tight* rather than merely correct, which is what
[cost-based execution](/docs/design/cost-based-execution) does by
measuring cardinality with capped probes before committing to a plan.

## 6. Visibility joins the key {#visibility}

A read is not evaluated against the index at large. It runs against
the layers currently visible, and visibility grows as the query runs:
before its own reads, each statement **materialises** the rows its
content verbs produce onto new layers, which later statements see.
This is where the order of \(Q\) from §2 has teeth.

The visible set is therefore part of every read's key — the SQL binds
the layer ids. That keeps the cache honest when a layer appears, at a
price: it **fragments**. Two queries that differ only in an earlier
statement have different visibility, hence different keys, hence no
sharing — even where they do identical work.

## 7. What a materialisation contains {#materialisation}

So look at what the materialising step itself produces. For command
\(c\) of statement \(t\), with \(\Lambda_{t-1}\) the slice visible
before the statement and \(O_c\) the earlier statements' selections it
references by label:

$$M_c \;=\; U_c\Bigl(\bigcup_{\ell \,\in\, \Lambda_{t-1}} C(\ell)\Bigr) \;\cup \bigcup_{o \,\in\, O_c} g_c(o)$$

— rows its populates produce from layer content, plus rows it builds
from earlier selections. Caching *this* is a different proposition
from caching a read: it saves **producing** rows, once, across queries
and across processes, where the in-RAM cache saves **re-reading** rows
already produced inside one process. Those are the
[two tiers](/docs/design/caching).

## 8. Production cannot be cached whole {#whole-fails}

Cache \(M_c\) as one entry and the key must name everything it read —
the entire slice \(\Lambda_{t-1}\). Then adding a single ephemeral
layer upstream changes the key, and a full-text scan over a
kernel-sized corpus is redone on account of a layer that contributed
nothing to it. The most expensive work in the system would be at the
mercy of its cheapest, most volatile input.

## 9. Hence the partition {#partition}

The escape is that \(U_c\) is not an arbitrary function. For the
populates the language actually has, scanning a union of layers gives
the same answer as scanning each and taking the union —

$$U_c\Bigl(\bigcup_{\ell} C(\ell)\Bigr) \;=\; \bigcup_{\ell} U_c\bigl(C(\ell)\bigr)$$

— so \(M_c\) can be stored in pieces, each keyed on the one input it
read. The expensive, stable piece (the scan of the committed corpus)
is then keyed apart from the cheap, volatile ones, and survives their
churn. Which pieces exist, what each one's name folds, and what that
name guarantees is
[Partitioning a Materialisation](/docs/design/shards).

## Where to read more

- [Execution Engine](/docs/design/execution-engine) — the fixpoint of §3.
- [Cost-Based Execution](/docs/design/cost-based-execution) — making §5's bound tight.
- [Caching](/docs/design/caching) — the two tiers of §7.
- [Partitioning a Materialisation](/docs/design/shards) — the split of §9.
