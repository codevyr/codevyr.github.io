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

Askl supports eight symbol types organized in a hierarchy:

| Type | Level | Description |
|------|-------|-------------|
| `field` | 1 | Struct fields, interface methods (dispatch points) |
| `function` | 2 | Functions, methods, procedures |
| `type` | 2 | Structs, interfaces, type declarations |
| `data` | 2 | Package-level variables, constants |
| `macro` | 2 | C/C++ preprocessor macros (`#define`) |
| `file` | 3 | Source files |
| `module` | 4 | Packages, modules, namespaces |
| `directory` | 5 | Filesystem directories |

Higher-level symbols can **contain** lower-level symbols (e.g., a module contains files, which contain functions). Fields are the lowest level — they can be contained by types, functions, files, and modules. Functions, data, types, and macros share the same level.

### 3. Relationships

Askl supports two types of relationships between symbols:

#### Reference Relationships (`refs`, default)

Shows what symbols **call** or **reference** each other. This is the default relationship type.

```askl
"foo" { "bar" }        /* Functions foo calls (references) */
"foo" refs { "bar" }   /* Explicit refs (same as above) */
```

#### Containment Relationships (`has`)

Shows what symbols **contain** other symbols based on source code location.

```askl
mod("mypackage") { func }   /* Functions contained in module */
file("/main.go") { "handler" }  /* handler function in main.go */
```

> **Note:** Container types (`dir`, `file`, `mod`) implicitly set both containment and reference relationships for their children. You don't need explicit `has` when using them — `mod("pkg") { func }` works directly.

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
yes/no; `derive(type="refs,has")` stays a string because it is a list.

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
their matches, constrained by the statement's filters. Inside a scope that is exactly
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
| `ignore("pattern")` | Exclude symbols matching the pattern (alias for `not "pattern"`) |
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
| `any` (no name) | Include every symbol type (removes an inherited type filter) |

> **Note:** Type selectors like `func` behave differently based on arguments:
> - Without a name (`func`) → **Filter** (constrains to type, **inherits to all descendants**)
> - With a name (`func("foo")`) → **Selector** (queries matching symbols, does NOT inherit)
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
| `has` | Use containment relationships instead of references |
| `refs` | Use reference relationships (default) |
| `derive(type="...")` | Set relationship type with options |
| `preamble` | Apply subsequent verbs to the global scope |
| `label("name")` / `@name` | Label this statement for reuse |
| `unnest` | Include transitive children/references and all containment levels |
| `!` (forced) | Force display of relationships |

> **Weakness is inferred, not written.** There is no weak marker: bare
> `{ }` scopes and commands with only filters are non-constraining
> *echoes* by default, and `select` is what makes a command constraining.
> See [Weakness and Bindness](/docs/design/semantics/#weakness-and-bindness).

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
relationship verbs (`has`, `refs`, `derive`, `unnest`) and labels are not
predicates; `search(...)`, `loc(...)`, layer literals, and `!"..."` are
not a single predicate query; bare `any` and `select` constrain nothing.
Inside an expression a verb contributes only its predicate — `file or dir
{ … }` does not switch the scope to containment; write `has` explicitly.

**Three rules keep expressions predictable:**

- Per `or`, operands are either all anchors or all filters.  An
  all-anchor group is ONE branch: one query and one no-match warning for
  the whole disjunction — `"a" or "b"` says *either is fine*, while
  juxtaposed `"a" "b"` keeps a warning per name (*each should exist*).
- An expression without an anchor must constrain a single dimension
  (type, project, package, or one filter kind).  Filters of different
  dimensions conjoin by juxtaposition; for a cross-dimension disjunction,
  split into sibling statements.
- A filter group inherits into `{ }` scopes **as a unit**, and a child
  statement writing the same dimension replaces it wholesale:
  `func or method { data "x" }` finds `x` as data, not as an always-empty
  conjunction.  Exclusions (`not ...`) accumulate instead — they narrow,
  like `ignore`, and inherit unconditionally.

> **Footgun:** `func ("foo")` — with a space — is still an argument list,
> not `func` AND a grouped name: a `(` after a verb name binds as
> arguments.

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

Selects function symbols. Explicitly sets the relationship to **references only** — this overrides any inherited containment from container parents.

```askl
func("handler")         /* Functions matching "handler" */
func("http.Handler")    /* Functions matching both "http" and "Handler" */
func select             /* All functions (budget-bounded) */
file("/main.go") { func }  /* Functions in main.go (filter mode) */
```

### type

Selects type symbols (structs, interfaces, type declarations). Like `func`, explicitly sets the relationship to **references only**.

```askl
type("Request")          /* Types matching "Request" */
type("http.Request")     /* Types matching both "http" and "Request" */
type select              /* All types (budget-bounded) */
mod("net/http") { type }  /* Types in module (filter mode) */
type("Request") { type }  /* Types referenced by Request */
```

**Default child types:** types.

### data

Selects data symbols (package-level variables and constants). Like `func` and `type`, explicitly sets the relationship to **references only**.

```askl
data("Debug")            /* Data symbols matching "Debug" */
data("config.Debug")     /* Data symbols matching both "config" and "Debug" */
data select              /* All data symbols (budget-bounded) */
mod("config") { data }   /* Data symbols in module (filter mode) */
```

**Default child types:** data.

### macro

Selects macro symbols (C/C++ preprocessor `#define` directives). Like `func` and `type`, explicitly sets the relationship to **references only**. Macros are indexed with their body range, so function calls inside a macro body become children of the macro by offset containment.

```askl
macro("LOG")              /* Macros matching "LOG" */
macro select              /* All macros (budget-bounded) */
func("main") { macro }   /* Macros referenced by main */
macro("LOG") { func }    /* Functions called inside LOG's body */
```

**Default child types:** macros and functions.

### field / method

Selects field symbols — struct members that act as function pointer dispatch points (C) or interface method signatures (Go). `method` is an alias for `field`. Like `func`, explicitly sets the relationship to **references only**.

Field names use compound naming: `struct_name.field_name` (e.g., `file_operations.read`). A simple name like `field("read")` matches any field whose last component is "read" across all structs, while `field("file_operations.read")` uses pattern matching for a precise match.

```askl
field("read")                      /* Fields matching "read" (broad) */
field("file_operations.read")      /* Precise match */
method("Read")                     /* Interface methods matching "Read" (Go) */
func("vfs_read") { field("read") { func } }  /* Full dispatch chain */
type("file_operations") has { field }         /* All fields of a struct */
{ field("file_operations.read") }             /* Who calls through this field? */
```

**Default child types:** functions.

### mod

Selects module/package symbols. Implicitly sets **refs+has** for children, so contained symbols are found without explicit `has`.

```askl
mod("util")              /* Modules matching "util" */
mod("k8s.io/api")        /* Modules matching the pattern */
mod select               /* All modules (budget-bounded) */
mod("pkg") { func }      /* Functions in module (no has needed) */
```

**Default child types:** modules and functions.

### file

Selects file symbols. Implicitly sets **refs+has** for children.

```askl
file("main.go")          /* Files whose last component is "main_go" (leaf match) */
file("/src/main.go")     /* Exact path match */
file select              /* All files (budget-bounded) */
dir("/src") { file }     /* Files in /src directory */
```

**Default child types:** functions and modules.

### dir

Selects directory symbols. Implicitly sets **refs+has** for children.

```askl
dir("cmd")               /* Directories named "cmd" (leaf match) */
dir("/src")              /* Exact path match for /src */
dir select               /* All directories (budget-bounded) */
dir("/") { file }        /* Files in root directory (no has needed) */
dir("/") {}              /* Shows directories and files (default child types) */
```

**Default child types:** directories and files.

### Default Type Inheritance

At root level (no parent type selector), the default is **all types** — no filtering is applied. When a bare type selector is used, it sets default child types for its scope and **inherits** to all descendants:

| Type Selector | Default Child Types | Inherits |
|---------------|-------------------|----------|
| *(root level)* | all types | — |
| `func` | functions | yes |
| `type` | types | yes |
| `data` | data | yes |
| `macro` | macros, functions | yes |
| `field` / `method` | functions | yes |
| `mod` | modules, functions | yes |
| `file` | functions, modules | yes |
| `dir` | directories, files | yes |

Bare type selectors (without a name) inherit by default — their type filter propagates into all descendant scopes. Named type selectors like `func("foo")` do **not** inherit.

```askl
mod("mypackage") {}      /* Children include modules AND functions */
mod("mypackage") { func }  /* Children explicitly filtered to functions only */
dir("/") {}              /* Children include directories AND files */
data { { "bar" } }       /* data filter inherits — inner scope also filters to data */
data { any { "bar" } }   /* any removes inheritance — inner scope matches all types */
```

### Container Types and Implicit Relationships

Container types (`dir`, `file`, `mod`) automatically set both containment and reference relationships for their children, with inheritance. This means:

```askl
/* These are equivalent: */
dir("/src") { file }
dir("/src") has { file }

/* func overrides back to refs-only: */
dir("/") { func("main") { "bar" } }
/* func explicitly sets REFS, so "bar" is found via call graph, not containment */
```

## Relationship Modifiers

### refs (References)

Explicitly use reference-based relationships. This is the default. Inherits to all descendants until overridden.

```askl
"foo" refs { "bar" }  /* foo calls bar */
"foo" { "bar" }       /* Same - refs is default */
```

Use `refs` to override an inherited `has`:

```askl
has { "foo" refs { "bar" } }
/* foo found via containment, bar found via references */
```

### has (Containment)

Use containment-based relationships. Inherits to all descendants until overridden.

```askl
has { func }              /* Functions contained in parent */
has { "foo" { "bar" } }   /* HAS propagates: bar also found via containment */
```

Containment is determined by source code byte ranges: if symbol A's range contains symbol B's range, and A's type level is higher than B's, then A contains B.

### derive (Relationship Configuration)

Advanced relationship modifier with explicit control over type and inheritance.

```askl
derive(type="has")                  /* Same as has (inherits by default) */
derive(type="refs")                 /* Same as refs (inherits by default) */
derive(type="has,refs")             /* Both containment and references */
derive(type="refs", inherit=false) /* REFS for this scope only, children reset to default */
```

**Parameters:**
- `type`: Comma-separated relationship types (`"has"`, `"refs"`, or `"has,refs"`)
- `inherit`: Whether children inherit this setting, a boolean (default: `true`)

### unnest (Transitive Traversal)

By default, scopes show only **direct** children (for `has`) or **direct** references (for `refs`), and upward HAS derivation returns only the **innermost** parent. The `unnest` modifier removes these restrictions, enabling full transitive traversal through all nesting levels.

```askl
func("main") has { func }           /* Only direct children of main */
func("main") unnest has { func }    /* All transitively nested functions */
```

Without `unnest`, if `main` contains function `foo` which contains function `bar`, only `foo` appears. With `unnest`, both `foo` and `bar` appear.

`unnest` also affects reference traversal:

```askl
func("main") { func }               /* Only refs directly in main's body */
func("main") unnest { func }        /* Refs from main and all nested scopes */
```

For upward (caller/parent) derivation, `unnest` returns all containment levels instead of just the innermost parent:

```askl
{ "inner_symbol" }          /* Only the innermost container */
unnest { "inner_symbol" }   /* All containers at every level */
```

> **Note:** `unnest` does **not** inherit to child scopes. Each statement that needs transitive traversal must use `unnest` explicitly.

### any (Any Symbol Type)

`any` is the type verb that constrains **no** type. It behaves exactly like
`func`, `data` or `mod` — the same two forms, the same inheritance — except that
the type it asks for is "all of them".

**Bare `any`** is a filter, and its job is to cancel an inherited one:

```askl
data { any { "bar" } }     /* Inner scope matches all types, not just data */
func("main") { any }       /* Everything main references, not just functions */
```

Without it, a bare type selector like `data` inherits its type filter to every
descendant. A scope under `any` is unconstrained, so a plain `{ }` beneath it
stays unconstrained too.

**`any("name")`** is a selector, and matches that name at whatever type carries
it — where `func("ioctl")` finds only the function, `any("ioctl")` also finds
the macro and the struct:

```askl
any("ioctl")               /* the name, at every type */
any(g"ioctl_*")            /* globs work here as anywhere else */
func { any { func "bar" } } /* `any` as an intermediate hop */
```

> **Note:** bare `any` denotes the whole index, so — like bare `func` — it is a
> constraint rather than a reason for a statement to exist: `any` on its own is
> rejected by the [anchor rule](/docs/design/planning/#anchors). Use `any select`
> to enumerate.

### Relationship Inheritance

`has`, `refs`, and `derive` all inherit by default — their relationship type propagates to all descendants until explicitly overridden:

```askl
has {              /* HAS for all descendants */
    "foo" {        /* Still uses HAS (inherited) */
        "bar"      /* Still uses HAS (inherited) */
    }
}

has {              /* HAS for descendants */
    "foo" refs {   /* Override to REFS for this scope and below */
        "bar"      /* Uses REFS (inherited from refs override) */
    }
}
```

Container type selectors participate in this inheritance:
- `func`, `type`, `data`, `macro`, `field`/`method` explicitly set REFS, overriding any inherited refs+has
- `mod`, `file`, `dir` set refs+has with inheritance

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

Both positional arguments are quoted; the line must be ≥ 1.

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

### ignore (Symbol Filtering)

Excludes symbols matching a pattern from current and nested statements.

```askl
ignore("test") "main" {}        /* Ignore test functions */
ignore("builtin") ignore("fmt") "process" {}  /* Multiple ignores */
```

`ignore` is an alias: `not "test"` means the same as `ignore("test")`, and
`ignore("kl*", package="k8s.io/klog")` is `not (g"kl*" and
package("k8s.io/klog"))` — see [Boolean Operators](#boolean-operators).
Like all exclusions, ignores accumulate and inherit into nested scopes.

### preamble (Global Configuration)

Applies verbs to the global scope, affecting all subsequent statements. Use scope syntax `{ }` to group multiple preamble verbs across lines:

```askl
preamble {
    ignore("builtin")
    ignore("test")
}

"main"  /* This and all following queries ignore builtin and test */
```

Single-line syntax also works (all preamble verbs must be on the same line):

```askl
preamble ignore("builtin") ignore("test")
```

A preamble takes **constraints and directives, not selectors**. `project(...)`,
`ignore(...)`, `filter(...)` — including the name filters, which scope every
statement — bare type selectors and modifiers like `scope(...)` all belong
here. A selecting term does not: it would be applied to every statement rather
than selecting anything of its own, so it is rejected with an error rather than
silently ignored.

```askl
preamble project("myproject") "main"   /* error: a preamble cannot select */

preamble project("myproject")          /* write it as its own statement */
"main"
```

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
- `limit` — integer ≥ 1, default `500`. Matches above the cap are dropped and the query result carries a truncation warning so you know to narrow.

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

> **Note:** Since `#` is now used for the use-shortcut, comments use `/* ... */` syntax instead.

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
would: `func("handler") { "validate" ; "typo" }` returns nothing at all.

A `;` between **top-level** statements is not this operator. It ends the
statement rather than adding a condition; both statements run and their results
are unioned into one graph:

```askl
func("handler") { "validate" } ; func("handler") { "log" }
/* handlers calling validate, plus handlers calling log */
```

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
