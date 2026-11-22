---
title: "Askl Syntax Reference"
description: "Complete reference for the Askl query language syntax"
weight: 200
---

Askl is a pattern-matching query language designed for source code analysis. It allows you to find symbols (functions, modules, files), trace dependencies, and build custom views of your codebase.

## Overview

An Askl query consists of **statements** that select and filter code symbols and define relationships between them through **scopes**.

### Basic Structure

```askl
statement1
statement2 {
    nested_statement
}
```

## Core Concepts

### 1. Statements

A **statement** is the fundamental unit of an Askl query. Each statement contains:

- **Commands**: One or more verbs that define what to select or filter
- **Scope**: Optional nested statements that define relationships

#### Statement Separation

Newlines separate consecutive statements:

```askl
"foo"           /* First statement */
"bar"           /* Second statement */
```

Semicolons also work as separators (useful for single-line queries):

```askl
"foo"; "bar"    /* Two statements on one line */
```

At the top level the two statements are independent: both run, and their results
are unioned into one graph. **Inside a scope, `;` does something else** — it
separates *sibling* children, and those **conjoin**. See
[Sibling Statements in a Scope](/docs/syntax/#sibling-statements-in-a-scope).

Without newlines or semicolons, verbs on the same line belong to the same statement:

```askl
"foo" "bar"     /* Single statement with two selector verbs */
```

A scope `{` must be on the **same line** as its verb to attach:

```askl
"foo" { "bar" }  /* "bar" is inside foo's scope */

"foo"
{ "bar" }        /* Two separate statements (scope does NOT attach to "foo") */
```

### 2. Symbol Types

Askl knows eight symbol types:

| Type | Description |
|------|-------------|
| `field` | Struct fields, interface methods (dispatch points) |
| `function` | Functions, methods, procedures |
| `type` | Structs, interfaces, type declarations |
| `data` | Package-level variables, constants |
| `macro` | C/C++ preprocessor macros (`#define`) |
| `file` | Source files |
| `module` | Packages, modules, namespaces |
| `directory` | Filesystem directories |

A type says what a symbol *is*, not where it sits. Which symbol **contains** which — a module its files, a file its declarations, a struct its fields, a function a closure declared inside it — is a tree the indexer writes down and a scope reads; see [Containment](/docs/design/containment/) for the model and [Traversal modes](#traversal-modes) for how far a scope descends it.

### 3. Relationships

A scope asks how a parent and its children are related. There are four
relationships, one for each mark the report draws, and no edge is two of them:

| Word | Mark | The parent… |
|------|------|-------------|
| `has` | `▸` | **contains** the child, in the indexer's containment tree |
| `refs` | `→` | **refers to** the child: a token of the parent names it |
| `declares` | `↦` | **declares** the child: an invocation of the macro declared it |
| `holds` | `⇢` | **holds** the child: a slot (a struct field, an interface method) and a function stored in it |

```askl
"foo" refs { "bar" }               /* bar, if foo references it */
mod("mypackage") has { func }      /* Functions contained in module */
file("/main.go") has { "handler" } /* handler, if main.go contains it */
```

Three more words narrow `refs` to the references that say what they do to
their target: `calls`, `writes` and `takes` (also spelled `reads`). All of them
are described under [Relationship Words](#relationship-modifiers).

#### The default is all four {#default-relationship}

A scope with no relationship word on it accepts **any** of the four: `"foo" { "bar" }` keeps `bar` whether `foo` references it, contains it, declares it or holds it. That is why `mod("pkg") { func }` and `file("/main.go") { "handler" }` work without `has`, and why `"foo" { "bar" }` reads as "foo calls bar" when both are functions, which cannot contain one another. A word governs the statement it is written in and no other: the scopes nested below go back to the default unless [`inherit(relationship)`](#inherit) carries the word down. Type selectors — `func`, `mod`, `file`, … — never change the relationship; they only constrain the symbol type.

### 4. Pattern Matching

Name arguments come in two string types:

- **Plain strings** (`"name"`) use exact, case-sensitive matching.
- **Glob strings** (`g"name*"`) opt into wildcard matching — see [Glob Patterns](#glob-patterns-g) below.

A prefix glued to the opening quote selects the string type. The prefix `re"..."` is reserved for future regex support; unknown prefixes are rejected with a parse error.

For plain strings, how a name is matched depends on whether it contains separator characters:

#### Simple Names (Leaf Matching)

A name with **no separators** matches the **last component** of the symbol path. This uses a fast B-tree index lookup:

- `func("handler")` matches functions whose last path component is "handler"
- `"main"` matches symbols named "main" (not "main_helper" or "domain")
- `field("read")` matches fields named "read" across all structs

**Separator characters** depend on the symbol type:
- **Code symbols** (func, type, data, macro, field, mod): `.` `/` `:`
- **Files and directories**: `/` `:` (`.` is NOT a separator — it's part of filenames)

So `file("main.go")` is a simple name (no `/` or `:`) and matches files whose last path component is "main_go".

#### Compound Names (Pattern Matching)

A name with **separators** uses pattern matching across the full symbol path:

- `"cli.Run"` matches symbols containing both "cli" and "Run" tokens in order
- `func("http.Handler")` matches functions with "http" and "Handler" in their path
- `"cli"` will NOT match "click" (exact token matching)

#### Path-based Matching for Files and Directories

File and directory selectors support additional matching modes:

| Argument | Matching | Example |
|----------|----------|---------|
| Starts with `/` | Exact path match | `file("/src/main.go")` |
| Simple name (no `/` `:`) | Leaf match (last component) | `dir("kueue")` matches directories **named** "kueue" |
| Compound (has `/` or `:`) | Leaf-anchored pattern | `dir("pkg/kueue")` matches "kueue" dirs with "pkg" earlier in path |

For compound dir/file queries, intermediate tokens match anywhere but the last is anchored: `dir("pkg/kueue")` matches directories named "kueue" that have "pkg" somewhere earlier in their path.

To match a token **anywhere** in the path (non-anchored), use `match="contains"`:

```
dir("kueue", match="contains")   // matches any directory with "kueue" in its path
file("main", match="contains")   // matches any file with "main" in its path
```

> **Note:** The `match` parameter has no effect when the argument starts with `/` (exact path match always takes precedence).

#### Glob Patterns (`g"..."`)

A string prefixed with `g` is a **glob pattern**: `*` matches any run of characters (including none). Globbing is opt-in — a `*` inside a plain string is not a wildcard.

```
g"handle*"          // functions starting with "handle"
g"*alloc*"          // anything containing "alloc"
file(g"*.go")       // all Go files
ignore(g"*_test")   // exclude test symbols
```

Glob strings work in every name position: bare selectors, `func()`/`file()`/`mod()`/... (in both selector and filter mode), `ignore()`, and `filter("exact_name", ...)`/`filter("compound_name", ...)`.

**Matching rules:**

- **Anchored**: a glob matches the whole name (the leaf, or for compound patterns the full symbol name). To match anywhere, add explicit wildcards: `g"*handle*"`, not `g"handle"`. For leaf globs you can also pass `match="contains"` to wrap the pattern implicitly.
- **Smart case**: an all-lowercase pattern matches case-insensitively; any uppercase letter makes the match case-sensitive. `g"is*"` matches `IsSorted`, but `g"Is*"` does not match `issorted`.
- **Simple patterns** (no separators) match against the last path component, like plain simple names. Literal characters are normalized the same way symbol names are indexed, so `g"my-app*"` matches a file stored as `myapp_c`.
- **Compound patterns** (containing `.`, `/` or `:`) match the full symbol name, anchored: `g"fmt.Print*"` matches names that start with `fmt.Print`, and `g"/src/*"` matches paths under `/src/`.
- A pattern must contain at least one literal character that survives normalization; bare `g"*"` (and patterns like `g"-*"` whose only literal is stripped) are rejected.

> **Note:** Plain strings remain the right tool for pasted symbol names. A name like `"(*Kubelet).Run"` matches exactly even though it contains `*`, because plain strings never treat `*` as a wildcard.

> **Performance:** Glob matching is served by a trigram index, which needs a run of at least 3 literal characters to narrow the search. Patterns whose literals are all shorter than that (e.g. `g"*ab*"`) fall back to scanning every symbol in scope — prefer a longer literal anchor where possible.

### 5. Argument Types

Verb arguments are typed. There are three primitive types:

| Type | Written as | Example |
|------|-----------|---------|
| String | quoted, optionally with a type prefix | `"vfs_read"`, `g"handle*"` |
| Integer | bare digits, 64-bit signed | `500`, `42` |
| Boolean | the bare keywords `true` / `false` | `true`, `false` |

```askl
search("EXPORT_SYMBOL", whole_word=true, limit=2000)
loc("main.c", 42)
label("handlers", inherit=true)
```

**A quoted value is a string, even when it reads like a number or a flag.**
`limit="500"` passes the *string* `"500"` where an integer is wanted, and
`whole_word="true"` passes a string where a boolean is wanted. Both are errors,
and the error names the spelling to use:

```
search: 'limit' expects an integer, found the string "500" — write limit=500
```

This matters because arguments used to be strings without exception. Queries
written before argument types existed need the quotes removed from `limit`,
`whole_word`, `inherit`, `forced`, and `loc()`'s line number.

Which type an argument takes follows what it means, not what it looks like.
`case="smart"` stays a string because it is one of three named modes, not a
yes/no.

Literals belong in argument position only. A bare `42` or `true` at the start
of a statement is not a selector — to name a symbol called `42`, quote it:
`"42"`.

Every verb declares which arguments it takes, and anything else is an error
rather than silently dropped — `func("main", bogus=1)` and `func("a", "b")`
(which reads one name) both fail at parse. Whitespace around `=` is fine:
`limit = 500` and `limit=500` are the same.

## Verb Types

Verbs in Askl fall into three categories:

### Selectors

**Selectors add symbols to the result.** They query the database and produce nodes in the graph.

| Verb | Description |
|------|-------------|
| `"name"` / `g"pattern"` | Select symbols matching a name (exact) or glob pattern |
| `func("name")` | Select functions matching a name |
| `type("name")` | Select types matching a name |
| `data("name")` | Select data symbols (variables, constants) matching a name |
| `macro("name")` | Select macros matching a name |
| `field("name")` | Select field symbols (function pointer dispatch points) matching a name |
| `method("name")` | Alias for `field` — select interface methods / virtual dispatch points |
| `mod("name")` | Select modules matching a name |
| `file("name")` | Select files matching a name |
| `dir("name")` | Select directories matching a name |
| `use("label")` / `#label` | Select symbols from a labeled statement |
| `search("text")` | Full-text search: create a symbol per byte-range match of `text` in indexed source content |
| `select` | Make the statement **binding**: enumerate everything the statement's filters allow (budget-bounded), and keep the statement constraining in composition |

Multiple selectors in a statement are **OR-ed** — the statement selects the union of
their matches, constrained by the statement's filters. A named type selector
carries its own type, so `func("open") data("fops")` is the function and the
variable. A bare type word filters every selector beside it (`func "open"
"close"`), which is why two type words side by side — `macro func`, `data
func("open")` — are an error: write `macro or func`. Inside a scope that is exactly
the contrast with `;`: `X { A B }` is one child whose selectors disjoin, `X { A ; B }`
two children that conjoin
([Sibling Statements in a Scope](/docs/syntax/#sibling-statements-in-a-scope)).
The disjunction can also be written explicitly — `"open" or "close"` — which
additionally merges the operands into ONE branch with a single no-match
warning; see [Boolean Operators](#boolean-operators).

Every query group — a *component*: one or more top-level statements with
their nested scopes, joined by label references — must contain at least one
**anchor**: a name, a name filter (`filter("exact_name"/"compound_name", …)`),
`search(...)`, `loc(...)`, a layer literal, or `select`. A group made only of constraints
(`func`, `project(...)`, `filter("type", ...)`) cannot produce anything on its
own and is rejected with a hint instead of silently returning an empty result.

### Filters

**Filters constrain the selection without adding symbols.** They remove symbols that don't match criteria.

| Verb | Description |
|------|-------------|
| `ignore("pattern")` | Exclude symbols whose path carries the pattern, on any label (see [ignore](#ignore-symbol-filtering)) |
| `project("name")` | Only include symbols from a specific project |
| `package("path")` | Only include symbols strictly under a package path |
| `filter("kind", "value")` | Generic filter (see below) |
| `func` (no name) | Only include function symbols |
| `type` (no name) | Only include type symbols |
| `data` (no name) | Only include data symbols |
| `macro` (no name) | Only include macro symbols |
| `field` (no name) | Only include field symbols |
| `method` (no name) | Only include field symbols (alias) |
| `mod` (no name) | Only include module symbols |
| `file` (no name) | Only include file symbols |
| `dir` (no name) | Only include directory symbols |
| `any` (no name) | Include every symbol type (removes a type carried down with `inherit(type)`) |

> **Note:** Type selectors like `func` behave differently based on arguments:
> - Without a name (`func`) → **Filter**: constrains the statement it is written in to that type, and no other statement
> - With a name (`func("foo")`) → **Selector**: one branch of its statement, carrying its own type
> - `any` is the member of this family that constrains no type — the same two
>   forms apply to it (`any` filters, `any("foo")` selects)
> - `func select` enumerates all functions (budget-bounded)
>
> The old `filter="true"/"false"` argument was a manual query-plan toggle and has
> been **removed** (queries using it fail to parse with a migration hint): the
> engine now plans execution from measured cardinality. Use `select` where you
> wrote `filter="false"`; for namespace scoping (`mod("x", filter="true") "name"`)
> use composition (`mod("x") has { "name" }`) or `filter("compound_name", "x")`.

### Modifiers

**Modifiers change context or behavior** for the current statement or scope.

| Verb | Description |
|------|-------------|
| `has` | The scope asks for containment (`▸`) |
| `refs` | The scope asks for ordinary references (`→`) |
| `declares` | The scope asks for what a macro's invocations declared (`↦`) |
| `holds` | The scope asks for what a slot holds (`⇢`) |
| `calls`, `writes`, `takes` / `reads` | `refs`, narrowed to the references that state that role |
| `inherit(relationship)`, `inherit(type)` | Carry the statement's relationship, or its type, into the scopes below |
| `preamble` | Apply subsequent verbs to every statement that follows |
| `label("name")` / `@name` | Label this statement for reuse |
| `unnest` | List every descendant, folded symbols included |
| `nested` | List exactly one structural level |
| `!` (forced) | Force display of relationships |

> **Weakness is inferred, not written.** There is no weak marker: bare
> `{ }` scopes and commands with only filters are non-constraining
> *echoes* by default, and `select` is what makes a command constraining.
> The exception is a bridge: a non-constraining command between two
> constraining parts of a statement constrains, so `func { "a" ; "b" }`
> finds functions calling both. See
> [Weakness and Bindness](/docs/design/semantics/#weakness-and-bindness).

## Boolean Operators

Predicate verbs combine with `or`, `and`, `not`, and grouping parentheses
(adopted design: [Filter Expressions](/docs/design/filter-expressions)):

```askl
func or method { "foo" }                 /* foo, of either type */
project("a") or project("b") { "bar" }   /* bar in either project */
"open" or "close"                        /* one branch: either name */
func and not g"test_*"                   /* functions except test_* */
not (g"kl*" and package("k8s.io/klog")) "main"
```

**Precedence:** `not` binds tighter than `and`, which binds tighter than
`or`; parentheses group. **Operators bind tighter than whitespace**: in
`func or method "foo"`, the type group forms first and `"foo"` is a
separate, juxtaposed selector. A bare newline still terminates the
statement; newlines are allowed inside parentheses.

**What may appear in an expression.** Predicate verbs only: name patterns,
type selectors (bare or named), `project(...)`, `package(...)`,
`filter(...)`, `ignore(...)`. Everything else is rejected with a hint:
traversal modes (`unnest`, `nested`), `inherit(...)` and labels are not
predicates; `search(...)`, `loc(...)`, layer literals, and `!"..."` are
not a single predicate query; bare `any` and `select` constrain nothing.
Inside an expression a verb contributes only its predicate — `file or dir
{ … }` does not switch the scope to containment; write `has` explicitly.

**Relationship words are an expression of their own.** `has`, `refs`,
`declares`, `holds` and the role words join with each other, with `or` and
parentheses only (`calls or holds { … }`), and never with a predicate: `func
or has` is an error that says to write them side by side. See
[Joining relationship words](#relationship-or).

**Three rules keep expressions predictable:**

- Per `or`, operands are either all anchors or all filters.  An
  all-anchor group is ONE branch: one query and one no-match warning for
  the whole disjunction — `"a" or "b"` says *either is fine*, while
  juxtaposed `"a" "b"` keeps a warning per name (*each should exist*).
- An expression without an anchor must constrain a single dimension
  (type, project, package, or one filter kind).  Filters of different
  dimensions conjoin by juxtaposition; for a cross-dimension disjunction,
  split into sibling statements.
- A filter group inherits into `{ }` scopes **as a unit**, when each of
  its positive members would on its own: `project("a") or project("b")`
  does, `func or method` does not, because a type word stays in its
  statement.  A child statement writing the same dimension replaces an
  inherited group wholesale.  Exclusions (`not ...`) accumulate instead —
  they narrow, like `ignore`, and inherit unconditionally.

> **Footgun:** `func ("foo")` — with a space — is still an argument list,
> not `func` AND a grouped name: a `(` after a verb name binds as
> arguments.

Two shapes are rejected outright rather than guessed at: `not not x`
(remove both), and a line that *begins* with `or` or `and` — an operator
continues a statement onto the next line only when it ends the previous one.

## Type Selectors

Type selectors target specific symbol types. As explained above, they act as **selectors** (with a name) or **filters** (without a name).

### Selector vs Filter Behavior

| Syntax | Role | Behavior |
|--------|------|----------|
| `func("name")` | Selector | Queries functions matching "name" |
| `func` | Filter | Constrains to functions; derives from its neighbours |
| `func select` | Selector | Enumerates ALL functions (budget-bounded) |

**Why this default?** A bare type predicate denotes a huge set; on its own it is
a constraint, not a selection. The engine decides *how* each statement is
materialised from measured cardinality (capped probes + semi-join refinement),
so the old `filter=` plan toggle is gone.

### func

Selects function symbols.

```askl
func("handler")         /* Functions matching "handler" */
func("http.Handler")    /* Functions matching both "http" and "Handler" */
func select             /* All functions (budget-bounded) */
file("/main.go") { func }  /* Functions in main.go (filter mode) */
```

### type

Selects type symbols (structs, interfaces, type declarations).

```askl
type("Request")          /* Types matching "Request" */
type("http.Request")     /* Types matching both "http" and "Request" */
type select              /* All types (budget-bounded) */
mod("net/http") { type }  /* Types in module (filter mode) */
type("Request") { type }  /* Types referenced by Request */
```


### data

Selects data symbols (package-level variables and constants).

```askl
data("Debug")            /* Data symbols matching "Debug" */
data("config.Debug")     /* Data symbols matching both "config" and "Debug" */
data select              /* All data symbols (budget-bounded) */
mod("config") { data }   /* Data symbols in module (filter mode) */
```


### macro

Selects macro symbols (C/C++ preprocessor `#define` directives). The calls written in a macro's body are references recorded inside the macro's own containment interval, so `macro("LOG") { func }` lists them, together with the functions its invocations declared; a macro that calls a macro records only what its own body names, one `#define` per layer (see [Traversal modes](#traversal-modes)).

```askl
macro("LOG")              /* Macros matching "LOG" */
macro select              /* All macros (budget-bounded) */
func("main") { macro }   /* Macros referenced by main */
macro("LOG") { func }    /* Functions called inside LOG's body */
macro("DEFINE_MUTEX") { data }   /* The variables its invocations declared */
```


### field / method

Selects field symbols — struct members that act as function pointer dispatch points (C) or interface method signatures (Go). `method` is an alias for `field`.

Field names use compound naming: `struct_name.field_name` (e.g., `file_operations.read`). A simple name like `field("read")` matches any field whose last component is "read" across all structs, while `field("file_operations.read")` uses pattern matching for a precise match.

```askl
field("read")                      /* Fields matching "read" (broad) */
field("file_operations.read")      /* Precise match */
method("Read")                     /* Interface methods matching "Read" (Go) */
func("vfs_read") { field("read") { func } }  /* Full dispatch chain */
type("file_operations") has { field }         /* All fields of a struct */
{ field("file_operations.read") }             /* Everything that refers to the field: its callers and its assigners */
calls { field("file_operations.read") }       /* Only who calls through it */
field("file_operations.read") holds { func }  /* Only the functions stored in it */
```

The last two ask by [relationship](#relationship-modifiers), and need an index
that states roles and records `holds`
([what today's indices state](#what-an-index-states)).


### mod

Selects module/package symbols. A bare scope under it accepts either edge kind, so contained symbols are found without `has`.

```askl
mod("util")              /* Modules matching "util" */
mod("k8s.io/api")        /* Modules matching the pattern */
mod select               /* All modules (budget-bounded) */
mod("pkg") { func }      /* Functions in module (no has needed) */
```


### file

Selects file symbols.

```askl
file("main.go")          /* Files whose last component is "main_go" (leaf match) */
file("/src/main.go")     /* Exact path match */
file select              /* All files (budget-bounded) */
dir("/src") { file }     /* Files in /src directory */
```


### dir

Selects directory symbols.

```askl
dir("cmd")               /* Directories named "cmd" (leaf match) */
dir("/src")              /* Exact path match for /src */
dir select               /* All directories (budget-bounded) */
dir("/") { file }        /* Files in root directory (no has needed) */
dir("/") {}              /* Everything under the root directory */
```


### A type word and the scopes around it {#type-word-scope}

**A type word filters the statement it is written in, and no other.**

```askl
func { "module_mutex" }    /* the functions that use module_mutex, whatever it is */
"main" { func { } }        /* the functions main calls, and everything THEY use */
mod("mypackage") { }       /* everything in the module */
mod("mypackage") { func }  /* its functions */
```

In `func { "module_mutex" }` the word `func` is about the users. What is used
may be a variable, a macro or a type, and is found by its name. Nothing is
set for the child scope either: under any parent, a scope with no type word
accepts every type.

To carry a type into every scope below, ask for it with
[`inherit(type)`](#inherit) on the same statement. A scope that writes its own
type replaces the inherited one from there down, and bare `any` removes it:

```askl
data inherit(type) { { "bar" } }        /* both inner scopes are data too */
data inherit(type) { any { "bar" } }    /* any stops it: the inner scopes match all types */
```

### Scopes without a relationship word

A scope that carries no relationship word — and is handed none by an `inherit(relationship)` above it — accepts every relationship ([the default is all four](#default-relationship)). Type selectors do not change that:

```askl
/* Files inside /src */
dir("/src") { file }

/* Functions the module contains OR references; `has` keeps only the contents */
mod("pkg") { func }
mod("pkg") has { func }

/* "bar" is kept if main references it or contains it — its callees, in practice */
dir("/") { func("main") { "bar" } }

/* Say which relationship each scope asks for, where the default would be ambiguous */
mod("amdgpu") has { func refs { "drm_dev_enter" } }
```

## Relationship Words {#relationship-modifiers}

**A relationship word selects the edges drawn with its mark.** The report
draws a pair of symbols once, with every relationship the pair has
([reading the report](/docs/api/#query-parameters)); a relationship word asks
for the pairs drawn with one of those marks.

| Word | Mark | Selects |
|------|------|---------|
| `has` | `▸` | containment |
| `refs` | `→` | ordinary references: a token of the parent names the child |
| `declares` | `↦` | what the invocations of a macro declared |
| `holds` | `⇢` | what a slot holds: the functions stored in a struct field, the implementations of an interface method |
| `calls`, `writes`, `takes` (also spelled `reads`) | a word after the type | `refs`, narrowed to the references that state that role |

The four relationships are disjoint — `refs` does not include what a macro
declares or what a slot holds — and a scope with no word accepts all four
([the default](#default-relationship)).

**A word acts on the statement it is written in.** It describes the edge into
that statement's braces, and it may stand anywhere among the statement's
terms:

```askl
func calls { "kfree" }     /* the functions that call kfree */
calls func { "kfree" }     /* the same query: the order of terms means nothing */
calls { "kfree" }          /* whoever calls kfree */
"vfs_read" calls { }       /* what vfs_read calls */
```

It reaches no further than those braces. A statement nested inside is back to
the default unless it writes a word of its own, or
[`inherit(relationship)`](#inherit) carries this one down. The words take no
arguments.

> **Today's indices.** No published index states what a reference does, and
> none records `declares` or `holds` as relationships of their own yet. Until
> the indexers write them, a role word answers "the index states no roles"
> and `declares` and `holds` find nothing. See
> [what today's indices state](#what-an-index-states) before relying on an
> empty answer.

### has (Containment)

Selects containment: the parent contains the child.

```askl
mod("pkg") has { func }        /* Functions the module contains */
file has { "foo" { "bar" } }   /* a file contains foo; the scope under foo is the default again */
```

Containment is read from the depth-first interval the indexer records for every declaration — not from byte ranges or symbol types, which cannot tell apart what a macro generates. See [Containment](/docs/design/containment/).

### refs (Ordinary references)

Selects the references a token makes: a call, a variable used, a type named in a declaration, a macro invoked.

```askl
"foo" refs { "bar" }  /* foo names bar */
"foo" { "bar" }       /* any relationship — the same answer for two functions */
```

`refs` is one of four relationships, not "everything that is not containment":
it leaves out what a macro declares and what a slot holds. Each scope says
what it asks for:

```askl
has { "foo" refs { "bar" } }
/* foo found via containment, bar found via references */
```

### declares and holds {#declares-holds}

Two relationships are recorded by no token of their source:

- **`declares`** — an invocation that supplies a declaration's name declares
  it. `DEFINE_MUTEX(lock)` makes `DEFINE_MUTEX ↦ lock`
  ([what a macro declared](/docs/c-indexing/#declares)).
- **`holds`** — a dispatch slot and a function stored in it. `.read = my_read`
  makes `file_operations.read ⇢ my_read`; a Go interface method holds the
  methods that implement it ([dispatch](/docs/dispatch/)).

```askl
macro("DEFINE_MUTEX") declares { data }        /* what the macro generates */
macro("DEFINE_MUTEX") refs { }                 /* what its body names */
field("file_operations.read") holds { func }   /* the functions stored in the slot */
```

Only a macro declares and only a slot holds. Asked of anything else the scope
is empty, and says why. `"vfs_read" holds { func }`:

```
`holds` related nothing: only a slot holds (a field, or an interface method), and `"vfs_read"` selects none
```

### Role words: calls, writes, takes {#role-words}

A reference can state what its token does to the target. The report prints
those words after the type (`→ handler  (func, takes)`), and the same words
ask for them:

| Word | The token… |
|------|------------|
| `calls` | is the callee of a call, including a call through a struct field |
| `writes` | is assigned or initialised |
| `takes` | is used without being called or written: a function's address passed, stored or returned |
| `reads` | the same as `takes`, as it reads for data and fields: the value is used |

`takes` and `reads` are one role with two spellings. One token can state
several roles: `counter++` both writes and reads `counter`, and either word
finds it.

```askl
func calls { field("platform_driver.probe") }    /* who calls through the slot */
writes { field("platform_driver.probe") }        /* who assigns it */
func takes { "my_handler" }                      /* who passes the function around */
```

A role word narrows `refs`: it never matches containment, a `declares` or a
`holds`. Three laws keep its answers honest:

- **An unstated role is unknown, never "no".** A reference that states nothing
  is matched by no role word. An index written before roles existed states
  nothing at all; in an index that does state them, some references still
  state nothing, for example a type named in a declaration, a macro
  invocation, or a name in a `#define` body that was only read as text.
  Invoking a function-like macro is not a call. The answer says what
  it passed over ([notices](#relationship-notices)), and `refs` lists
  everything.
- **`calls` is literal.** `vfs_read` calls the slot `file_operations.read`;
  it does not call `my_read`. The functions behind the slot are one more hop,
  and the hop is written:

  ```askl
  "vfs_read" calls { field holds { func } }   /* across an ops table */
  "vfs_read" calls { { func } }               /* a bare middle scope crosses it too */
  ```

- **A role is one role.** `writes { "hook" }` does not list a function that
  only calls `hook`.

### Joining relationship words: `or` {#relationship-or}

**Relationship words join with `or`, and a pair matches if any of its edges
does.** Parentheses group; there is nothing else to group with.

```askl
calls or has { "kfree_sensitive" }        /* whoever calls it or contains it */
"vfs_read" calls or holds { func }
"vfs_read" has or refs { }                /* what it contains or refers to */
has or refs or declares or holds { "x" }  /* the default, spelled out */
func or data calls or holds { "x" }       /* two expressions: a type group and a relationship group */
```

The last line is a type expression and a relationship expression side by
side. They never mix inside one expression:

```
`func` is a type, `calls` a relationship: write them side by side: `func calls { … }`
```

**A statement says its relationship once.** Two relationship expressions in
one statement — next to each other or apart — are an error, as two type words
are:

```
join relationship words with `or`: `has or refs`
```

**`and` and `not` are not supported yet.** A pair of symbols can have several
edges — a file contains a function *and* calls it — so "both" and "not" would
have to be decided for the pair as a whole, which one test on one edge cannot
do. They are refused with what to write instead:

```
`and` between relationships is not supported yet: `calls and writes` would have to be decided for the pair as a whole. Ask each in its own query.
`not calls` is not supported yet: name what you want: `writes or takes`, `refs`, `declares`.
```

### What a word passed over: notices {#relationship-notices}

**A word that passed over what the scope would otherwise have answered says
so**, once per statement, in the response's warnings. The notices exist so
that a short or empty answer is not read as a fact about the code.

**The index states no roles.** A role word, in a project whose index states
none. `func calls { field("platform_driver.probe") }` on the kernel today:

```
the `linux` index states no roles; `calls` cannot match there. `refs { … }` lists its references
```

**The index has no such rows.** `declares` or `holds`, asked of a macro or a
slot, in a project whose index records neither.
`macro("DEFINE_MUTEX") declares { data }` on the kernel today:

```
the `linux` index has no `declares` rows: nothing in it records what a macro declares
```

**Only a slot holds, only a macro declares.** Asked of something else:

```
`holds` related nothing: only a slot holds (a field, or an interface method), and `"vfs_read"` selects none
`declares` related nothing: only a macro declares, and `field("file_operations.read")` selects none
```

The next four need an index that states roles or has `declares` and `holds`
rows. No published index does yet ([below](#what-an-index-states)), so they
are quoted from the engine's test index, with its names.

**References that state no role were passed over.** In an index that states
roles, some references still state none. `"caller" calls { }`:

```
`calls` passed over 2 symbols that `"caller"` names without stating a role: `maybe`, `plain`. Whether it calls them is not known; `refs { … }` lists them
```

It counts distinct symbols, names up to three, and counts only function,
field and data targets: for a type or a macro a missing role is not an
omission.

**A slot holds it.** A role word found nothing, and the target is stored in a
slot. `calls { "my_read" }`:

```
nothing calls `"my_read"` directly; `slot` holds it (`⇢`): `calls { field holds { "my_read" } }` goes through the slot
```

**`refs` lists only ordinary references.** `refs` found nothing, and the pair
is related by `holds` or `declares`. `field("slot") refs { "my_read" }`:

```
`refs` lists only ordinary references (`→`); `field("slot")` holds 1 symbol that `"my_read"` matches (`⇢`): `my_read`. `holds { … }` or `{ … }` lists it
```

**A listing passed over a slot.** This one is about a filter, not a
relationship word. `"via" { func }` lists the functions `via` calls, and `via`
also calls through a field, which `func` leaves out together with the
functions stored in it:

```
`func` passed over 1 slot that `"via"` calls through: `slot`; follow it: `"via" calls { field holds { func } }`
```

The query it gives runs as written. It counts only slots that hold something
the listing asks for and does not already list, and it is not said under
`has`, `declares`, `holds`, `writes` or `takes`, where no call is asked for.

Under an `or`, a notice is said of the words it is true of: `calls or refs`
says nothing about unstated roles, because `refs` lists those references
anyway.

### What today's indices state {#what-an-index-states}

What an index states is observed when it is uploaded, per project: whether any
reference states a role, and whether it has `declares` or `holds` rows. **No
published index has any of the three yet**, so today:

- `calls`, `writes`, `takes` and `reads` match nothing, and say
  ``the `linux` index states no roles``. Use `refs`, or the default scope, and
  read the code for which reference is a call.
- `declares` and `holds` find nothing, and say the index has no such rows.
- What a macro declared and what a slot holds are still in the index, recorded
  as ordinary references. So today `refs` and the default scope list them:
  `macro("DEFINE_MUTEX") { data }` is the mutexes, and
  `field("file_operations.read") { func }` the implementations.

When an indexer starts writing these, the same queries change their answers
without changing their text: the role words begin to match, `declares` and
`holds` begin to list, and `refs` stops listing what a macro declared and
what a slot holds (when that leaves a scope empty, the notice
"`refs` lists only ordinary references" says so). **The default
scope lists all four relationships before and after**, so a query written
without a word keeps its answer.

### Traversal modes: `nested` and `unnest` {#traversal-modes}

A scope asks a containment question, and there are three answers it can want.
All three are the same walk over the index's containment tree; they differ only
in what they list. See [Containment](/docs/design/containment/) for the model.

| mode | lists |
|---|---|
| `nested` | exactly one structural level |
| *(default)* | every descendant, at any depth, except folded symbols |
| `unnest` | every descendant |

**The default is not "direct".** Containment is a real chain — module contains
directories, directories contain files, files contain declarations — so a
module's *direct* children are files, and `mod("amdgpu") { func }` would return
nothing. The default instead reaches every level and leaves out one kind of
thing: **folded symbols**.

**A symbol is folded when nobody wrote it down in normal code**, meaning in
text outside every `#define` body. Normal code writes a symbol down by

- declaring it with text of its own — an ordinary function, struct or variable;
- typing its whole name where a macro declares it — `lock` in
  `DEFINE_MUTEX(lock)`;
- typing its name in a reference, or the part of a pasted name a caller
  chooses — `send` in `MCA_PML_CALL(send(buf))`, which names the field
  `pml_send`.

What is left is what the preprocessor made for itself: `__compiletime_assert_123`,
the constructor behind an initializer macro, the fields of a generated struct
that nobody reads, a function reached only through an X-macro list.

```askl
mod("amdgpu") { func }                         /* the functions people wrote or use */
mod("amdgpu") { unnest func }                  /* those, and everything macros generated */
mod("amdgpu") { "__compiletime_assert_123" }   /* a named symbol is always found */
```

Folding is a property of the *symbol*, decided over the whole project: a
`kh_put(tbl, h, k)` in one file unfolds `kh_put_tbl`, which a
`KHASH_INIT(tbl, …)` in another file generated. An index whose producer never
folds anything — Go has no preprocessor — lists everything.

**Folding hides; it does not disconnect.** It decides what is *listed* and
plays no part in what contains what. In

```c
void f(void) { BUILD_BUG_ON(sizeof(struct x) > 4); }
```

the macro generates a declaration that sits inside `f`, and the `struct x`
typed in its argument sits in `f` too: whoever wrote the invocation typed it,
and the compiler read it there. `"f" { }` lists `x`, and the generated
declaration is simply absent from the list. Upward is the same: `{ "x" }` lists
`f`, its file and its directory. A folded declaration that *does* contain
something — a generated `kh_put_x` that nobody calls by name, using the
`my_hash` handed to `KHASH_INIT(x, int, my_hash)` — is passed over the same
way, and the file still lists `my_hash`.

**A listing says what it left out.** Containment survives folding; a call
chain does not. `SYSCALL_DEFINE3(read, …)` generates `__x64_sys_read`, which
calls `__se_sys_read`, which calls `__do_sys_read` — and nobody ever types the
middle name, so it is folded. `"__x64_sys_read" { }` therefore lists no callee,
and `{ "__do_sys_read" }` no caller. So that this does not read as "calls
nothing", a default listing that passed over folded symbols says so in its
warnings:

```
`{ }` leaves out 1 folded (macro-generated) symbol: `__se_sys_read`. Add `unnest` to list it, or name it
```

```askl
"__x64_sys_read" { unnest }          /* the hop is listed */
"__x64_sys_read" { unnest { } }      /* and what it calls */
unnest { "__do_sys_read" }           /* upward: on the statement that lists */
```

The warning counts symbols, names up to three, and reports only what the
statement would otherwise have listed: `file("/a.c") has { func }` mentions the
folded functions of the file and not its folded types. A listing that a limit
cut says it is [incomplete](#incompleteness) instead, and is not asked what
else it left out.

**Naming a symbol ignores folding.** A statement that names what it wants —
exactly or by glob — is not an enumeration, so `"f" { "__compiletime_assert_123" }`
returns the declaration although `"f" { }` does not list it.

**What a macro calls is reached by edges, not by folding.** Macro edges are
*layerwise*: each layer records only what its own body names, so

```c
#define DEV_PRINT(x) do { printk(x); } while (0)

void handler(void) { DEV_PRINT(threshold); }
```

gives three edges — `handler -> threshold`, `handler -> DEV_PRINT`, and
`DEV_PRINT -> printk` — and `handler -> printk` is a two-hop traversal, exactly
as it would be for a chain of functions. A wrapper around a wrapper chains the
same way: `handler -> FATAL -> ERR -> MSG -> output`, one edge per `#define`.

```askl
macro("DEV_PRINT") { "printk" }        /* what the macro's body calls */
func("handler") { macro }              /* the macros it invokes */
```

**What a macro declares is an edge from the macro.** An invocation that
supplies a declaration's name declares it, so

```c
static DEFINE_MUTEX(lock);
struct hooks { HOOK(struct ops_ret, open); };
```

gives `DEFINE_MUTEX -> lock` and `hooks -> HOOK -> hooks.open -> ops_ret`. The
variable does not lead to the macro; the file that wrote the invocation does.

```askl
macro("DEFINE_MUTEX") { }             /* every mutex declared with it */
{ "lock" }                            /* its users, and DEFINE_MUTEX */
macro("UCS_PROFILE_FUNC") { func }    /* the functions declared through it */
macro("KHASH_INIT") { unnest }        /* including the ones nobody calls by name */
```

The two kinds of edge a macro has are two relationships: what its invocations
declared is [`declares`](#declares-holds), what its body names is `refs`. The
default scope lists both.

```askl
macro("DEFINE_MUTEX") declares { data }   /* only what it declared */
macro("DEFINE_MUTEX") refs { }            /* only what its body names */
```

Published indices record a declaration as an ordinary reference for now, so
today the default scope is the one that lists the mutexes
([what today's indices state](#what-an-index-states)).

A macro that only declares a symbol *again* does not lead to it:
`EXPORT_SYMBOL(foo)` has no edge to `foo`. A macro is one node, so a path
through it joins invocations that have nothing to do with each other. See
[C indexing](/docs/c-indexing/#declares).

A name the macro *pastes together* belongs to whoever completed it. With
`#define CALL(a) ops.do_ ## a`, the caller of `CALL(send(x))` chose `send`, so
the edge to `ops.do_send` is the caller's, direct, and `CALL` has none. See
[C indexing](/docs/c-indexing/) for the rule.

**`nested`** narrows to exactly one structural level, and ignores folding. It
is how to ask for a container's own contents and nothing deeper:

```askl
file("main.c") { nested }              /* the file's own declarations, nothing deeper */
```

Because it is literal about levels and ignores folding, in the `BUILD_BUG_ON`
example `"f" { nested }` lists the generated declaration as well as `x`: both
sit directly in `f`. It does not look *inside* a macro either. A generated
declaration is placed where the compiler read it, not under the macro, so
`macro("X") { nested }` returns `X`'s edges: what its body names and what its
invocations declared, folded or not.

**`unnest`** lists everything, folded or not:

```askl
func("main") { unnest }             /* everything inside main */
```

Upward derivation uses the same knob. A bare `{ "x" }` surfaces every container
of `x` whose symbol is not folded; `nested` asks for just the one container `x`
literally sits in, and `unnest` for all of them:

```askl
{ "data_d" }            /* func_a, its file, its directory */
nested { "data_d" }     /* func_a */
```

> **Where the modifier goes.** A traversal mode belongs to the statement whose
> reach it bounds. For downward traversal that is the statement *inside* the
> braces — `file("main.c") { nested }`, not `file("main.c") nested { }` — and
> for upward derivation it is the statement that owns the scope. Modes do not
> inherit into child scopes, and they do not combine: the last one written wins.

### any (Any Symbol Type)

`any` is the type verb that constrains **no** type. It behaves exactly like
`func`, `data` or `mod` — the same two forms — except that the type it asks
for is "all of them".

**Bare `any`** is a filter. A scope with no type word already accepts every
type, so its one job is to cancel a type that `inherit(type)` carried down:

```askl
data inherit(type) { any { "bar" } }   /* Inner scopes match all types, not just data */
```

**`any("name")`** is a selector, and matches that name at whatever type carries
it — where `func("ioctl")` finds only the function, `any("ioctl")` also finds
the macro and the struct:

```askl
any("ioctl")               /* the name, at every type */
any(g"ioctl_*")            /* globs work here as anywhere else */
func { { func "bar" } }     /* an untyped intermediate hop needs no `any` */
```

> **Note:** bare `any` denotes the whole index, so — like bare `func` — it is a
> constraint rather than a reason for a statement to exist: `any` on its own is
> rejected by the [anchor rule](/docs/design/planning/#anchors). Use `any select`
> to enumerate.

### Carrying a word into nested scopes: `inherit` {#inherit}

**A word governs the statement it is written in; `inherit(kind)` carries it.**
A relationship word describes the edge into its own statement's braces, and a
type word filters its own statement. Neither reaches the scopes nested below
unless the statement says so:

```askl
file has { "foo" { "bar" } }
/* the file contains foo; the scope under foo is the default: foo refers to bar, or contains it */

file has inherit(relationship) { "foo" { "bar" } }
/* has is asked under foo too: foo must contain bar */

"a" calls inherit(relationship) { "b" { "c" { "d" } } }   /* a four-deep call chain */
"a" calls { "b" calls { "c" calls { "d" } } }             /* the same, written out */

data inherit(type) { { "bar" } }                          /* both inner scopes are data too */
```

- `inherit(relationship)` carries the statement's relationship — a word or an
  `or` of them — into every scope below, **until a statement writes its own**.
  That statement's word is then the one in force for its braces, and it is
  carried further only if that statement says `inherit(relationship)` too.
- `inherit(type)` carries the statement's type the same way: a type word, a
  named type selector, `filter("type", …)` or a type group
  (`func or method inherit(type)`).
- Carrying both is two verbs:
  `has inherit(relationship) data inherit(type) "x" { { } }`.
- Like every term, `inherit(...)` acts on the statement it is written in and
  may stand anywhere in it, before or after what it carries.
- With nothing to carry it changes nothing: `"foo" inherit(type) { }` is
  `"foo" { }`.
- It copies words into braces that exist. It never adds a hop.
- A [preamble](#preamble-carries) carries what it writes without being told
  to.

The kind is one of two bare words, and the errors name them:

```
`inherit` needs what to carry: write `inherit(relationship)` or `inherit(type)`
`inherit` carries a statement's relationship or its type, not `expression`: write `inherit(relationship)` or `inherit(type)`
the kind is a word, not a string: write `inherit(type)`
`inherit` carries one kind: write `inherit(relationship) inherit(type)` to carry both
```

Traversal modes (`nested`, `unnest`) are not carried by either kind; they
belong to the one statement they are written on.

## Generic Verbs

### select (Binding Enumeration)

`select` takes no arguments. It makes the statement **binding** — "I want
instances from this" — which does three things at once: it anchors the
component (so the anchor rule is satisfied), it makes its own command
constraining in composition (never a weak echo), and it enumerates
everything the command's filters allow, bounded by the result budget with
a truncation warning.

```askl
select                       /* Everything visible, budget-bounded */
func select                  /* All functions, budget-bounded */
project("linux") select      /* Everything in project linux, budget-bounded */
select filter("compound_name", "test")  /* Everything under namespace test */
```

### Name selection (quoted strings)

Selecting by name is written directly as a quoted string — there is no
functional form:

```askl
"main"          /* Symbols whose last name component is "main" */
"http.Handler"  /* Symbols with both "http" and "Handler" in their path */
g"user.*ate"    /* Glob pattern (see Pattern Matching above) */
```

### loc (Location Anchor)

Creates an ephemeral symbol at a specific file location — an anchor for
"start from this line":

```askl
loc("main.c", 42)                    /* file path (suffix match), 1-based line */
loc("main.c", 42, project="linux")   /* restricted to one project */
loc("drm_drv.c", 120) { func }       /* what does this line's code call? */
```

The path is a string; the line is a bare integer ≥ 1 — `loc("main.c", "42")` is rejected.

### filter (Generic Filter)

A generic filter verb supporting multiple filter kinds.

```askl
filter("type", "func") "main"       /* Constrain a name search to functions */
filter("type", "func") select        /* All functions (type filter + select) */
filter("compound_name", "main")     /* Namespace/compound-name anchor */
filter("exact_name", "/src/main.go") /* Exact-name anchor */
```

**Filter kinds:**
- `"type"`: Filter by symbol type (`"func"`, `"type"`, `"data"`, `"macro"`, `"field"`, `"method"`, `"mod"`, `"file"`, `"dir"`)
- `"compound_name"`: Filter by compound name pattern (token matching)
- `"exact_name"`: Filter by exact symbol name

**Parameters:** `inherit` — boolean, default `false`; `filter("compound_name", "main", inherit=true) { … }` carries the constraint into nested scopes. For a type filter, [`inherit(type)`](#inherit) beside it does the same.

### ignore (Symbol Filtering)

Excludes symbols matching a pattern from current and nested statements.

```askl
ignore("test") "main" {}        /* Ignore test functions */
ignore("builtin") ignore("fmt") "process" {}  /* Multiple ignores */
```

`ignore` and `not` coincide on simple names but are not aliases: `not "x"`
negates the leaf name only, while `ignore("x")` excludes any symbol whose
path carries the label `x` — `"foo.bar" ignore("foo")` is empty,
`"foo.bar" not "foo"` keeps it. A plain string stays literal inside
`ignore`, so a wildcard needs a glob: `ignore(g"kl*", package="k8s.io/klog")`
is `not (g"kl*" and package("k8s.io/klog"))` — see
[Boolean Operators](#boolean-operators). Like all exclusions, ignores
accumulate and inherit into nested scopes.

### preamble (Global Configuration)

Applies verbs to the global scope, affecting all subsequent statements. Use scope syntax `{ }` to group multiple preamble verbs across lines:

```askl
preamble {
    ignore("builtin")
    ignore("test")
}

"main"  /* This and all following queries ignore builtin and test */
```

Single-line syntax also works, but every verb must be on the `preamble` line —
a newline ends the statement, so verbs on the next line form a statement of
their own rather than joining the directive:

```askl
preamble ignore("builtin") ignore("test")
```

#### A preamble carries what it writes {#preamble-carries}

In a statement, a relationship word and a type word stay in that statement
unless [`inherit(...)`](#inherit) carries them. A preamble is not a statement
and has no braces of its own, so what it writes is written for **every
statement after it, at every depth**, with no `inherit(...)`:

```askl
preamble has
file("/main.go") { func { } }    /* both scopes ask for containment */

preamble func
"main" { { } }                   /* main's callees and theirs, functions only */
```

A statement that writes its own relationship or type replaces the preamble's
from there down, exactly as it replaces one carried by `inherit(...)`.

#### A preamble is a directive, not a statement

A preamble configures the query; it never produces rows of its own. Three
consequences follow, and each is an error rather than a silent surprise.

**It takes constraints and directives, not selectors.** `project(...)`,
`ignore(...)`, `filter(...)` — including the name filters, which scope every
statement — bare type selectors, relationship words and modifiers like `unnest` or `nested` all belong here. A
selecting term does not: it would be applied to every statement rather than
selecting anything of its own.

```askl
preamble project("myproject") "main"   /* error: a preamble cannot select */

preamble project("myproject")          /* write it as its own statement */
"main"
```

**It takes no label plumbing.** A label names *one* statement's result set, and
"every statement" is not one result set — so neither `@name` nor `#name` has
anything here to name or to read. Labels belong on the statements a preamble
configures:

```askl
preamble @lib                          /* error: a preamble is not a statement */

preamble project("myproject")          /* label the statement instead */
func("main") @lib
#lib { }
```

**It must be top level.** Inside a scope, "applies to every statement" has no
reading, so a nested `preamble` is rejected instead of being quietly narrowed to
the block. A `preamble` inside another `preamble` is refused for the same
reason: the outer one already covers every statement.

#### Preambles may repeat, and apply forward

A preamble is not a header, so nothing caps how many a query opens or where
they sit among the top-level statements. **Each one configures the statements
that follow it** — so a second preamble re-scopes the rest of the query, which
is how a single query spans two projects:

```askl
preamble project("ucx")
dir("cuda_ipc") { func { g"cuMemcpy*" } }

preamble project("linux")              /* everything below is Linux */
"drm_xe_vm_bind"
```

Separate preambles write separate parts of the same global command, so
directives on different dimensions **accumulate**:

```askl
preamble project("myproject")
preamble ignore("test")                /* both apply from here on */
"main"
```

Writing the same dimension twice **replaces** it, following the usual
[slot rules](#filters) — a later `project(...)` wins over an earlier one. A
trailing preamble with no statement after it configures nothing, and is
harmless.

### project (Project Filter)

Filters results to a specific project (useful in multi-project setups).

```askl
project("myproject") "main" {}    /* Only symbols from myproject */
```

#### Filtering a project vs. narrowing the request

`project(...)` is a **predicate**: the query runs across every project the
request can see, and keeps the rows belonging to the named one. Which projects
are visible at all is a property of the **request**, not of the query text:

- HTTP: `POST /query?projects=linux,rdma-core`
- MCP: the `projects` argument of `askl_run`

Narrowing there is the cheaper of the two — the other projects stop existing
for that run, instead of being filtered out afterwards — and it leaves the
query text portable across workspaces. Omit it and every indexed project is
visible, as before. An unknown project name is an error rather than an empty
result.

The two compose in the obvious way: narrowing to `linux` and then asking for
`project("rdma-core")` returns nothing, because `rdma-core` is not visible to
that request.

### package (Package Path Filter)

Keeps only symbols strictly under a package path — the positive form of
`ignore(package=...)`. It inherits into nested scopes and composes in
[boolean expressions](#boolean-operators).

```askl
package("k8s.io/klog") "main" {}          /* main, under k8s.io/klog only */
not (g"kl*" and package("k8s.io/klog")) "main" {}
```

### search (Full-text Content Search)

Full-text search over the raw source bytes of every indexed file. Each occurrence of the query becomes an ephemeral symbol anchored at that byte range, and downstream verbs (`{ }`, `has`, `refs`, etc.) can compose with it just like any other selector.

**Signature:**

```askl
search(query, case="smart", whole_word=false, limit=500)
```

- `query` — required positional; the string to search for. **≥ 3 characters.**
- `case` — a string, since it is tri-state: `"smart"` (default), `"sensitive"`, or `"insensitive"`.
- `whole_word` — boolean: `false` (default, substring match) or `true` (word-boundary match).
- `limit` — integer ≥ 1, default `500`, applied **per project**: each visible project contributes at most `limit` matches, and the result carries a truncation warning when any project hit it, so you know to narrow.

> ⚠️ **No regex support.** The query is matched as a *literal string* in every mode. `search("foo.*bar")` looks for the exact seven-character sequence `foo.*bar`, not a regex. `search("[a-z]+")` looks for the seven-character sequence `[a-z]+`. This is the most common surprise for users coming from `grep` / `ripgrep`; document your queries accordingly.

**Smart case (default).** Matches ripgrep's convention: if the query is all-lowercase the match is case-insensitive; if it contains any uppercase character it becomes case-sensitive. So `search("foo")` matches `foo`/`Foo`/`FOO`, but `search("Foo")` matches only `Foo`. To override, pass `case="sensitive"` or `case="insensitive"` explicitly.

**Whole-word vs substring.**

| `whole_word` | Behaviour |
|---|---|
| `false` (default) | `search("foo")` matches `foo`, `foobar`, `xfoo`, `foo_bar` — any substring occurrence. |
| `true` | `search("foo")` matches only occurrences bounded by non-word characters on both sides. Word characters are `[A-Za-z0-9_]`, so `foo_bar` counts as one word (no match on `foo`), but `foo.bar` splits on the dot and both `foo` and `bar` match. |

**Composition examples:**

```askl
project("kubernetes") search("mana_ib_reg_user")
    /* Search only inside the kubernetes project */

search("HandleFunc", case="sensitive")
    /* Case-sensitive substring search */

search("open", whole_word=true) {}
    /* For every occurrence of the word "open", pull its children (via has/refs) */

project("linux")
search("EXPORT_SYMBOL", whole_word=true, limit=2000)
    /* Higher limit for a common macro */
```

See **[search()](/docs/design/search)** for the full architecture — the four SQL variants, the pg_trgm / tsvector pipeline, the byte-offset PL/pgSQL helper, and the correctness invariant across filter compositions. The results `search()` produces live on a per-query **ephemeral layer**; see **[Layers](/docs/design/layers)** for that data model and **[Caching](/docs/design/caching)** for how repeat and composed queries are kept cheap.

### forced (Override Relationships)

Forces display of relationships that don't exist in the actual code.

**Syntax:**
```askl
"parent" {
    !"forced_child"
}
```

**When to use:**
- Function pointer calls not detected by analysis
- Complex runtime relationships
- Simplified architectural views
- Working around analysis limitations

## Labels and Reuse

### label (Define a Label)

Labels a statement for later reuse with `use` (or the `#` shortcut).

**Long form:**
```askl
label("handlers") "handler" {}
```

**Shortcut syntax:**
```askl
@handlers "handler" {}
```

The `@` prefix is shorthand for `label("...")`. Use `@@` for inheritable labels:

```askl
@@handlers "handler" {}
/* Equivalent to: label("handlers", inherit=true) "handler" {} */
```

### use (Reuse a Label)

References a previously labeled statement.

**Long form:**
```askl
label("handlers") "handler" {}
"main" { use("handlers") }    /* Reuse the handlers selection */
```

**Shortcut syntax:**
```askl
@handlers "handler" {}
"main" { #handlers }    /* Reuse the handlers selection */
```

The `#` prefix is shorthand for `use("...")`.

**Forced usage:**

```askl
label("handlers") "handler" {}
"main" { use("handlers", forced=true) }  /* Force the relationship */
```

### Ordering: labels reference earlier statements

A label may only be referenced from a **later** top-level statement:
the statement that defines the label must come earlier in the query
(statements are separated by `;` or a newline). For a label consumed
by a layer-creating verb — e.g. a `@label` referenced inside a
`layer { … }` block's ephemeral operations — this is enforced at parse
time: a same-statement or forward reference is rejected, with a hint to
split the query with `;` so the labelled statement comes first.

```askl
@handlers "handler" {} ; "main" { #handlers }   /* OK — label defined earlier */
```

The rule exists because top-level statements are the query's only time
axis: each statement materialises its ephemeral layers atomically
against the visibility left by earlier statements, and **nesting does
not sequence materialisation** — everything inside one statement runs
against the same pre-statement state, so only an earlier statement's
label names a selection that is guaranteed complete.

## Shortcuts

Askl provides shortcut syntax for labels and reuse, using the `@` and `#` prefixes:

| Shortcut | Equivalent | Description |
|----------|-----------|-------------|
| `@foo` | `label("foo")` | Labels the current statement as "foo" |
| `@@foo` | `label("foo", inherit=true)` | Labels with propagation to child scopes |
| `#foo` | `use("foo")` | Uses the labeled statement's results |

**Example:**
```askl
@handlers func("Handle") {}
"main" { #handlers }

/* Equivalent long form: */
label("handlers") func("Handle") {}
"main" { use("handlers") }
```

> **Note:** Comments are `/* ... */` or `//` to the end of the line. `#` is the use-shortcut, not a comment.

## Query Examples

### Basic Call Graph

```askl
"main" {{}}  /* main and two levels of callees */
```

### Who Calls This Function?

```askl
{ "targetFunction" }  /* All callers of targetFunction */
```

### Module Contents

```askl
mod("mypackage") { func }  /* All functions in module */
```

### Type Queries

```askl
type("Request")             /* Find a type by name */
type("Request") { type }   /* Types referenced by Request */
mod("net/http") { type }   /* All types in a module */
```

### Data Queries

```askl
data("Debug")               /* Find a data symbol by name */
func("main") { data }      /* Data symbols referenced by main */
mod("config") { data }     /* All data symbols in a module */
```

### Directory Contents

```askl
dir("/src") { file }       /* All files in /src */
dir("/src") {}             /* Directories and files in /src */
```

### Mixed Relationships

```askl
mod("pkg") { "handler" {{}} }
/* Module pkg, functions named "handler" within it, and their call graph */
```

### Filter by Project and Type

```askl
project("backend") mod("api") { func("Create") }
/* Create functions in api module of backend project */
```

### Exclude Test Code

```askl
preamble ignore("test") ignore("mock")
"main" {{}}  /* Call graph excluding test/mock code */
```

### Asking by Relationship

These ask for one [relationship](#relationship-modifiers) instead of all four.
The ones with a role word, `declares` or `holds` need an index that records
them; on [today's indices](#what-an-index-states) they answer with a notice
that says so, and the default scope beside each is what lists the symbols now.

```askl
/* Who calls through a slot, and who assigns it */
func calls { field("platform_driver.probe") }
func writes { field("platform_driver.probe") }
func { field("platform_driver.probe") }          /* today: both, together */

/* What a macro declares, and what its body names */
macro("DEFINE_MUTEX") declares { data }
macro("DEFINE_MUTEX") refs { }
macro("DEFINE_MUTEX") { data }                   /* today: the mutexes */

/* A call across an ops table */
"vfs_read" calls { field holds { func } }
"vfs_read" { field { func } }                    /* today: the same chain */

/* A call chain, four deep */
"a" calls inherit(relationship) { "b" { "c" { "d" } } }

/* Only the contents, at every level */
file("/main.go") has inherit(relationship) { func { } }
```

## Scope Types

### Callee Scope

Shows what the parent symbol calls:

```askl
"parent" { "child" }  /* parent calls child */
```

### Caller Scope

Shows what calls the nested symbol:

```askl
{ "target" }  /* Functions that call target */
```

### Nested Scopes

Multi-level relationships:

```askl
"a" { "b" { "c" } }  /* a calls b, b calls c */
```

### Sibling Statements in a Scope

A scope may hold several statements, separated by `;` or a newline. They
**conjoin**: the parent survives only where it relates to *each* of them.

```askl
func("handler") { "validate" ; "log" }
/* handlers that call BOTH validate and log */
```

Two selectors written in **one** statement disjoin instead — that is the whole
difference between the two spellings:

```askl
func("handler") { "validate" "log" }
/* handlers that call EITHER validate or log */
```

A sibling that matches nothing empties the conjunction, exactly as a lone child
would: `func("handler") { "validate" ; "typo" }` returns nothing at all. The
parent need not have a name: `func { "validate" ; "typo" }` returns nothing
too, because a bare `func` between two constraining siblings joins them into
one query — the functions calling both — rather than leaving two independent
answers.

The warnings point at the cause. A name that matches nothing gets its own
no-match warning and nothing else is reported. When every sibling matches but
nothing relates to all of them, one note says so, for example
`` `func` matched symbols, but none of them also relate to `"e"` ``.

A `;` between **top-level** statements is not this operator. It ends the
statement rather than adding a condition; both statements run and their results
are unioned into one graph:

```askl
func("handler") { "validate" } ; func("handler") { "log" }
/* handlers calling validate, plus handlers calling log */
```

## Result Limits and Truncation {#result-limits}

Askl bounds work and responses at every level, and always says so when a
bound was hit. Two different kinds of limit exist, and they mean
different things:

### The result budget (per command) {#result-budget}

Every **command** gets its own result budget — by default 100 distinct
symbols. Commands never compete for one shared budget: in

```askl
mod("amdgpu") { func };

"sysfs_remove_file_from_group"
```

the `func` command's hundreds of derived symbols are capped to its own
100, while the `"sysfs_remove_file_from_group"` command keeps its one
anchored symbol — a dedicated selector always shows its results, no
matter how large a neighbouring command's fan-out is. When any command
overflows its budget, the response's `truncated` flag is set and the kept
symbols are chosen deterministically (by file path, offset, and id).

The verbs documented as *budget-bounded* elsewhere on this page —
`select` and bare enumerations — read no more than a multiple of this
budget and are then capped to it. `search` is bounded by its own
per-project `limit=` instead; its matches meet the per-command symbol cap
only when the response is assembled.

#### Raising the limits: the `limits()` verb {#limits-verb}

`limits(symbols=N, listing=M)` overrides the defaults on the command it
sits on, and inherits into nested scopes (pass `inherit=false` to stop
that). At least one of `symbols` or `listing` is required; `0` means
unlimited.

- `symbols` — distinct symbols (nodes) the command's selection may ship.
- `listing` — rows in any single per-node listing: instances per node
  and distinct out-edges per node. An edge's call sites are never capped;
  the UI pages them from the edge endpoint. See
  [per-node listings](#paged-listings).

```askl
mod("amdgpu") limits(symbols=300) { func }
/* the module and its <=300 functions, instead of the default 100 */

func("handler") limits(listing=200)
/* show up to 200 instances and 200 out-edges on each handler node */
```

Because it rides in the query text, a shared `#q=` link carries its own
limits. A child command's own `limits()` wins over an inherited one, per
field.

### Discovery caps and the incompleteness warning {#incompleteness}

Separately from the result budget, every SQL read the engine performs is
capped for safety. When a statement's *discovery* read hits that cap —
the engine could not even enumerate everything that matches — the result
carries the warning:

```text
too many results in `{ "printk" }`: this part of the graph is incomplete.
Narrow it with a name or glob, a type filter, `ignore(...)`,
`project(...)`, or fewer hops
```

The warning quotes the statement it is about; a query can earn several.
The response is then marked `incomplete`, and every count in it is a lower
bound: the report prints "101 of at least 916 symbols", never "of 916".

This is stronger than budget truncation: the symbol set itself may be
missing entries, and a capped statement's partial answer is never used
to narrow its neighbours. The fix is what the warning says — a narrower
query — not pagination.

The warning is said only when a limit cut something out of the answer.
What the listing leaves out for what it is does not count: on the kernel
`macro("EXPORT_SYMBOL_GPL") { }` lists one symbol and says nothing,
although the macro also refers to 20,817 [folded](#traversal-modes)
ones. Ask for those with `unnest` and the same query is cut at the cap,
and says so.

### Per-node listings are paged, not warned {#paged-listings}

Once the symbol set is settled, what each node *carries* (its instance
list, its edge listing) is capped for payload size and marked truncated
in the response (`instances_truncated`, `edges_truncated`). That is an
invitation to fetch more — the UI shows "More references" / "More
instances" in the node popup, and clients can page the
[references and instances endpoints](/docs/api#references) directly. It
never means the graph is wrong.

## Query Rules and Validation

### Required Elements

Every component (a top-level statement with its nested scopes, plus any
label-linked ones) must contain at least one anchor at some nesting level.

**Valid examples:**
```askl
"foo" {}            /* Direct selection */
{"foo"}             /* Selection in scope */
{{{"foo"}}}         /* Deeply nested selection */
```

**Invalid examples** (rejected with a hint naming the fix):
```askl
func                /* Constraint only: which functions? */
project("linux")    /* Constraint only: everything in linux is not a selection */
func { file }       /* A whole component of constraints */
"a"; func           /* Second component has no anchor (per-component rule) */
```

Degenerate structure without any constraints (`{{{{}}}}`) is *legal* and
simply empty — only components that constrain something are held to the
anchor rule.

### Scopes Filter Results

Scopes only show results if the actual relationship exists. If `parent` doesn't call `child`, no results are displayed:

```askl
"parent" { "child" }  /* Empty if parent doesn't call child */
```
