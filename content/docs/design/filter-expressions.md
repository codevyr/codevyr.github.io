---
title: "Filter Expressions and the Slot Cascade"
description: "Boolean operators inside one dimension, record merge across dimensions, and CSS-like inheritance into scopes"
weight: 107
---

**Status: adopted design, not yet implemented.** This page fixes the
semantics of the boolean operators (`or`, `and`, `not`, parentheses)
before any grammar work lands, together with the inheritance law they
force us to make explicit. It extends
[Queries and their Meaning](/docs/design/semantics) — in particular the
per-tag override/accumulate fold of
[§2](/docs/design/semantics/#assembling-the-command) — from a table of
per-verb behaviours into one general law.

## 1. The problem {#the-problem}

Today the language has conjunction by juxtaposition (filters in one
statement), disjunction only between juxtaposed selectors, and negation
only as `ignore(...)`. Three things are inexpressible or awkward:

- *disjunction over filters*: "bar in project a **or** project b",
  "foo that is a func **or** a method";
- *general negation*: "everything except the test project";
- *a stated inheritance rule*: `func "foo" { "bar" }` makes `bar` a
  func, `func { method "bar" }` makes `bar` a method — behaviour that
  exists in the implementation as per-verb special cases
  (`DeriveMethod`, per-tag override) but is documented nowhere as a
  single law.

Adding operators naively reopens every one of these as a question:
what does `not has` mean, what does an inherited `or`-group do when
the child constrains the same thing, what may appear inside
parentheses at all. The answers below are chosen so that each is a
one-sentence rule a reader can apply without simulating the engine.

## 2. A statement is a record of aspects {#aspect-record}

[Semantics §2](/docs/design/semantics/#assembling-the-command) already
observes that a verb can carry several aspects at once. This design
promotes that observation to the organising principle: **a statement
is a record with one field per aspect, and juxtaposition is record
merge**, not conjunction. The aspects:

| Aspect | Verbs | Merge law | Inheritance |
|---|---|---|---|
| **predicate** | filters and anchors | this page | this page |
| **relationship mode** | `has`, `refs`, `derive`, `unnest` | override | own channel, as today |
| **bindings** | `@x`, `@@x`, `#x`, `label` | accumulate | opt-in (`@@`) |
| **execution environment** | `preamble`, `layer`, ephemeral verbs | own rules | own rules |

The predicate is the only truth-valued aspect, and **boolean operators
exist only inside it**. `not has` is a *type* error, not a syntax
restriction — the same clause/expression distinction SQL draws between
`WHERE` (boolean-composable) and `ORDER BY` (not), rendered on askl's
flat surface. The cautionary precedent is find(1), which has the same
flat surface of juxtaposed primaries — options, tests, actions — and
chose to let *actions* participate in its `-o`/`-a` expressions as
side-effecting always-true primaries. The result is the classic
`find . -name '*.c' -o -name '*.h' -print` trap (prints only the `.h`
files). askl keeps non-predicate verbs out of the algebra entirely.

A few verbs contribute to two aspects: `file` and `dir` are predicates
*and* switch the scope's relationship context to containment. Inside a
compound expression, only the predicate contribution of a verb is
read; its other aspects are suppressed. `file or dir { ... }` leaves
the scope on the default `REFS|HAS` — write `has` explicitly if
containment is meant. The alternative ("apply the side effect when all
disjuncts agree") is exactly the kind of condition a reader cannot
check locally, and is rejected.

## 3. The predicate record: dimensions and slots {#dimensions-and-slots}

The command predicate keeps its shape from
[semantics §6](/docs/design/semantics/#the-commands-predicate):

$$P(c) \;=\; F_c \,\wedge\, (b_1 \vee \dots \vee b_r)$$

What changes is the structure of \(F_c\). It is a **record from
dimensions to slots**, where a dimension is a verb tag — symbol type,
project, name exclusion, filter kind — and a slot's value is a boolean
expression **over that one dimension only**:

```text
func or method                  // type slot
project("a") or project("b")    // project slot
g"foo*" and not g"*_test"       // name-filter slot
not (func or method)            // type slot, negated group
```

A slot is to \(F_c\) what a property value is to a CSS rule: internally
structured, externally atomic. Across slots the only combinator is
juxtaposition — AND by the record law — so \(F_c\) is a record, never a
formula.

**Cross-dimension compounds.** An expression that mixes dimensions and
contains no anchor — `(project("a") and func) or (project("b") and
method)` — is a builder error: *cross-dimension filter; use
juxtaposition, or split into siblings*. Per-project type scoping is
written as siblings:

```text
project("a") func { ... };
project("b") method { ... }
```

This restriction is what keeps the inheritance law below one sentence
long: the case that would need a clever merge rule is unrepresentable.
Cross-dimension compounds that *do* contain anchors — `(func and
"foo") or (method and "bar")` — are legal, because they are branches
\(b_i\), and branches never inherit, so the inheritance law never
meets them.

**Anchor rule.** An expression is anchor-capable iff it contains an
anchor-capable atom **not under `not`**. Bare type selectors are dual
(anchor-capable filters), so `func and not g"test_*"` is a valid
anchored statement: all functions except those matching `test_*`. A
negation alone anchors nothing.

**`or` versus juxtaposition between anchors.** Both disjoin, but they
are observably different, and deliberately so: an all-anchor
`or`-group forms **one** branch with one no-match probe for the whole
disjunction, while juxtaposed anchors remain separate branches with
per-branch probes. `"a" or "b"` says *either is fine, don't warn me*;
`"a" "b"` says *each of these should exist*.

## 4. Inheritance: the slot cascade {#slot-cascade}

The law, in one sentence:

> A child statement that mentions dimension D replaces the inherited
> D-slot wholesale; dimensions it does not mention flow through;
> exclusions accumulate.

- `func "foo" { "bar" }` — the child does not mention the type
  dimension; `func` flows through; `bar` is a func.
- `func { method "bar" }` — the child mentions the type dimension;
  `method` replaces `func` wholesale; `bar` is a method. Silently: the
  replacement is the *intended common idiom*, not an anomaly to warn
  about.
- `project("a") or project("b") { "bar" }` — the group is one project
  slot and inherits as a unit; `bar` is in either project.
- `project("a") { not project("test") ... }` — negations
  (**exclusions**) do not replace, they *accumulate*: the child means
  \(a \wedge \neg test\). Negations narrow; this matches today's
  `ignore`, and gitignore's `!`-re-include rules are the standing
  warning against making negation cleverer than that.

Which slots flow down at all remains the per-verb `DeriveMethod`
policy: `project` always, bare type selectors by default,
`filter(...)` by opt-in, anchors never — anchors are *this
statement's* entry points, not context. A compound slot value inherits
iff all its atoms do.

This cascade is not new behaviour. It is exactly what the `VerbTag`
override already implements — `project("a") project("b")` keeps `b`, a
child's type verb removes the inherited one — codified as the single
law and extended over slot *expressions* rather than single verbs.

Two consequences worth stating plainly. First, replacement means a
child can *escape* its parent's scope: `project("a") { project("b")
... }` selects in `b`, not in \(a \cap b\) and not nothing. That is
the CSS reading of override and the existing askl reading; narrowing
is written by narrowing the slot's own value. Second, because
inheritable filters are single-dimension by construction, the
distributive product of an inherited disjunction against the child's
filters — \((I_1 + I_2)(C_1 + C_2) = \sum_i\sum_j I_i C_j\) — holds with
no same-dimension conflicts possible: surviving inherited slots and
child slots never share a dimension. The DNF form is a proof device
only; the engine compiles expression trees directly (below) and never
materialises a normal form.

## 5. Dimension resets fall out of the cascade {#dimension-resets}

`any` is already [the type verb that constrains no
type](/docs/syntax/#any-any-symbol-type) — the same family as `func`, with the same
two forms (`any` filters, `any("foo")` selects a name across all
types). Under the slot cascade this needs no reset semantics at all:
writing `any` writes an unconstrained value into the type slot, and
**replacement does the cancelling** — an inherited `func` is undone by
an ordinary slot write, not by a bespoke operator. What used to be a
modifier special case is now a corollary of the general law.

The pattern generalises per dimension: any filter family may expose
its unconstrained member — a future argument-less `project()` would
unconstrain the project slot the same way. This is CSS per-property
`unset`, spelled as an ordinary value.

Inside compound expressions, `any` is rejected by the concreteness
gate of §10 — it compiles to *no constraint*, which the expression
compiler must never meet under `not` or `or` — with a span error
pointing at the juxtaposed spelling (`any "bar"`), which means the
same thing.

Two things the cascade deliberately cannot express: cancelling an
accumulated exclusion (`not ...` only accumulates), and a whole-record
"start fresh" (CSS `all: unset`). Both remain future work; if either
becomes pressing, a dedicated reset verb is the answer, and the
never-implemented `scope(isolated=)` stub should be removed or
finished as part of that decision.

## 6. `not` everywhere; `ignore` retires {#not-and-ignore}

`not` applies to any predicate atom or parenthesised predicate group:

- `not "foo"` ≡ `ignore("foo")`
- `not (g"mlx5_*" and package("drivers/net"))` ≡
  `ignore("mlx5_*", package="drivers/net")` — this requires promoting
  the package test to a first-class positive filter `package(...)`
  (today it exists only inside `ignore`), which is a uniformity win on
  its own.

`ignore` remains as a deprecated alias.

**Closed-world honesty.** Negation is evaluated per dimension against
everything indexed: `not func` matches methods — and macros, files,
directories, every symbol whose type is not func. `not project("a")`
includes symbols belonging to no project at all. The reference
documentation states this in one explicit paragraph rather than
leaving it to be discovered.

**The `!`/`not` confusable.** `!"foo"` (*force* this selector) and
`not "foo"` (*exclude* this name) will sit in the same lexical
position while meaning nearly opposite things, and every mainstream
language reads `!` as negation. `!` keeps working now; the adopted
direction is to deprecate the `!` sugar in favour of the existing
`forced(...)` spelling once `not` lands, so that the only negation-
shaped token in the language is negation.

## 7. Surface syntax {#surface-syntax}

The grammar owns **shapes**; the builder owns **classes**. Verb names
stay ordinary identifiers resolved by the dispatch table — no per-verb
keywords in the grammar — mirroring the typed-string precedent: the
grammar admits, the parser types, and errors still surface during
parsing with proper spans ("`has` is a relationship verb and cannot
appear in a filter expression").

```pest
// or/and/not reserved; guards so idents like `order` don't split.
or_kw  = @{ "or"  ~ !XID_CONTINUE }
and_kw = @{ "and" ~ !XID_CONTINUE }
not_kw = @{ "not" ~ !XID_CONTINUE }
atom            = _{ ("(" ~ compound_filter ~ ")") | verb }   // verb = today's rule
compound_filter = { not_kw* ~ atom ~ ((or_kw | and_kw) ~ not_kw* ~ atom)* }
statement       = { (compound_filter* ~ scope) | compound_filter+ | scope }
```

Precedence `not` > `and` > `or` lives in a Pratt-parser table on the
flat rule, not in layered grammar rules (which parse the same language
through deep single-child trees; a PEG alternation
`or_expr | and_expr | not_expr` is dead code outright, since ordered
choice makes the first alternative subsume the rest).

**Reading rule: operators bind tighter than whitespace.** This inverts
find(1) and regex intuition, where implicit conjunction binds tighter
than the disjunction operator — but it is forced, because juxtaposition
is record merge, not an operator, so it cannot have a binding strength
inside expressions. The inversion is benign: outcomes coincide with
either reading. `"a" or "b" "c"` is \(a \vee b \vee c\) whichever way
it is grouped (anchors disjoin), and `func or method project("x")` is
\((func \vee method) \wedge x\) under both intuitions.

**Whitespace footgun.** The grammar's implicit whitespace makes
`func ("foo")` an argument list today, and inside expressions the
argument reading stays greedy: `method or func ("foo")` parses as
\(method \vee func(\text{"foo"})\), not
\((method \vee func) \wedge \text{"foo"}\). Kept greedy for
compatibility; documented; a lint on spaced argument parentheses is a
possible later nicety.

A `compound_filter` that is a single verb with no operators is exactly
today's verb, of any class — nothing about existing programs changes.
Compound expressions admit predicate atoms only, and two builder
checks apply on top of classification: an anchor-free compound must be
single-dimension (§3), and every atom in a compound must compile to a
concrete filter (§9).

## 8. Worked examples {#worked-examples}

```text
func or method { "foo" }              // foo, of either type, inherited into the scope
project("a") or project("b") { "bar" }// bar in either project
project("a") "foo" { "bar" }          // bar still in project a (anchors never inherit)
func "foo" { "bar" }                  // bar is a func (type slot flows through)
func { method "bar" }                 // bar is a method (child slot replaces, silently)
project("a") { not project("test") }  // a AND NOT test (exclusions accumulate)
func and not g"test_*"                // anchored: all functions except test_*
(func and "foo") or (method and "bar")// one anchor branch; never inherits
(project("a") and func) or (project("b") and method)
                                      // builder error: anchor-free cross-dimension
                                      // filter → write as two siblings
any "bar"                             // bar of any type (type slot reset; other slots inherit)
```

## 9. What the state of the art says {#state-of-the-art}

Systems that face *hierarchical scope + typed filters + conflict on
the same key* cluster into four families:

| Family | Examples | Verdict for askl |
|---|---|---|
| refinement-only | XPath steps, jq, Kusto pipelines | pure conjunction; cannot express the `func { method }` idiom |
| **slot cascade** | CSS, editorconfig, `RUST_LOG` targets, tsconfig `extends` | whole-key replacement, most-specific wins — simple and massively adopted |
| ordered rules | gitignore, iptables | powerful, notoriously unpredictable at scale |
| full logic | CodeQL, Datalog, Semgrep | maximally expressive, the opposite of concise |

askl takes the slot cascade; editorconfig — sections scoped by glob,
properties inherited from outer sections, overridden whole-property by
inner ones — is nearly isomorphic to askl scoping and confuses nobody.

The negative results shaped the design as much as the positive ones.
tsconfig and pre-flat ESLint grew per-key special merge rules (array
concat here, deep merge there), and that unpredictability is a chronic
complaint — ESLint's flat-config rewrite abandoned cascade merging
outright; this is why a literal-substitution merge law considered for
compound slots was dropped in favour of making cross-dimension filters
unrepresentable. gitignore's `!`-re-include semantics are why
exclusions only accumulate. find(1)'s actions-in-expressions are why
non-predicate verbs stay out of the algebra. And no surveyed system
resolves conflicts by satisfiability analysis ("these two can never
both hold, so drop one") — conflict resolution is everywhere keyed
*syntactically*, which is what makes it predictable, so askl's cascade
keys on the dimension, never on whether a conjunction happens to be
empty.

## 10. Implementation notes {#implementation-notes}

The IR already speaks boolean: `CompositeFilter` carries And/Or/Not
nodes, and the operators compile one-to-one onto its constructors.
Composition centralises into two functions — an expression compiler
(atom → the verb's own leaf filter; And/Or/Not → the corresponding
node) and the record compiler
\(\bigwedge(\text{slots sorted by tag}, \text{exclusions})\), with
branches fused on top exactly as
[today's fuse path](/docs/design/semantics/#the-commands-predicate)
already ORs selector predicates. Verbs keep compiling their own
leaves; what they lose is the pairwise merge protocol — replacement
lives in the slot record, so verbs stop knowing about each other.
Sorted slot iteration keeps the
[filter hash](/docs/design/layer-keys/#filter-hash) deterministic.

Two invariants make the compilation total:

- **No `false`, by construction.** In the IR, an absent filter means
  *no constraint*, disjunction widens to it, and negation of it is
  itself — there is no representation of *matches nothing*. Sound
  today only because `ignore` never negates an absent filter. Under
  general `not`, the rule is: every atom admitted into a compound
  expression must compile to a concrete filter (checked at
  classification), so `Not` and `Or` never meet the absent case inside
  an expression; a top-level slot compiling to nothing keeps its
  "unconstrained" meaning and is dropped by the conjunction.
- **Fusable atoms only.** Anchors whose emission is not exactly *one
  query over the fused predicate* — `search(...)` with its limits and
  side effects, `loc(...)` — cannot appear inside compound
  expressions: `search("x") or "foo"` has no single-query semantics.
  The existing per-selector fusability contract becomes the admission
  test, lifted to branch level: a branch is fusable iff all its anchor
  atoms are.
