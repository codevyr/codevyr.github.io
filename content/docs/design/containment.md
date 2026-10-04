---
title: "Containment"
description: "Why source ranges cannot express what contains what, and the DFS interval, nesting level and fold depth that replace them"
weight: 100
---

Every query that nests — `mod("amdgpu") { func }`, `func("main") { "printk" }`,
`{ "inner" }` — is asking a containment question, and the answer has to come
from data the indexer wrote down. This page is about what that data is and why
the obvious choice does not work.

The short version: **containment is a position in a depth-first traversal, not
a source range.** A symbol owns a half-open interval `[dfs_enter, dfs_exit)`,
and `Y` is inside `X` exactly when

```
X.dfs_enter < Y.dfs_enter < X.dfs_exit
```

with both in the same object. Two further columns, `nesting_level` and
`fold_depth`, say *how far* inside and *through what kind of boundary*.

## 1. Why ranges are not enough {#why-not-ranges}

The natural encoding is the one the source already has: a symbol spans bytes
`[start, end)`, and `Y` is inside `X` when `X`'s range contains `Y`'s. That is
what askl used, and it is wrong in C for two independent reasons.

**A macro expansion cannot contain what it generates.** The indexer used to
record an expansion clamped to the macro *name* — measured on the deployed
index, **5,439 of 5,439** sampled expansions had width exactly the name's
length, so an expansion could never own anything. The clamp bought one real
thing: in `printk(foo)` the argument `foo` came out a *sibling* of `printk`
rather than something it contains, which is right, because the caller wrote it.

But that was range-based parentage, and ranges no longer decide parentage. The
clamp is gone: an expansion now records the whole invocation, which is the span
a reader wants, and the argument stays the caller's because of a rule about
*ownership* rather than a narrowed span — see [ephemeral rows](#ephemeral).
Two questions, two mechanisms, instead of one span forced to answer both.

**Generated symbols share one extent.** At `arch/x86/events/amd/core.c:1067`,
`PMU_FORMAT_ATTR(event, "config:0-7,32-35")` generates four symbols:

| instance | kind | range | width |
|---|---|---|---|
| `PMU_FORMAT_ATTR` | expansion | `[27763,27778)` | 15 |
| `format_attr_event` | definition | `[27763,27805)` | 42 |
| `event_show` | definition | `[27763,27805)` | 42 |
| `struct (unnamed …:1067:1)` | definition | `[27763,27805)` | 42 |
| `__compiletime_assert_1121` | declaration | `[27763,27805)` | 42 |

Three of them are byte-for-byte identical, and one of them —
`__compiletime_assert_1121`, an internal assertion helper — genuinely lives
*inside* `event_show`. No comparison of those ranges can say so. The visible
symptom was `mod("amdgpu") { func }` returning compiler-internal assert
helpers alongside the module's real functions.

Switching to spelling locations does not help; it fails on the opposite axis.
Expansion locations distinguish the invocations and collapse the generated
symbols; spelling locations distinguish the generated symbols and collapse the
invocations — and scatter the generated symbols across whatever macro
definition textually produced them, which for `__compiletime_assert_1121` is a
different macro entirely. A `##`-pasted identifier has no file location at all:
its name lives in clang's virtual `<scratch space>` buffer.

So the tree cannot be *derived*. It has to be *stated*.

## 2. Stating what a range cannot say {#stated}

Mostly the ranges do the work: a declaration sits inside the text of whatever
declares it, so an indexer can sort by extent and read the tree off the result.
The cases above are where that fails in C. Go has a smaller version of the same
problem:

```go
var First, Second = func() int { … }, func() int { … }
```

One range covers both names, and `First`'s identifier is not contiguous with
`First`'s own initialiser — so **no extent assignment** can say which closure
belongs to which variable. The parser knows; nothing else does.

So a producer may *state* an instance's parent instead of leaving it to the
ranges. The consequence is worth stating plainly, because it looks like a bug
the first time you see it: **a child's source range may lie entirely outside its
parent's.** `First` occupies five bytes and owns a closure thirty bytes away.
That is the encoding working as intended — the interval decides containment and
never consults a range.

## 3. The three axes {#axes}

| axis | columns | question it answers |
|---|---|---|
| position | `dfs_enter`, `dfs_exit` | is Y inside X at all? |
| depth | `nesting_level` | how many levels down? |
| provenance | `folded`, `fold_depth` | across how many *generated* boundaries? |

### Position

The indexer already performs a depth-first walk of the AST, so entry and exit
numbers are two counters. Siblings get **disjoint intervals even when their
source ranges are identical**, which is the only thing that orders the
`PMU_FORMAT_ATTR` cluster:

```
enter exit depth  node
  1    10    0    PMU_FORMAT_ATTR
  2     5    1      event_show
  3     4    2        __compiletime_assert_1121
  6     7    1      format_attr_event
  8     9    1      unnamed struct
```

`event_show` `[2,5)` contains the assert; `format_attr_event` `[6,7)` does not.

Numbering is **sparse** — persistent instances are spaced one stride (1024)
apart — so a query-time ephemeral row (a `search()` match, say) can be slotted
between two neighbours without renumbering anything. `dfs_enter` and `dfs_exit`
are `BIGINT`: the largest real object is 194,446 instances, which at one stride
each would use 18.5% of a 32-bit range, and that is too little headroom for a
number that only grows.

Containment never crosses objects, so every predicate is scoped by
`object_id` and every read is one btree range scan on `(object_id, dfs_enter)`.
No recursion, no GiST, no anti-join.

### Depth

`nesting_level` counts structural levels from the object's root, and the
containers above a declaration are real levels: a file object carries an
instance for its module and one for its directory as well as for itself, all
covering the whole file. So a function is three levels down, not one — which is
why `X { Y }` cannot mean "depth + 1" (see [modes](#modes)).

**Which container is outermost is the indexer's choice, not a fact.** A module
spans several directories while a directory can hold several modules, so the two
genuinely overlap and each language may answer differently — the C indexer puts
modules outermost, the Go indexer does too, and nothing would break if one
changed. No query depends on it: traversal is the interval plus at most one
equality, and never names a container type.

### Provenance

`folded` marks an instance that exists only because of a macro expansion.
`fold_depth` counts the folded ancestors above it. These are what separate
*where a symbol sits* from *which levels are interesting*, and they are why
`mod("amdgpu") { func }` returns the module's ~900 real functions rather than
either nothing (strict depth) or every compiler-internal helper (plain
transitive descent):

| instance | folded | fold_depth | in `mod { func }`? |
|---|---|---|---|
| a file | no | 0 | — |
| `event_show`, generated, whose ancestors are not | yes | 0 | **yes** — a real function of the module |
| `__compiletime_assert_1121`, inside generated `event_show` | yes | 1 | **no** |
| the expanded `foo` inside `printk(foo)` | — | 1 | **no** |
| the argument `foo`, owned by the caller | — | 0 | **yes** |

## 4. The three traversal modes {#modes}

Every mode is the interval range plus at most one equality. See
[Traversal modes](/docs/syntax/#traversal-modes) for the surface syntax.

| mode | extra predicate | reads |
|---|---|---|
| `nested` | `Y.nesting_level = X.nesting_level + 1` | `nesting_level` |
| default | `Y.fold_depth = X.fold_depth + (X.folded ? 1 : 0)` | `fold_depth` |
| `unnest` | — | nothing |

The default is *unfolded*, not *direct*. Direct would make
`mod("amdgpu") { func }` empty, since a module's direct children are files.
Unfolded passes through the structural levels and stops at a macro boundary —
and the `+ (X.folded ? 1 : 0)` term is exactly what lets
`mod { macro { func } }` step one level past a fold on purpose.

Two consequences worth stating, both accepted:

- `X { Y }` no longer means strictly "direct". In Go, `func { func }` returns
  closures nested at any depth, and a reference inside a local declaration
  counts as the enclosing function's.
- `nested` is the only reader of `nesting_level`, and the only way to ask the
  question the other two modes cannot: *what does this expansion produce,
  immediately*.

## 5. Ephemeral rows {#ephemeral}

Verbs like [`search()`](/docs/design/search/) materialise instances at query
time into an existing object. Those rows need a position too, or they fall
outside every interval and become invisible.

Two rules place them.

**An ephemeral instance is a leaf.** It gets a width-1 interval, one integer
wide, because nothing is ever placed inside one — a reference sits below every
slot, and an ephemeral row may not own another. That makes laminarity
*automatic*: width-1 integer intervals cannot partially overlap, so two rows
under one owner, from whatever batches, are always siblings and never nest by
accident. The gap holds 1022 of them, which is more than a query may return, so
exhausting it is unreachable rather than a caveat.

**Its owner is the innermost *unfolded, persistent* instance whose extent
contains it.** Each word is doing work:

- *persistent* — placement is a function of committed data alone, so a
  **cached** layer's numbering cannot depend on which query populated it first;
- *unfolded* — **a macro expansion owns no text.** Every byte at the call site
  of `foo(a)` was typed by the caller, so the owner of a match anywhere inside
  it is the enclosing function. An expansion owns the symbols it *generates*,
  through the interval, and nothing else;
- *innermost* — narrowest extent, then **deepest in the tree**, with
  `dfs_enter` only as a final tiebreak. The same rule both indexers use to place
  a persistent reference, so one byte range gets one owner whether the row is
  stored or materialised at query time.

  Deepest matters because of the container chain: a match outside every
  declaration — an `import` block, a comment between functions — has the module,
  the directory and the file as candidates, all co-extensive. Taking the
  outermost numbers it *below its own file's interval*, out of reach of
  `file(…) { … }` in every traversal mode, since every mode starts from the
  interval and only adds an equality. Keeping compiler helpers from owning user
  text is the *unfolded* rule above, not this one.

The row then takes its depth and fold depth from that owner, so it behaves
exactly as a symbol at that position would.

Two accepted limitations:

- `search("a") { search("b") }` returns nothing rather than the nested reading.
  The two searches materialise into separate layers, and letting one site the
  other is what would make a cached layer depend on its neighbour. The fix is
  sequencing — materialise the outer first and search only inside its matches —
  not numbering. `"foo" { search("b") }`, the common query, works, because the
  container is persistent.
- `macro("foo") { search(...) }` finds nothing, since none of that text is the
  macro's. The arguments are searchable from the enclosing function, which is
  where they were written.

## 6. Validation {#validation}

The engine reads only the *child's* `dfs_enter`, which means a malformed tree
produces plausible wrong answers instead of errors. So uploads are checked, per
object, before anything is stored:

- every interval is non-empty (`dfs_exit > dfs_enter`);
- every interval is at least one **stride** (1024) wide — the gap query-time
  rows are numbered into ([ephemeral rows](#ephemeral)), which numbering by
  stride provides for free. A merely *narrow* owner loses its own search matches
  just as silently as an empty one;
- `dfs_enter` is unique — the DFS order is total;
- intervals nest, never straddle;
- `nesting_level` equals the actual depth;
- `fold_depth` is the parent's plus its `folded` flag;
- every ref's `dfs_enter` falls strictly inside some instance interval.

A note on that last one, because the cheap version of it is wrong: a ref may
legitimately sit after a nested instance has closed while an ancestor is still
open, so it must be tested against the top-level intervals, not against the
nearest instance below it.

An index produced before the containment tree existed has every interval at
`[0,0)` and fails the first check, so this doubles as the guard against an
unnumbered project being served as a silently empty graph.
