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
([layer-keys §4](/docs/design/layer-keys/#command-hash)).

A verb can carry **several aspects at once**, contributing each to a
different slot of \(c\): `search` both *populates* and
*filters* (its match predicate constrains the selection); `project`
only filters; the aspects a verb lacks are no-ops in the fold. The rest
of this page combines the surviving contributions, one aspect at a
time.

## 3. Filters compose {#filters-compose}

Each filter \(g\) of the assembled command induces a **selection**
\(\sigma_g\): the function keeping exactly the rows that satisfy
\(g\). Selections compose as functions, and composing them is the same
as conjoining their predicates:

$$\sigma_{g_1} \circ \sigma_{g_2} \;=\; \sigma_{g_1 \wedge g_2} \;=\; \sigma_{g_2} \circ \sigma_{g_1}$$

— order never matters, which is why every page writes one combined
predicate \(F_c = \bigwedge_j g_j\) rather than a composition chain.
Nothing here is conjunction-specific: \(\sigma\) is defined for any
Boolean predicate tree — \(F_c\) itself carries And/Or/Not nodes, and
§6 puts a disjunction of selector branches on top of it — and any such
tree normalises to a disjunction of conjunctions. Conjunction is merely
the law by which *separate* filters combine.

\(F_c\) and \(\sigma_{F_c}\) carry the same information in different
categories — the predicate is the syntactic object cache keys hash
([layer-keys §2](/docs/design/layer-keys/#filter-hash)), the selection
is the function evaluation applies — and the map between them is
many-to-one, since equivalent but structurally different trees induce
the same selection while hashing apart (harmless: at worst a redundant
materialisation).

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
exactly when each populate \(u_i\) is. What that additivity buys,
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
command \(c\)'s selection. For every **constraining** neighbour \(n\)
of \(c\) — its parent, and each of its children — a row survives only
with *evidence* linking it to that neighbour's selection:

$$N_c \;=\; \{\, x \in D(c) \;:\; \forall n.\; \exists\, y \in N_n.\; (x, y) \in E_{\mathrm{rel}} \,\}$$

\(E_{\mathrm{rel}}\) is the **evidence relation** — a reference edge or
a containment edge, whichever the nesting asks for — matched **per
symbol**: if any instance of a symbol is evidenced, every instance of
that symbol survives. Weak neighbours (§9) impose no condition and drop
out of the conjunction.

Each \(N_c\) is defined through its neighbours' \(N_n\), which are
defined through theirs. The system is mutually recursive, and its
answer is the **fixpoint**: start from the denotations and narrow, a
command at a time, until nothing changes. That every step only removes
rows is what makes the fixpoint reachable, and
[Evaluating the Fixpoint](/docs/design/evaluation) is how it is
reached.

Between the denotation and the selection sits the object the engine
actually asks the database for. A **read** evaluates \(P(c)\) together
with whatever the neighbours can contribute as *conditions* rather than
as finished results, which is enough to bound it on both sides:

$$N_c \;\subseteq\; \text{read} \;\subseteq\; D(c)$$

The upper bound is why a read is worth caching where a selection is not
([From Result to Cache §5](/docs/design/derivation/#upper-bound)), and
making it tight is the whole business of
[Cost-Based Execution](/docs/design/cost-based-execution).

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
result graph, but an empty match does not eliminate its parent or children.
A strong substatement's selection participates in the composition: a parent
survives only if it actually relates to something the strong child selected.
It is exactly the strong neighbours that appear in §7's conjunction.

A substatement is a *weakness candidate* when its command is **non-constraining**:
every selector is a unit verb or a bare (nameless) type selector. Filters
alone do not make a command constraining — `filter("compound_name", "x")` on
an otherwise bare substatement leaves it a candidate.

The **propagation rule** is then iterated to a fixpoint of its own. A
candidate becomes weak iff:

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

Weakness otherwise keeps its defaults: anchored commands are constraining
by construction, and the propagation rule above decides the rest.

## Where to read more

- [Evaluating the Fixpoint](/docs/design/evaluation) — how §7's fixpoint
  is computed, and why the computation stops.
- [Cost-Based Execution](/docs/design/cost-based-execution) — using
  anchors and measured cardinality to make a read tight.
- [Partitioning a Materialisation](/docs/design/shards) — the forest
  \(f_c\) is split across, and the theorems the split satisfies.
- [Layer Keys and Hashing](/docs/design/layer-keys) — how \(F_c\) and
  the per-verb inputs become the cache keys \(H(c)\).
