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
length, so an expansion could never own anything. Widening it to the whole
invocation does not help either: then it contains the *arguments*, and the
caller wrote those.

Neither span is wrong, because the record is. **Using a macro is a reference,
not an instance.** A macro is declared once, at its `#define`, and every use is
one site naming it — which is what a reference is. So there is no expansion
instance to size and no question of what one contains.

Deleting them removed **37.8% of criu's instances** (54,829 → 34,090) and left
the reference set byte-for-byte identical, which is what proves the two records
were saying the same thing. `func { macro }` and `mod("amdgpu") { macro }` keep
working through the reference, exactly as `func("main") { "printk" }` does —
`printk` has no instance in `main` either.

**Generated symbols share one extent.** At `arch/x86/events/amd/core.c:1067`,
`PMU_FORMAT_ATTR(event, "config:0-7,32-35")` generates four symbols:

| instance | kind | range | width |
|---|---|---|---|
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

## 2b. One file, many translation units {#union}

A `.c` file is compiled once, but a header is compiled once per file that
includes it — and under C's preprocessor each of those compilations can see
*different declarations*. glibc's `stddef.h` is built for exactly that: you
`#define __need_size_t` before including it and get only `size_t`.

So the index takes the **union** of every translation unit's view of a file.
Identical declarations from two compilations are one declaration — the indexer
merges a file's parses on symbol, kind and extent, so parsing the same text
again adds nothing, while a declaration only some compilations saw is added.
Once numbered, though, the engine identifies an instance by its *position*:
`(object, dfs_enter, layer)` is the unique key, and the extent is what the UI
highlights, not identity. So two declarations at one extent that the merge
keeps apart — one `DECL(alpha)` line included twice, declaring `alpha_f` and
`alpha_d` — stay two rows with two positions.

The alternative, keeping whichever compilation reached the file first, loses by
arrival order, and it loses *silently* in a specific way: the symbol table is
project-wide and accumulates across all of them, so the name survives while its
only instance does not. That is a symbol a query can name and never locate.
Measured on criu before the fix, reversing the order of 211 translation units
changed **which 21 symbols were locatable at all**.

Two consequences worth stating, both accepted:

- **Both arms of an `#ifdef` co-exist**, and nothing records which build each
  belongs to. The index describes the *text*, not one configuration. This is
  not a new ambiguity — the symbol table always worked this way — it makes the
  two halves agree.
- **A name can be two kinds of thing.** criu compiles one file twice, the
  second time with `-Dvdso_fill_symtable=vdso_fill_symtable_compat`, so
  `vdso_fill_symtable` is a function in one compilation and a macro in the
  other. Both are true of the source, and both are in the index.

A symbol with no instance at all is still possible and still correct: a macro
the compiler defines (`__GNUC__`, `__has_include`) or one that came from a `-D`
flag has no declaration in the tree to point at. On criu that is 51 of 136 such
symbols. Naming them is useful — they are referenced — so they stay.

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
  1    10    0    the file
  2     5    1      event_show
  3     4    2        __compiletime_assert_1121
  6     7    1      format_attr_event
  8     9    1      unnamed struct
```

`event_show` `[2,5)` contains the assert; `format_attr_event` `[6,7)` does not.
The macro itself is not in this tree: it is the *reference* that put these four
here, and it lives at its own `#define`.

Real numbers, from glibc's `__SOCKADDR_ALLTYPES` as indexed in criu — thirteen
union members generated at one extent, inside an ordinary unfolded definition:

```
 enter   exit  depth  node                                 range
 25600  53248    2    __SOCKADDR_ARG                       [2819,2878)
 26624  27648    3      __SOCKADDR_ARG.__sockaddr__        [2835,2854)
 28672  29696    3      __SOCKADDR_ARG.__sockaddr_at__     [2835,2854)
   …                      … eleven more, byte-for-byte the same range
 51200  52224    3      __SOCKADDR_ARG.__sockaddr_x25__    [2835,2854)
```

Thirteen identical ranges, thirteen disjoint intervals. No comparison of those
ranges could have ordered them, and none has to.

Numbering is **sparse** — persistent instances are spaced one stride (1024)
apart — so a query-time ephemeral row (a `search()` match, say) can be slotted
between two neighbours without renumbering anything. `dfs_enter` and `dfs_exit`
are `BIGINT`: the largest real object is 194,446 instances, which at one stride
each would use 18.5% of a 32-bit range, and that is too little headroom for a
number that only grows.

Containment never crosses objects, so every predicate is scoped by
`object_id` and every mode predicate is one btree range scan on
`(object_id, dfs_enter)`. No recursion, no GiST. The one anti-join left is on
the reference path: when a scope is answered through references in `nested`
or the default mode, a `NOT EXISTS` over the same DFS numbers decides which
container a reference belongs to, so that a narrower container does not
shadow its parent's — it reads intervals, never extents.

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
| the argument `foo` in `printk(foo)`, owned by the caller | — | 0 | **yes** |

Folding answers *what did this macro bring into existence*, and nothing else.
What a macro **calls** is not a fold question: references are **layerwise**, one
edge per `#define`, so `handler -> DEV_PRINT -> printk` is a chain of ordinary
edges and `handler -> printk` is a traversal of it. The producer records the
body reference where the macro author wrote it and an invocation reference at
each call site; neither needs a fold level, and a reference's `fold_depth` is
non-zero now only when it sits inside macro-*generated* code.

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

`unnest` lost its main reason to exist, and that is the point. It was the only
way to reach what a macro called, because the edge `handler -> DEV_PRINT` did
not exist and containment was the only route across. It does: `unnest` is now
for ordinary containment that runs deeper than a fold rule allows, not for
macros.

Two consequences worth stating, both accepted:

- `X { Y }` no longer means strictly "direct". In Go, `func { func }` returns
  closures nested at any depth, and a reference inside a local declaration
  counts as the enclosing function's.
- `nested` is the only reader of `nesting_level`, and the only way to ask for
  exactly one structural level: a file's own declarations, or the single
  container a symbol literally sits in. It is **not** a window onto what a
  macro generated — a generated declaration is placed where it was written, so
  on the deployed ucx index `macro("ucs_offsetof") { nested }` returns only
  `ucs_offsetof`, and `macro("__KHASH_IMPL") { nested }` returns the six macros
  its body calls.

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
accident. The gap holds 1022 of them per owner — above the default read cap
(1000) and `search()`'s default `limit` (500 per project), so a query at the
defaults cannot exhaust it. A raised `search(limit=…)` can: the 1023rd match
under one owner repeats a position, and the identity index drops it silently.

**Its owner is the innermost *unfolded, persistent* instance whose extent
contains it.** Each word is doing work:

- *persistent* — placement is a function of committed data alone, so a
  **cached** layer's numbering cannot depend on which query populated it first;
- *unfolded* — **a macro expansion owns no text.** Every byte at the call site
  of `foo(a)` was typed by the caller, so the owner of a match anywhere inside
  it is the enclosing function — and there is no expansion instance to pick in
  the first place. What the word still excludes is macro-*generated* code,
  which owns the text of the declarations it produced but never a call site;
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

Three accepted limitations:

- `X { search(…) }` searches X's *extent*, not X's subtree. A scoped search
  fuses the scope's byte ranges into its own corpus, so a declaration whose
  range lies outside its container's (§2) is reached by `X { unnest }` but
  its text is not by `X { search(…) }`. Extents stop being containment; they
  stay load-bearing for search.
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
- `nesting_level` equals the actual depth, and a root instance has
  `fold_depth` 0;
- `fold_depth` is the parent's plus its `folded` flag;
- every ref's `dfs_enter` falls strictly inside some instance interval, and
  the ref's `nesting_level` and `fold_depth` are its owner's plus one and
  plus the owner's `folded` flag.

A note on the containment check, because the cheap version of it is wrong: a
ref may legitimately sit after a nested instance has closed while an ancestor
is still open, so it must be tested against the top-level intervals, not
against the nearest instance below it.

A failing object rejects the upload with HTTP 400 naming it
(`object <local_id>: …`); nothing of that chunk is stored.

An index produced before the containment tree existed has every interval at
`[0,0)` and fails the first check, so this doubles as the guard against an
unnumbered project being served as a silently empty graph.
