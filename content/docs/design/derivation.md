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

A query \(Q\) returns a graph: the symbol instances that survive it, and
the reference and containment edges among them.

$$\mathrm{askl}(Q) \;=\; (N, E)$$

\(N\) is a set of **instances** throughout this page — the same objects
the \(N_c\) of §3 hold, closed per symbol, so that no instance survives
without the others of its symbol
([semantics §7](/docs/design/semantics/#selections)). A reader sees them
grouped by symbol, and the evidence conditions of §3 are matched at
symbol level, but every set below is a set of instances. \(E\) is derived
— the index's edges restricted to the nodes that survive — so the whole
question is which nodes \(N\) a query selects, and how to avoid
computing them twice.

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

Write \(D(c)\) for what command \(c\) denotes on its own — the rows its
own predicate \(P(c)\) matches, filters and selector branches together,
computable from the command alone
([semantics §6](/docs/design/semantics/#denotation)). Selection is
narrower than that: a row survives only if it also has evidence with what
**each** of \(c\)'s constraining neighbours selected — its parent, each
strong child, each label provider, collected as \(\mathrm{Cons}(c)\), so
that `func { "a" ; "b" }` asks for a call to both. Writing
\(\mathrm{sym}(x)\) for the symbol an instance \(x\) belongs to and
\(\hat{E}_{c,n}\) for the **evidence relation** between
symbols — a reference or a containment edge
([semantics §7](/docs/design/semantics/#selections)):

$$N_c \;=\; \Bigl\{\, x \in D(c) \;:\; \forall\, n \in \mathrm{Cons}(c).\ \exists\, y \in N_n.\ \bigl(\mathrm{sym}(x),\, \mathrm{sym}(y)\bigr) \in \hat{E}_{c,n} \,\Bigr\}$$

Each \(N_c\) is defined in terms of its neighbours' \(N_n\), which are
defined in terms of theirs. The system is mutually recursive, and its
answer is its **greatest** fixpoint: start above it and narrow, a
command at a time, until nothing changes. Running that loop is
[evaluating the fixpoint](/docs/design/evaluation); it terminates
because every step only removes rows.

## 4. A fixpoint resists caching {#fixpoint-resists-caching}

The natural thing to cache is \(N_c\) — a command's answer. It is also
the one thing that cannot be named cheaply. \(N_c\) is a joint
function of the whole system: add a filter to a *sibling's* child and
the fixpoint may move, so an honest key for \(N_c\) would have to name
every command a constraint can reach it from — \(c\)'s whole connected
component — and, because the reads that fixpoint starts from run under
whatever earlier statements made visible (§6), every statement before it
as well. A cache keyed on less would serve one query's answer to another.

So selections are not cached. Something upstream of them is.

## 5. Cache an upper bound instead {#upper-bound}

Before the fixpoint runs, each command *reads*: it asks the database
for a superset of its selection, evaluating its own predicate \(P(c)\)
together with whatever its neighbours can contribute as conditions
rather than as results. Write \(\mathrm{nb}(c)\) for those neighbour
conditions:

$$N_c \;\subseteq\; \llbracket\, P(c) \wedge \mathrm{nb}(c) \,\rrbracket \;\subseteq\; D(c)$$

The left inclusion is a hypothesis rather than an observation: the
fixpoint is approached from above, so a read may stand in for a
denotation as the starting point only because it still contains \(N_c\)
([semantics §7](/docs/design/semantics/#selections)). The right one is
what makes the read cacheable, for a reason the fixpoint lacks: the
read's predicate mentions only \(c\) and its immediate neighbourhood.
Edit one statement of a long query and every command whose visibility
the edit leaves untouched reads *byte-identically* — same SQL, same
binds — so it hits the in-RAM result cache without being recomputed.
Which commands those are is §6's subject, and it is not all of them: an
edit that changes what a later statement can see changes that
statement's reads too. Two consequences follow. Interactive editing gets
cheap, which is what the cache is for; and it pays to make \(P(c)\)
*tight* rather than merely correct, which is what
the [planner](/docs/design/planning) does by measuring cardinality with
capped probes before committing to a plan.

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

So look at what the materialising step itself produces. Four symbols,
each owned elsewhere: \(U_c\) is the command's combined populate
([semantics §4](/docs/design/semantics/#content-verbs-union)); \(C(\ell)\)
is the content stored on layer \(\ell\), and \(\Lambda_{t-1}\) the slice
visible before statement \(t\) — which from here on is *the statement
\(c\) belongs to*, with one project in view so that the per-project root
stays silent ([shards §1](/docs/design/shards/#notation)); and \(g_c\)
builds rows from each output in \(O_c\), the earlier statements'
selections the command references by label
([shards §3](/docs/design/shards/#node-kinds)). Command \(c\)'s
contribution to the materialisation of statement \(t\) is then

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

- [Evaluating the Fixpoint](/docs/design/evaluation) — the fixpoint of §3.
- [Planning from Measured Cardinality](/docs/design/planning) — making §5's bound tight.
- [Caching](/docs/design/caching) — the two tiers of §7.
- [Partitioning a Materialisation](/docs/design/shards) — the split of §9.
