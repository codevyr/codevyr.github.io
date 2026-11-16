---
title: "Containment"
description: "Why source ranges cannot express what contains what, the DFS interval and nesting level that replace them, where a declaration and a reference sit, and why folding is kept out of it"
weight: 100
---

Every query that nests — `mod("amdgpu") { func }`, `func("main") { "printk" }`,
`{ "inner" }` — is asking a containment question, and the answer has to come
from data the indexer wrote down. This page is about what that data is and why
the obvious choice does not work.

The short version: **containment is a position in a depth-first traversal, not
a source range.** An instance has a half-open interval `[dfs_enter, dfs_exit)`,
and `Y` is inside `X` exactly when

```
X.dfs_enter < Y.dfs_enter < X.dfs_exit
```

with both in the same object. One further column, `nesting_level`, says *how
far* inside. What a query *lists* by default is a different question with a
different answer — one flag on the symbol, [folding](#folding) — and
containment never consults it.

## 1. Why ranges are not enough {#why-not-ranges}

The natural encoding is the one the source already has: a symbol spans bytes
`[start, end)`, and `Y` is inside `X` when `X`'s range contains `Y`'s. That is
what askl used, and it is wrong in C for two independent reasons.

**A macro expansion cannot contain what it generates.** The indexer used to
record an expansion clamped to the macro *name* — measured on the deployed
index, **5,439 of 5,439** sampled expansions had width exactly the name's
length, so an expansion could never contain anything. Widening it to the whole
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
parent's.** `First` occupies five bytes and contains a closure thirty bytes
away. That is the encoding working as intended — the interval decides
containment and never consults a range.

C has the mirror image: **a declaration's bytes may lie inside one that does
not contain it.**

```c
#define PROFILE_FUNC(type, name, p) int blah; type blah_##name(int p)

PROFILE_FUNC(int, send, x) { return helper(x); }
```

`blah` is made of the macro's own tokens. clang knows they are in the
`#define`, but the range libclang reports for a declaration that ends in a
macro body ends at the *end of the invocation* — a convention from 2011, so
that an editor can highlight `y = NO` in one piece — and the indexer anchors
the start at the invocation too. So the only extent `blah` has is
`PROFILE_FUNC(int, send, x)`. The function `blah_send` starts at that same
invocation and runs on through a body the author typed, so the invocation, and
`blah` with it, lies in the function's text. Read as text, a global variable is
inside a function.

The extent of a generated declaration says *came out of this invocation*, not
*its text is here*. What does tell the two apart is the walk: the compiler read
`blah` at file level, before `blah_send` was opened. So that is where it sits —
beside the function, with its bytes inside the function's. The rule is in
[where things sit](#placement).

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

Three consequences worth stating, all accepted:

- **Two compilations can read one declaration in different places.** A file
  compiled twice under two names declares `work` and `work_variant` on the
  same bytes, and an assert a macro generates in the body is one instance that
  each compilation read inside a different function. A tree has one parent per
  node and neither is wrong, so the walk's answer is dropped for that instance
  and its text decides. Keeping the first to arrive would make the tree depend
  on thread timing.
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

## 3. The two axes {#axes}

| axis | columns | question it answers |
|---|---|---|
| position | `dfs_enter`, `dfs_exit` | is Y inside X at all? |
| depth | `nesting_level` | how many levels down? |

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
union members generated at one extent, inside an ordinary definition:

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

### Folding is not an axis {#folding}

There used to be a third axis: `folded` and `fold_depth` on every instance and
reference, counting the generated declarations above it, with the default
traversal comparing the counts. It answered "which levels are interesting" by
*position*, and position is the wrong thing to ask.

- **It hid by where a symbol sat.** A generated symbol was left out when it sat
  inside another generated symbol and listed when it sat directly in a
  function. On the kernel that listed 455,417 `__compiletime_assert_N`
  instances and hid 59,880.
- **It leaked into containment.** For the counts to come out right, a reference
  typed in a generating macro's arguments could not sit in the declaration the
  macro generated, so placement skipped folded instances. A function whose
  *name* merely passed through a macro — `UCS_PROFILE_FUNC(ret, name, …) { body }`
  — counted as folded, took no references, and every call in its body landed
  on the file: ucx's `ucp_tag_send_nbx` had 71 references written inside it and
  none placed in it.

So the two are separate, and neither knows about the other.

**Containment is decided by text and by the walk**, as
[where things sit](#placement) spells out. Nothing is placed, or passed over,
because of folding.

**Folding is one flag on the symbol**, `symbols.folded`: *not listed by
default*. The indexer sets it for a symbol nobody wrote down in normal code —
the rule is in [C indexing](/docs/c-indexing/#folding). It is per symbol
because the question spans the project: a reference in one file unfolds a
symbol that a macro in another file generated.

```c
void f(void) {
    BUILD_BUG_ON(sizeof(struct x) > 4);   /* generates __compiletime_assert_7 */
}
```

| | sits in | listed by default? |
|---|---|---|
| `f` | the file | yes |
| `__compiletime_assert_7`, whose extent is the invocation | `f` | **no** — its symbol is folded |
| the reference to `struct x`, typed in the argument | `f` | **yes** |

`f { }` lists `x` because the reference lies in `f`'s interval, and leaves the
assert out because nobody wrote its name. Neither answer consults the other.

What a macro **calls** is not a fold question either: references are
**layerwise**, one edge per `#define`, so `handler -> DEV_PRINT -> printk` is a
chain of ordinary edges and `handler -> printk` is a traversal of it. The
producer records the body reference where the macro author wrote it, an
invocation reference at each call site, and on the macro's own name a
reference to each declaration an invocation [named](#declares).

## 3b. Where things sit {#placement}

Two sentences place every declaration and every reference:

> **A declaration somebody typed contains what lies in its text. A declaration
> a macro generated has no text of its own, so it contains only what the
> compiler read inside it.**

*Generated* means the declaration's whole extent is one macro invocation.
`DEFINE_MUTEX(lock)` generates `lock`; `PROFILE_FUNC(int, send, x) { … }` does
not generate `blah_send`, because its extent runs on through a typed body.

```c
int f(struct x *p) {
    BUILD_BUG_ON(sizeof(struct x) > MAX_DEPTH);
    return helper(READ_ONCE(p->state));
}
```

Both macros generate a `__compiletime_assert_N` whose extent is its invocation.
Text alone would hand those declarations everything the invocations mention.
They read none of it: `struct x`, `MAX_DEPTH`, `p->state` and both macro names
were read in `f`, and sit in `f`.

| text | read inside | sits in |
|---|---|---|
| `struct x` in the `BUILD_BUG_ON` argument | `f` | `f` |
| `KHASH_INIT` in `KHASH_INIT(x, int, my_hash)` at file level | nothing | the file, beside the functions it generated |
| `my_hash`, used once in each of three generated functions | each of the three | each of the three |
| `helper` in a typed body | `f` | `f` |

The lookup asks the extents around a piece of text innermost first. If it was
read inside a declaration with that extent, it sits there. Otherwise it sits in
the declarations with that extent that were typed. Otherwise the next wider
extent is asked. One function does this for an instance's parent and for a
reference.

**A generated declaration sits where it was read**, for the same reason: the
invocation's extent is not its text, so text cannot say what it is inside. It
is not given to a declaration the walk enters unless the compiler read it
there. That is what puts `blah` beside `blah_send` in [§2](#stated). What the
walk does not enter — the file, a `#define` — still holds a generated
declaration by text.

**Fields and enum constants are read inside, like functions.** With
`#define PAIR(type_t, a, b) type_t a; type_t b;`, the invocation
`PAIR(struct cfg, first, second)` declares two fields on one extent, each a
`struct cfg`. The one `cfg` token is read once inside each field and sits in
each, so `holder.first -> cfg` and `holder.second -> cfg` both exist.

**A reference is what it names, the token that names it, and where it sits.**
One token can be read in several declarations — `my_hash` above, or a body
compiled twice under two names — and is then one reference in each, three rows
that differ only in their position. The edge exists from every function that
uses the helper. A listing of *occurrences* collapses them again: one line of
source is shown once, however many declarations read it.

Two typed declarations can share one text, when a file is compiled twice under
two names. Both contain what lies in it, so a macro use in such a body is a
reference in each. An instance still takes one parent, the first.

What this gives up: a macro *use* is recorded before the walk, so it is read
inside nothing. A macro named in the arguments of a generating invocation —
`NFT_REG32_NUM` in `DECLARE_BITMAP(reg_inited, NFT_REG32_NUM)` — sits in the
struct around the generated field, not in the field.

## 3c. What a macro declared {#declares}

The two sentences above leave one edge pointing the wrong way. In

```c
static DEFINE_MUTEX(lock);
PROFILE_FUNC(int, send, x) { return helper(x); }
```

`lock` and `blah_send` have text of their own, and the invocation lies in it,
so placement by text gives `lock -> DEFINE_MUTEX` and `blah_send ->
PROFILE_FUNC`: the variable *uses* the macro. Nothing leads from a macro to
what it made.

The difficulty is that this is not a tree. The invocation and the declaration
overlap partially — `PROFILE_FUNC(int, send, x)` is inside `blah_send`'s
extent, and `blah_send` came out of `PROFILE_FUNC`. One of the two has to
give, and the invocation is the one that has no text to lose:

> **An invocation that supplies a declaration's name declares it: the macro
> refers to the declaration, and the invocation sits where the declaration
> sits, not inside it.**

So `file -> DEFINE_MUTEX -> lock` and `file -> PROFILE_FUNC -> blah_send ->
helper`. No declaration contains the invocation that names it; that is the
one exception to placing a typed declaration's contents by text.

Three alternatives were weighed and dropped.

| | what it is | why not |
|---|---|---|
| invocation as an *instance* of the macro, containing what it generated | precise per invocation | one symbol with thousands of instances, which is the fan-out that made `mod("amdgpu")` slow; and references would have to be forbidden from sitting in one |
| derive the edge in the engine from the invocation's span and the declaration's extent | no new rows | paid per invocation at query time, even for an empty answer — 1.8 s to learn that `NULL` declares nothing on Linux — and five read paths to teach |
| both edges for the partial overlap | keeps `blah_send -> PROFILE_FUNC` | a cycle through every definition a macro names |

*Supplying the name* is tested on the name token, not on the extent. An
earlier form asked whether the declaration lies wholly inside the invocation,
and missed 1,012 of the kernel's 1,098 file-level `DEFINE_MUTEX` variables,
which are written `static DEFINE_MUTEX(x)`.

Two clauses keep the edge to what the invocation is *for*. It must name the
declaration **directly** — the fields of a struct it named are reached through
the struct. And the declaration must be **introduced** there, as a definition
or the first declaration: `EXPORT_SYMBOL_GPL(foo)` redeclares `foo`, and
without this clause that one macro would list 20,800 kernel symbols.

The reference is the macro's and is written in a `#define`, so it unfolds
nothing ([folding](#folding)).

What it does not solve: a macro is one node, so `other -> HOOK -> hooks.open`
reads as a path when two structs use `HOOK`. Only a per-invocation record
could refuse that path, and that is the instance model again.

## 4. The three traversal modes {#modes}

Every mode is the interval range plus at most one equality. See
[Traversal modes](/docs/syntax/#traversal-modes) for the surface syntax.

| mode | extra predicate | reads |
|---|---|---|
| `nested` | `Y.nesting_level = X.nesting_level + 1` | `nesting_level` |
| default | `NOT folded`, on Y's symbol | the symbol row the query already joins |
| `unnest` | — | nothing |

The default is not *direct*. Direct would make `mod("amdgpu") { func }` empty,
since a module's direct children are files. The default reaches every level
and leaves out folded symbols — and a folded *container* is passed over, not
stopped at: what sits inside it is listed on its own merits. The same predicate
runs upward, on the container's symbol, so `{ "x" }` lists the function and
its file and not a generated declaration between them.

A statement that **names** a symbol never goes through this filter. It selects
by name and is then constrained by its neighbour's interval, so
`f { "__compiletime_assert_7" }` returns a declaration that `f { }` does not
list. The filter exists only where a statement is *derived* from its
neighbour.

`unnest` is how to see what the default leaves out: everything the
preprocessor generated for itself. It is not needed to reach what a macro
*calls* — that is an edge.

Three consequences worth stating, all accepted:

- `X { Y }` no longer means strictly "direct". In Go, `func { func }` returns
  closures nested at any depth, and a reference inside a local declaration
  counts as the enclosing function's.
- `nested` is the only reader of `nesting_level`, and the only way to ask for
  exactly one structural level: a file's own declarations, or the single
  container a symbol literally sits in. It ignores folding and is literal about
  levels, so in the example above `f { nested }` lists the generated assert as
  well as `x`: both sit directly in `f`. It is **not** a window onto what a
  macro generated — a generated declaration is placed where it was written, so
  `macro("ucs_offsetof") { nested }` returns only `ucs_offsetof`, and
  `macro("__KHASH_IMPL") { nested }` returns the macros its body calls.
- One line of source can be several references. A token read in three
  generated functions is an edge from each ([where things sit](#placement)),
  and the engine's unique key for a reference includes its position for that
  reason.

## 5. Ephemeral rows {#ephemeral}

Verbs like [`search()`](/docs/design/search/) materialise instances at query
time into an existing object. Those rows need a position too, or they fall
outside every interval and become invisible.

Two rules place them.

**An ephemeral instance is a leaf.** It gets a width-1 interval, one integer
wide, because nothing is ever placed inside one — a reference sits below every
slot, and an ephemeral row may not contain another. That makes laminarity
*automatic*: width-1 integer intervals cannot partially overlap, so two rows
in one container, from whatever batches, are always siblings and never nest by
accident. The gap holds 1022 of them per container — above the default read cap
(1000) and `search()`'s default `limit` (500 per project), so a query at the
defaults cannot exhaust it. A raised `search(limit=…)` can: the 1023rd match
in one container repeats a position, and the identity index drops it silently.

**It sits in the innermost *persistent* instance whose text holds it.** Each
word is doing work:

- *persistent* — placement is a function of committed data alone, so a
  **cached** layer's numbering cannot depend on which query populated it first.
  The rule itself is part of every layer's cache key, so changing it retires
  the layers the old rule placed;
- *innermost* — narrowest extent, then **deepest in the tree**, with
  `dfs_enter` only as a final tiebreak.

  Deepest matters because of the container chain: a match outside every
  declaration — an `import` block, a comment between functions — has the module,
  the directory and the file as candidates, all co-extensive. Taking the
  outermost numbers it *below its own file's interval*, out of reach of
  `file(…) { … }` in every traversal mode, since every mode starts from the
  interval and only adds a predicate;
- *text* — an extent is not always text ([§2](#stated)). **An instance whose
  bytes lie in a wider extent that does not contain it has no text there, and a
  match passes it over.** In the `PROFILE_FUNC` example a match on `x` has
  `blah` as its narrowest extent, and `blah` sits beside `blah_send`, not in
  it. Placed in `blah`, the match would be outside the function it was typed
  in, and `func("blah_send") { search("x") }` — which reads the function's
  bytes — would find the text and then report that nothing related. Bytes
  disagreeing with the tree is the one thing the engine can see about what a
  declaration is made of, and it is enough for this case.

  "Does not contain it" is asked of the extent, not of one instance. Several
  declarations can share an extent and a tree gives what lies in them one
  parent; being inside any of them is agreement. So nothing moves in an index
  whose tree follows its bytes.

This is the indexer's law as far as the engine can see it, and it cannot see
all of it. An instance row does not say it was generated, so a match in
`KHASH_INIT(x, int, my_hash)` at file level lands in one of the functions the
invocation generated — the earliest — where the indexer puts a *reference* in
the file beside them. That match is still inside everything that contains the
invocation, so no scoped search loses it; it is attributed to one member of
the family rather than to none.

Folding plays no part. There used to be an exclusion here, of *folded*
instances; it made where a match sits depend on what is shown.

The row then takes its depth from that container, so it behaves exactly as a
symbol at that position would.

Four accepted limitations:

- A match has one place. The text in `PROFILE_FUNC(int, send, x)` lies in the
  bytes of both `blah` and `blah_send`; `func("blah_send") { search("x") }`
  finds it and `data("blah") { search("x") }` does not.
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
- `macro("foo") { search(...) }` searches the macro's `#define`, its only
  instance. The text at its call sites is searchable from the functions that
  wrote it.

## 6. Validation {#validation}

The engine reads only the *child's* `dfs_enter`, which means a malformed tree
produces plausible wrong answers instead of errors. So uploads are checked, per
object, before anything is stored:

- every interval is non-empty (`dfs_exit > dfs_enter`);
- every interval is at least one **stride** (1024) wide — the gap query-time
  rows are numbered into ([ephemeral rows](#ephemeral)), which numbering by
  stride provides for free. A merely *narrow* container loses its own search
  matches just as silently as an empty one;
- `dfs_enter` is unique — the DFS order is total;
- intervals nest, never straddle;
- `nesting_level` equals the actual depth;
- every ref's `dfs_enter` falls strictly inside some instance interval, and
  the ref's `nesting_level` is one more than that of the instance it sits in.

Folding is not validated: it is a flag on the symbol with no arithmetic to get
wrong, and an index that sets none lists everything.

A note on the containment check, because the cheap version of it is wrong: a
ref may legitimately sit after a nested instance has closed while an ancestor
is still open, so it must be tested against the top-level intervals, not
against the nearest instance below it.

A failing object rejects the upload with HTTP 400 naming it
(`object <local_id>: …`); nothing of that chunk is stored.

An index produced before the containment tree existed has every interval at
`[0,0)` and fails the first check, so this doubles as the guard against an
unnumbered project being served as a silently empty graph.
