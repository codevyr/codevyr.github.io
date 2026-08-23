---
title: "Queries and their Meaning"
description: "Folding a bag of verbs into one predicate and one populate, and the fixpoint in which neighbouring selections constrain each other"
weight: 105
aliases:
  - /docs/design/command-algebra/
---

The rest of this chapter talks as though every command had one
predicate and one function: reads evaluate a predicate \(P(c)\), cache
keys hash a filter predicate \(F_c\), and the
[partition](/docs/design/shards) splits a content map \(f_c\) across
layer slices. A command has no such thing to begin with. It is a *bag
of verbs* — `search("foo") project("linux") func` — and each verb
contributes something different, or several things at once. Building
those single objects out of the bag is the first half of this page.

The second half says what a query *selects*, which is not what its
commands denote. A command's denotation is computable from the command
alone; its selection is not, because each substatement's neighbours
constrain it and it constrains them back. Meaning here is a **fixpoint
of mutual constraint** — which decides how a query is evaluated
([Evaluating the Fixpoint](/docs/design/evaluation)) and, because a
fixpoint cannot be named cheaply, what the caches are allowed to store
([From Result to Cache](/docs/design/derivation)).

Each substatement carries exactly one command, and each command
belongs to exactly one substatement; per-substatement and per-command
are the same indexing, and this page says "command" for both.
Throughout, \(c\) is an assembled command, and subscripting by \(c\)
means *that command's*: \(F_c\) its filter predicate, \(U_c\) its
combined populate, \(f_c\) its content map, \(N_c\) its selection.

## 1. What a verb denotes {#what-a-verb-denotes}

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
[layer-decomposability axiom](/docs/design/shards/#decomposition-axiom)
trivially. The names track the code: the engine fills a layer's rows
by running the verb's populate closures. `project("linux")` denotes a
**predicate** on rows. Verbs are the units of *composition*: the
sections below fold them into the command (§2), compose one selection
\(\sigma_{F_c}\) from its filters (§3) and one union of populates from
its content verbs (§4), and evaluation happens at the composed level —
what a command contributes to its statement's materialisation is the
content map \(f_c(A)\) (§5).

## 2. Assembling the command {#assembling-the-command}

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
function override): **associative**, so the fold needs no parentheses,
but **not commutative**, since override gives the later verb
precedence. Source order is semantics — which is why \(H(c)\) also
hashes per-verb inputs in source order
([layer-keys §4](/docs/design/layer-keys/#command-hash)). The adopted
[filter-expressions design](/docs/design/filter-expressions) codifies
this per-tag table into a single law — a record of per-dimension
slots, where positive slots replace and exclusions accumulate — and
extends it over boolean slot *expressions* rather than single verbs.

A verb can carry **several aspects at once**, contributing each to a
different slot of \(c\): `search` both *populates* and contributes a
**selector branch** — its match predicate is an alternative way into the
index (§6), not a narrowing of one; `project` only filters; the aspects
a verb lacks are no-ops in the fold. Which slot a contribution lands in
decides how it combines with its neighbours — branches disjoin, filters
conjoin — so the distinction is not bookkeeping. The rest of this page
combines the surviving contributions, one aspect at a time.

## 3. Filters compose {#filters-compose}

Each filter \(\gamma\) of the assembled command induces a **selection**
\(\sigma_\gamma\): the function keeping exactly the rows that satisfy
\(\gamma\). Selections compose as functions, and composing them is the
same as conjoining their predicates:

$$\sigma_{\gamma_1} \circ \sigma_{\gamma_2} \;=\; \sigma_{\gamma_1 \wedge \gamma_2} \;=\; \sigma_{\gamma_2} \circ \sigma_{\gamma_1}$$

— order never matters, which is why every page writes one combined
predicate \(F_c = \bigwedge_j \gamma_j\) rather than a composition chain.
Nothing here is conjunction-specific: \(\sigma\) is defined for any
Boolean predicate tree — \(F_c\) itself carries And/Or/Not nodes, and
§6 puts a disjunction of selector branches on top of it — and any such
tree normalises to a disjunction of conjunctions. Conjunction is merely
the law by which *separate* filters combine.

\(F_c\) and \(\sigma_{F_c}\) live in different categories — the
predicate is the syntactic object cache keys hash
([layer-keys §2](/docs/design/layer-keys/#filter-hash)), the selection
is the function evaluation applies — and the map from predicate to
selection is many-to-one: equivalent but structurally different trees
induce the same selection while hashing apart. So the predicate carries
strictly more than the selection does, and the surplus is its shape. That
is harmless here — at worst a redundant materialisation — but it is why
the two are never used interchangeably.

## 4. Content verbs union {#content-verbs-union}

Content verbs, by contrast, do *not* compose sequentially: no content
verb reads another's output — their mutual independence is a keying
requirement
([layer-keys §4](/docs/design/layer-keys/#command-hash))
— so several populates combine by pointwise **union**,
\((u \cup u')(A) = u(A) \cup u'(A)\). Write \(U_c\) for the command's
**combined populate**, with \(u_1, \dots, u_n\) the populates its
content verbs contribute (\(n \le m\), one per content verb; a
different letter than the folded verbs \(v_i\)):

$$U_c \;=\; u_1 \cup \dots \cup u_n \qquad U_c(A) \;=\; u_1(A) \cup \dots \cup u_n(A)$$

Applied to a slice \(A\), \(U_c(A)\) is every row any of the command's
populates writes for \(A\) — for `search("foo") search("bar")`, all
matches of `foo` in \(A\)'s text plus all matches of `bar`, unfiltered.
A command with no content verb is the case \(n = 0\), where the union has
no terms and \(U_c\) is the constant \(\emptyset\) — the unit of union,
and the reason every statement below can quantify over *all* commands
rather than over the content-bearing ones.

## 5. The content map {#content-map}

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
([acyclicity](/docs/design/layer-keys/#acyclicity)). A dual-aspect
verb like `search` contributes its populate to the union, while its
match predicate joins the command's predicate \(P(c)\) as a selector
branch (§6), not \(F_c\). Lowercase \(f_c\) is a function on rows;
uppercase \(F_c\) the predicate it applies; the composition order reads
populate-then-filter.

Because a selection acts row by row and a union of additive maps is
additive, \(f_c\) is additive — splittable across disjoint inputs —
whenever each populate \(u_i\) is. (Only that direction: a non-additive
populate whose stray rows the filter happens to discard would leave
\(f_c\) additive anyway, so additivity of the composite proves nothing
about its factors.) What that additivity buys,
and why it is the whole caching story, is the
[layer-decomposability axiom](/docs/design/shards/#decomposition-axiom).

## 6. The command's predicate and its denotation {#denotation}

\(F_c\) is not everything a command asserts about a row. A **selector
branch** — a name pattern, a `search` match, a `loc` position — is not
a filter narrowing a set that already exists but an alternative way in,
and a command may carry several: in `func("open") "close"` the two
selectors are OR-ed, so the command matches either. Conjoining branches
would be wrong. A command's own **predicate** is therefore its filters
conjoined with the *disjunction* of its branches,

$$P(c) \;=\; F_c \,\wedge\, \bigl(b_1 \vee \dots \vee b_r\bigr)$$

(the second factor dropped when the command has no branch), and its
**denotation** is what that predicate picks out of \(\mathrm{Inst}\),
the universe of symbol instances the query can see:

$$D(c) \;=\; \llbracket P(c) \rrbracket \;=\; \{\, x \in \mathrm{Inst} \;:\; x \models P(c) \,\}$$

\(D(c)\) is computable from \(c\) alone — no other part of the query
appears in it. That independence is exactly why it is not the answer,
and why it is the right place to start looking for one.

## 7. Selections constrain each other {#selections}

A query is a forest of *substatements*
([terminology](/docs/design/overview/#terminology)), each holding a
command, a **scope** of child substatements, and a **selection** — the
instances it currently holds, a set closed per symbol.

```askl
func("processRequest") {   /* substatement A: functions named processRequest */
    "validate"             /* substatement B: symbols named validate, called by A */
}
```

Nesting is not sequence; it is constraint, and it runs both ways. A
survives only if it actually calls something B selected, and B survives
only among symbols called by something A selected. Write \(N_c\) for
command \(c\)'s selection.

**What counts as evidence.** Write \(\mathrm{sym}(x)\) for the symbol an
instance \(x\) belongs to. For a command \(c\) and a neighbour \(n\), the
**evidence relation** \(\hat{E}_{c,n}\) is a relation *between symbols* —
a reference edge or a containment edge, whichever the nesting between the
two asks for, oriented from \(c\)'s side to \(n\)'s. Swapping the roles
inverts it, \(\hat{E}_{n,c} = \hat{E}_{c,n}^{-1}\), so each of the two
neighbours reads the same edges in its own direction and there is no
ambiguity about which way a condition points.

**What must be evidenced.** Not every neighbour imposes a condition. The
**constraining neighbours** of \(c\), written \(\mathrm{Cons}(c)\), are

- its parent, if that parent is strong (§9.1);
- each of its strong children;
- each `use()` provider it names.

Weak neighbours (§9.1) are not among them and impose nothing. Every
constraining neighbour is a condition in its own right, and a row
survives only when it has evidence with **every** one of them:

$$N_c \;=\; \Bigl\{\, x \in D(c) \;:\; \forall\, n \in \mathrm{Cons}(c).\;\; \exists\, y \in N_n.\;\; \bigl(\mathrm{sym}(x),\, \mathrm{sym}(y)\bigr) \in \hat{E}_{c,n} \,\Bigr\}$$

Sibling children therefore **conjoin**: `func { "a" ; "b" }` keeps the
functions that call `"a"` *and* `"b"`, and a child that selects nothing
empties its parent, which is what a false conjunct does. Disjunction is
written inside a *single* command instead — in `func { "a" "b" }` the one
child carries two selector branches (§6), and branches disjoin. So the
two spellings are two different operators, and the `;` that conjoins does
so only inside a scope: between top-level statements `;` separates whole
answers, which are unioned into the query's result
([From Result to Cache §2](/docs/design/derivation/#decomposition)),
imposing nothing on each other.

Two properties now follow from the formula instead of having to correct
it. Evidence is **matched per symbol** — only \(\mathrm{sym}(x)\) occurs
on the left, so if any instance of a symbol is evidenced then every
instance of it survives, which is what it means for a selection to be
closed per symbol, and what forces the
[planner](/docs/design/planning/#refinement) to work at symbol level as
well. And a `use()` provider's edge is
**one-way**: the provider appears in its user's \(\mathrm{Cons}\), the
user never in the provider's, so a query carrying a label is covered by
the same equation as one that does not — with the constraint flowing in
one direction only.

**The fixpoint.** Each \(N_c\) is defined through its neighbours'
\(N_n\), which are defined through theirs. Read the system as one
equation on whole *assignments*: a point of the lattice
\(\prod_c \mathcal{P}(D(c))\) — one subset of each denotation, ordered by
inclusion in every component — is carried by the right-hand side above to
another point, and monotonically, since enlarging a neighbour's set can
only add witnesses. A query's answer is the **greatest** fixpoint of that
map. The least one will not do, because every condition is existential:
it demands a witness, so with all neighbours empty no row has one and the
all-empty assignment reproduces itself — a perfectly good fixpoint of any
nested query, answering it with nothing. Greatest says the intended thing
instead: keep every row that no constraint actually rules out. Reaching
it is a descent — start above it and remove what the conditions deny —
and because the denotations are finite the lattice has finite height, so
the descent stops.
[Evaluating the Fixpoint](/docs/design/evaluation) is that descent as an
algorithm.

\(\mathrm{Cons}(c)\) is settled before any of this begins. Weakness has a
fixpoint of its own (§9.1), computed over the shape of the query alone and
never over a selection, so the two are stratified rather than mutually
recursive: which neighbours constrain is decided first, and only then is
the system above solved.

Between the denotation and the selection sits the object the engine
actually asks the database for. A **read** evaluates \(P(c)\) together
with whatever the neighbours can contribute as *conditions* rather than
as finished results, which is enough to bound it on both sides:

$$N_c \;\subseteq\; \text{read} \;\subseteq\; D(c)$$

The left inclusion is what lets the engine begin the descent at the reads
rather than at the denotations: a narrowing iteration lands on the
greatest fixpoint only if it starts above it, and every read is above it.
The right one is why a read is worth caching where a selection is not
([From Result to Cache §5](/docs/design/derivation/#upper-bound)), and
making it tight is the whole business of
[Planning from Measured Cardinality](/docs/design/planning).

## 8. Dependency kinds {#dependency-kinds}

Before a command can produce useful output, some of its neighbours may
have to have produced theirs. Each substatement therefore carries a
list of *dependencies*, of two kinds:

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

The graph these kinds induce, and the order it is walked in, belong to
[Evaluating the Fixpoint](/docs/design/evaluation).

## 9. Weakness and bindness {#weakness-and-bindness}

Two related but distinct properties govern how substatements participate in a
query, at two different levels.

### 9.1. Weakness (command-level, compositional) {#weakness}

**Weakness answers: does this command's selection constrain its neighbours?**
A weak substatement is a display echo — it contributes nodes and edges to the
result graph, but it never *narrows* a neighbour that has already resolved, so
an empty match cannot eliminate its parent or children.
A strong substatement's selection participates in the composition: a parent
survives only if it actually relates to something the strong child selected.
It is exactly the strong neighbours that appear in §7's conjunction.

There is one deliberate half-exception, and it runs the other way: a weak
neighbour may *supply* a selection to a substatement that has none. Supplying
is not narrowing — it is what carries data through a chain of weak
intermediaries, as in `{{"a"}}`, where every scope above the leaf is weak and
only the leaf selects anything. So the rule in full, and the one the engine
applies on **both** notification edges — parent to child and child to parent —
is:

> A weak neighbour may seed a substatement that has nothing; it may never
> narrow one that has already resolved.

The two edges are stated together on purpose. The same weak command means the
same thing whichever side of the braces it is written on, so
`X { data(inherit="false") }` and `data(inherit="false") { X }` both leave `X`
standing when the weak half matches nothing.

A substatement is a *weakness candidate* when its command is
**non-constraining**: every selector is a unit verb or a bare (nameless)
type selector, **and** no verb of any kind constrains a name. A name is a
name wherever it is written — `filter("exact_name", "x")` pins a set as
tightly as `"x"` does, and makes its command constraining even though it
is a filter. Type predicates stay structural, so bare `func` and a
type filter both leave a command a candidate.

The **propagation rule** is then iterated to a fixpoint of its own. A
candidate becomes weak iff:

- its parent is weak (or it is top-level), **or**
- all of its children are weak (vacuously true for a leaf).

The consequence worth internalising: a candidate **sandwiched between a
strong parent and a strong child stays strong**. In

```askl
select func {{ "b" }}
```

the outer command is strong (`select`), the leaf `"b"` is strong, so the bare
middle scope — a candidate — satisfies neither weakening condition and
constrains: only callers of `"b"` that the outer level also relates to
survive. Drop the `select` and the outer command is a bare type selector,
so it becomes a top-level candidate, turns weak, weakness flows down
through the middle, and the same query becomes an echo that shows *every*
caller. Give the outer command a name instead of `select` and it is
strong again, by either route.

### 9.2. Bindness (component-level, outcome) {#bindness}

**Bindness answers: does this component demand instances at all?**
A **component** is one or more statements connected by label
references — the unit at which a query is asked whether it wants
anything. A binding component wants results; a non-binding one is
structure or directive. `preamble project("ucx")` is non-binding — it
configures the query. `{{}}` is non-binding structure. Non-binding
components are silently empty; binding ones must be *satisfiable*:
each needs at least one **anchor**, otherwise the query is rejected
with a hint.

An **anchor** is a verb that can produce instances on its own: a name
pattern, a name filter (`filter("exact_name", …)`,
`filter("compound_name", …)`), `search(...)`, `loc(...)`, a layer
literal, or `select`. Everything else — bare type selectors,
`project(...)`, `preamble` — is a pure constraint, whose denotation is
the whole index and which therefore only ever narrows what an anchor
produced. A component of pure constraints denotes nothing worth
materialising, which is why it is rejected rather than answered.

### 9.3. `select` bridges the two levels {#select-bridges}

`select` is the user-visible verb for both properties, named for the outcome:

- at the **component** level it declares the component binding — and since it carries
  an always-true anchor, a `select`-bearing component is always satisfiable
  (an unanchored command with `select` enumerates everything its filters
  allow, bounded by the result budget);
- at its **command** it implies strength — wanting instances *here* means
  this command's selection participates in the composition, so a
  `select`-carrying command is never a weakness candidate.

Weakness otherwise keeps its defaults, and one invariant links the two
levels: **an anchored command is never weak**. Every anchor either is a
constraining selector or carries a name constraint, so an anchored command
fails the candidate test of §9.1 outright and the propagation rule never
reaches it. That is what keeps the planner and the composition talking
about the same query: only anchored substatements probe
([Planning §3](/docs/design/planning/#anchors)), so every neighbour a
probe resolves is a strong one, and any constraint the planner derives
from it is a constraint §7's conjunction imposes too.

## Where to read more

- [Evaluating the Fixpoint](/docs/design/evaluation) — how §7's fixpoint
  is computed, and why the computation stops.
- [Planning from Measured Cardinality](/docs/design/planning) — anchors,
  capped probes, and how a read is made tight.
- [Partitioning a Materialisation](/docs/design/shards) — the forest
  \(f_c\) is split across, and the theorems the split satisfies.
- [Layer Keys and Hashing](/docs/design/layer-keys) — how \(F_c\) and
  the per-verb inputs become the cache keys \(H(c)\).
