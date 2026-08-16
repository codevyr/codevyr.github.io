---
title: "Design: The Command Algebra"
description: "How verbs fold into a command — override vs accumulate — and how filters and content populates combine into the predicate and the content map"
weight: 135
---

Every substatement carries one **command** — its bag of verbs. This page
treats the command as a small algebraic object: how verbs fold into it,
and how its parts combine into the two functions the other design pages
use — the combined filter predicate \(F_c\) (keyed in
[Layer Keys and Hashing](/docs/design/layer-keys)) and the content map
\(f_c\) (the function [Partitioning a Materialisation](/docs/design/shards) splits
across cached layers). As on those pages, \(c\) ranges over
layer-bearing commands and \(t = 1, 2, \dots\) indexes layer-creating
top-level statements — each statement's commands contribute to its
materialisation; \(C(\ell)\) is the content rows of layer \(\ell\) and
\(R\) a project's root layer. \(c\) is the assembled command itself
(§2's fold), and subscripting by \(c\) means *that command's*:
\(F_c\) its filter predicate, \(U_c\) its combined populate, \(f_c\)
its content map.

## 1. What a verb denotes

Each verb denotes a functional contribution. A content verb
**populates**: `search("foo")` denotes a **populate** \(u\) — a
function that takes a corpus slice \(A\) (the content rows of some set
of layers) and returns the **content** it writes for *just that
slice*. That content may be rows *derived from* \(A\) — for `search`,
the byte-range matches found in the slice's text; for `loc`, a
resolved file position — or rows *added outright*, like a new file in
an incremental layer; either way it depends on nothing outside \(A\)
and the verb's own arguments. Writing content is what distinguishes a
populate from a filter, which only keeps or drops rows that already
exist. An added-outright populate is a constant function of \(A\), and
constants are union-additive (\(X = X \cup X\)), so such populates
satisfy the
[layer-decomposability axiom](/docs/design/shards/#2-verb-semantics-and-the-decomposition-axiom)
trivially. The names track the code: the engine fills a layer's rows
by running the verb's populate closures, and `search`'s populate is
implemented as a `ShardedScan`. `project("linux")` denotes a
**predicate** on rows. Verbs are the units of *composition*: the
sections below fold them into the command (§2), compose one selection
\(\sigma_{F_c}\) from its filters (§3) and one union of populates from
its content verbs (§4), and evaluation happens at the composed level —
what a command contributes to its statement's materialisation is the content map
\(f_c(A)\) (§5).

## 2. Assembling the command

A command is its verbs folded in source order. Write
\(\langle v \rangle\) for the one-verb command and \(\triangleright\)
for the fold step:

$$c \;=\; \langle v_1 \rangle \,\triangleright\, \langle v_2 \rangle \,\triangleright\, \dots \,\triangleright\, \langle v_m \rangle$$

\(\triangleright\) merges per verb *tag*, and today each tag hardcodes
one of two behaviours: **override** — a later verb replaces an earlier
one with the same tag (`project("a") project("b")` keeps only `"b"`;
`select`, `project`, and the type-kind filters behave this way) — or
**accumulate** — both survive (`search("a") search("b")`, or filters of
different kinds). This is the algebra of record update (Z notation's
function override; Docker-style overlays): **associative**, so the fold
needs no parentheses, but **not commutative**, since override gives the
later verb precedence. Source order is semantics — which is why
\(H(c)\) also hashes per-verb inputs in source order
([layer-keys §4](/docs/design/layer-keys/#4-combining-verbs-the-command-hash)).
Making the override/accumulate choice explicit in the syntax rather
than hardcoded per verb is future work.

A verb can carry **several aspects at once**, contributing each to a
different slot of \(c\): `search` both *populates* and
*filters* (its match predicate constrains the selection); `project`
only filters; the aspects a verb lacks are no-ops in the fold. The rest
of this page combines the surviving contributions, one aspect at a
time.

## 3. Filters compose

Each filter \(g\) of the assembled command induces a **selection**
\(\sigma_g\): the function keeping exactly the rows that satisfy
\(g\). Selections compose as functions, and composing them is the same
as conjoining their predicates:

$$\sigma_{g_1} \circ \sigma_{g_2} \;=\; \sigma_{g_1 \wedge g_2} \;=\; \sigma_{g_2} \circ \sigma_{g_1}$$

— order never matters, which is why every page writes one combined
predicate \(F_c = \bigwedge_j g_j\) (the \(F\) of
[Layer Keys and Hashing](/docs/design/layer-keys)) rather than a
composition chain. Nothing here is conjunction-specific: \(\sigma\) is
defined for any Boolean predicate tree — \(F\) itself carries And/Or/Not
nodes, and the emission predicate of
[Cost-Based Execution §2](/docs/design/cost-based-execution/#2-model)
adds an OR of selector branches — and any such tree normalises to a
disjunction of conjunctions. Conjunction is merely the law by which
*separate* filters combine.

\(F_c\) and \(\sigma_{F_c}\) carry the same information in different
categories, and both are needed: the predicate is the *syntactic* object
the cache keys hash ([layer-keys §2](/docs/design/layer-keys/#2-the-filter-hash)),
the selection is the *function* evaluation applies — and \(\sigma\) is
many-to-one, since equivalent but structurally different trees induce
the same selection while hashing apart (harmless: at worst a redundant
materialisation).

## 4. Content verbs union

Content verbs, by contrast, do *not* compose sequentially: no content
verb reads another's output — their mutual independence is a keying
requirement
([layer-keys §4](/docs/design/layer-keys/#4-combining-verbs-the-command-hash))
— so several populates combine by pointwise **union**,
\((u \cup u')(A) = u(A) \cup u'(A)\). Write \(U_c\) for the command's
**combined populate**, with \(u_1, \dots, u_n\) the populates its
content verbs contribute (\(n \le m\), one per content verb; a
different letter than the folded verbs \(v_i\)):

$$U_c \;=\; u_1 \cup \dots \cup u_n \qquad U_c(A) \;=\; u_1(A) \cup \dots \cup u_n(A)$$

Applied to a slice \(A\), \(U_c(A)\) is every row any of the command's
populates writes for \(A\) — for `search("foo") search("bar")`, all
matches of `foo` in \(A\)'s text plus all matches of `bar`, unfiltered.

## 5. The content map

A statement's materialisation
([terminology](/docs/design/overview/#terminology)) is assembled as the union
of per-command node groups. The content
map defined here is per command — it is what command \(c\) contributes
to that union. What a read of a command's layers *observes*
is computed from an input slice \(A\) in two steps: run each content
verb's populate on \(A\) and union their rows, then keep the rows that
pass the combined filter \(F_c\). The content map \(f_c\) is that
two-step recipe written as one function, composing the two objects the
previous sections built — §4's combined populate and §3's selection:

$$f_c \;=\; \sigma_{F_c} \circ U_c$$

— read right to left: \(U_c\) applies first, the selection after. The reason to name this composite at all: the
[partition](/docs/design/shards) needs *one function per
command* that it can aim at different layer slices and split
across them, and \(f_c\) is that function.

The composition is the *semantic* definition; physically its two factors
apply at different times. Materialisation **stores the populates' output
unfiltered**: no \(\sigma_{F_c}\) runs over the rows being inserted. At
most, \(F_c\)'s object-narrowing part (today: `project(...)`) and any
fused container scope restrict the corpus the populate *reads* —
`search` skips a filtered-out project's content entirely, while `loc`
consults \(F_c\) not at all (its own arguments fix what it reads). The
selection \(\sigma_{F_c}\) then applies at **read time**: every read of
the command's layers conjoins \(F_c\) with the layers' row ids in one
emission query, so reads observe exactly \(f_c\) even though the stored
rows are the unfiltered union.

There is no circularity here: \(F_c\) is **not** a function of the
populates — filters and populates are disjoint slots of \(c\)
([acyclicity](/docs/design/layer-keys/#6-acyclicity)). A dual-aspect
verb like `search` contributes its populate to the union, while its match
predicate joins the emission predicate
([Cost-Based Execution §2](/docs/design/cost-based-execution/#2-model)),
not \(F_c\). Lowercase \(f_c\) is a function on rows; uppercase
\(F_c\) the predicate it applies; the composition order reads
populate-then-filter.

Because a selection acts row by row and a union of additive maps is
additive, \(f_c\) is additive — splittable across disjoint inputs —
exactly when each populate \(u_i\) is. What that additivity buys,
and why it is the whole caching story, is the
[layer-decomposability axiom](/docs/design/shards/#2-verb-semantics-and-the-decomposition-axiom).

## Where to read more

- [Partitioning a Materialisation](/docs/design/shards) — the forest \(f_c\) is
  split across, and the theorems the split satisfies.
- [Layer Keys and Hashing](/docs/design/layer-keys) — how \(F_c\) and
  the per-verb inputs become the cache keys \(H(c)\).
- [Cost-Based Execution](/docs/design/cost-based-execution) — the
  emission predicate \(P(s)\) and how measured cardinality plans the
  query.
