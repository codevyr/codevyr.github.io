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
statement1;
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

Use semicolons to separate consecutive statements:

```askl
"foo";          # First statement
"bar"           # Second statement (semicolon optional at end)
```

Without semicolons, verbs belong to the same statement:

```askl
"foo" "bar"     # Single statement with two selector verbs
```

### 2. Symbol Types

Askl supports four symbol types organized in a hierarchy:

| Type | Level | Description |
|------|-------|-------------|
| `function` | 1 | Functions, methods, procedures |
| `file` | 2 | Source files |
| `module` | 3 | Packages, modules, namespaces |
| `directory` | 4 | Filesystem directories |

Higher-level symbols can **contain** lower-level symbols (e.g., a module contains files, which contain functions).

### 3. Relationships

Askl supports two types of relationships between symbols:

#### Reference Relationships (`@refs`, default)

Shows what symbols **call** or **reference** each other. This is the default relationship type.

```askl
"foo" { "bar" }        # Functions foo calls (references)
"foo" @refs { "bar" }  # Explicit refs (same as above)
```

#### Containment Relationships (`@has`)

Shows what symbols **contain** other symbols based on source code location.

```askl
@mod("mypackage") { @func }   # Functions contained in module
@file("/main.go") { "handler" }  # handler function in main.go
```

> **Note:** Container types (`@dir`, `@file`, `@mod`) implicitly set both containment and reference relationships for their children. You don't need explicit `@has` when using them — `@mod("pkg") { @func }` works directly.

### 4. Pattern Matching

Askl uses exact, case-sensitive token matching on fully qualified names:

- `"cli.Run"` matches symbols containing both "cli" and "Run" tokens
- `"cli"` will NOT match "click" (exact matching)
- Matching works on the complete package path + symbol name

#### Path-based Matching for Files and Directories

File and directory selectors support two matching modes:

| Argument | Matching | Example |
|----------|----------|---------|
| Starts with `/` | Exact path match | `@file("/src/main.go")` |
| No leading `/` | Compound name match | `@file("main.go")` matches any file with "main" and "go" in its path |

## Verb Types

Verbs in Askl fall into three categories:

### Selectors

**Selectors add symbols to the result.** They query the database and produce nodes in the graph.

| Verb | Description |
|------|-------------|
| `"name"` / `@select(name="...")` | Select symbols matching a name pattern |
| `@func("name")` | Select functions matching a name |
| `@mod("name")` | Select modules matching a name |
| `@file("name")` | Select files matching a name |
| `@dir("name")` | Select directories matching a name |
| `@use("label")` | Select symbols from a labeled statement |

Multiple selectors in a statement combine—all must match for a symbol to be included.

### Filters

**Filters constrain the selection without adding symbols.** They remove symbols that don't match criteria.

| Verb | Description |
|------|-------------|
| `@ignore("pattern")` | Exclude symbols matching the pattern |
| `@project("name")` | Only include symbols from a specific project |
| `@filter("kind", "value")` | Generic filter (see below) |
| `@func` (no name) | Only include function symbols |
| `@mod` (no name) | Only include module symbols |
| `@file` (no name) | Only include file symbols |
| `@dir` (no name) | Only include directory symbols |

> **Note:** Type selectors like `@func` behave differently based on arguments:
> - With a name (`@func("foo")`) → **Selector** (queries matching symbols)
> - Without a name (`@func`) → **Filter** (constrains to type, derives from parent)
> - Explicit: `@func(filter="false")` forces selector mode

### Modifiers

**Modifiers change context or behavior** for the current statement or scope.

| Verb | Description |
|------|-------------|
| `@has` | Use containment relationships instead of references |
| `@refs` | Use reference relationships (default) |
| `@derive(type="...")` | Set relationship type with options |
| `@preamble` | Apply subsequent verbs to the global scope |
| `@label("name")` | Label this statement for reuse |
| `!` (forced) | Force display of relationships |
| `?` (weak) | Make statement non-constraining |

## Type Selectors

Type selectors target specific symbol types. As explained above, they act as **selectors** (with a name) or **filters** (without a name).

### Selector vs Filter Behavior

| Syntax | Role | Behavior |
|--------|------|----------|
| `@func("name")` | Selector | Queries functions matching "name" |
| `@func` | Filter | Constrains to functions; derives from parent |
| `@func(filter="false")` | Selector | Queries ALL functions |
| `@func(filter="true")` | Filter | Explicit filter (same as bare `@func`) |

**Why this default?** Querying all symbols of a type is expensive. Inside scopes, filter mode is much more efficient—it derives from the parent's contained symbols instead of querying the entire database.

### @func

Selects function symbols. Explicitly sets the relationship to **references only** — this overrides any inherited containment from container parents.

```askl
@func("handler")         # Functions matching "handler"
@func("http.Handler")    # Functions matching both "http" and "Handler"
@func(filter="false")    # All functions (explicit selector mode)
@file("/main.go") { @func }  # Functions in main.go (filter mode)
```

### @mod

Selects module/package symbols. Implicitly sets **refs+has** for children, so contained symbols are found without explicit `@has`.

```askl
@mod("util")              # Modules matching "util"
@mod("k8s.io/api")        # Modules matching the pattern
@mod(filter="false")      # All modules
@mod("pkg") { @func }     # Functions in module (no @has needed)
```

**Default child types:** modules and functions.

### @file

Selects file symbols. Implicitly sets **refs+has** for children.

```askl
@file("main.go")          # Files matching "main.go" (compound match)
@file("/src/main.go")     # Exact path match
@file(filter="false")     # All files
@dir("/src") { @file }    # Files in /src directory
```

**Default child types:** functions and modules.

### @dir

Selects directory symbols. Implicitly sets **refs+has** for children.

```askl
@dir("cmd")               # Directories matching "cmd" (compound match)
@dir("/src")              # Exact path match for /src
@dir(filter="false")      # All directories
@dir("/") { @file }       # Files in root directory (no @has needed)
@dir("/") {}              # Shows directories and files (default child types)
```

**Default child types:** directories and files.

### Default Type Inheritance

Each type selector sets default child types for its scope:

| Type Selector | Default Child Types |
|---------------|-------------------|
| `@func` | functions |
| `@mod` | modules, functions |
| `@file` | functions, modules |
| `@dir` | directories, files |

```askl
@mod("mypackage") {}      # Children include modules AND functions
@mod("mypackage") { @func }  # Children explicitly filtered to functions only
@dir("/") {}              # Children include directories AND files
```

### Container Types and Implicit Relationships

Container types (`@dir`, `@file`, `@mod`) automatically set both containment and reference relationships for their children, with inheritance. This means:

```askl
# These are equivalent:
@dir("/src") { @file }
@dir("/src") @has { @file }

# @func overrides back to refs-only:
@dir("/") { @func("main") { "bar" } }
# @func explicitly sets REFS, so "bar" is found via call graph, not containment
```

## Relationship Modifiers

### @refs (References)

Explicitly use reference-based relationships. This is the default. Inherits to all descendants until overridden.

```askl
"foo" @refs { "bar" }  # foo calls bar
"foo" { "bar" }        # Same - @refs is default
```

Use `@refs` to override an inherited `@has`:

```askl
@has { "foo" @refs { "bar" } }
# foo found via containment, bar found via references
```

### @has (Containment)

Use containment-based relationships. Inherits to all descendants until overridden.

```askl
@has { @func }              # Functions contained in parent
@has { "foo" { "bar" } }   # HAS propagates: bar also found via containment
```

Containment is determined by source code byte ranges: if symbol A's range contains symbol B's range, and A's type level is higher than B's, then A contains B.

### @derive (Relationship Configuration)

Advanced relationship modifier with explicit control over type and inheritance.

```askl
@derive(type="has")                  # Same as @has (inherits by default)
@derive(type="refs")                 # Same as @refs (inherits by default)
@derive(type="has,refs")             # Both containment and references
@derive(type="refs", inherit="false") # REFS for this scope only, children reset to default
```

**Parameters:**
- `type`: Comma-separated relationship types (`"has"`, `"refs"`, or `"has,refs"`)
- `inherit`: Whether children inherit this setting (default: `"true"`)

### Relationship Inheritance

`@has`, `@refs`, and `@derive` all inherit by default — their relationship type propagates to all descendants until explicitly overridden:

```askl
@has {              # HAS for all descendants
    "foo" {         # Still uses HAS (inherited)
        "bar"       # Still uses HAS (inherited)
    }
}

@has {              # HAS for descendants
    "foo" @refs {   # Override to REFS for this scope and below
        "bar"       # Uses REFS (inherited from @refs override)
    }
}
```

Container type selectors participate in this inheritance:
- `@func` explicitly sets REFS, overriding any inherited refs+has
- `@mod`, `@file`, `@dir` set refs+has with inheritance

## Generic Verbs

### @select (Symbol Selection)

Selects symbols whose names match a specific pattern.

**Full syntax:**
```askl
@select(name="cli.Run")
```

**Shortcut syntax:**
```askl
"cli.Run"
```

**Examples:**
```askl
"main"          # Symbols containing "main"
"http.Handler"  # Symbols containing both "http" and "Handler"
"user.Create"   # Symbols containing both "user" and "Create"
```

### @filter (Generic Filter)

A generic filter verb supporting multiple filter kinds.

```askl
@filter("type", "func")              # Filter to functions (same as bare @func)
@filter("compound_name", "main")     # Filter by compound name match
@filter("exact_name", "/src/main.go") # Filter by exact name
```

**Filter kinds:**
- `"type"`: Filter by symbol type (`"func"`, `"mod"`, `"file"`, `"dir"`)
- `"compound_name"`: Filter by compound name pattern (token matching)
- `"exact_name"`: Filter by exact symbol name

### @ignore (Symbol Filtering)

Excludes symbols matching a pattern from current and nested statements.

```askl
@ignore("test") "main" {}        # Ignore test functions
@ignore("builtin") @ignore("fmt") "process" {}  # Multiple ignores
```

### @preamble (Global Configuration)

Applies verbs to the global scope, affecting all subsequent statements.

```askl
@preamble
@ignore("builtin")
@ignore("test")

"main"  # This and all following queries ignore builtin and test
```

### @project (Project Filter)

Filters results to a specific project (useful in multi-project setups).

```askl
@project("myproject") "main" {}    # Only symbols from myproject
```

### @forced (Override Relationships)

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

### @label (Define a Label)

Labels a statement for later reuse with `@use`.

```askl
@label("handlers") "handler" {}
```

### @use (Reuse a Label)

References a previously labeled statement.

```askl
@label("handlers") "handler" {};
"main" { @use("handlers") }    # Reuse the handlers selection
```

**Forced usage:**

```askl
@label("handlers") "handler" {};
"main" { !@use("handlers", forced=true) }  # Force the relationship
```

## Query Examples

### Basic Call Graph

```askl
"main" {{}}  # main and two levels of callees
```

### Who Calls This Function?

```askl
{ "targetFunction" }  # All callers of targetFunction
```

### Module Contents

```askl
@mod("mypackage") { @func }  # All functions in module
```

### Directory Contents

```askl
@dir("/src") { @file }       # All files in /src
@dir("/src") {}              # Directories and files in /src
```

### Mixed Relationships

```askl
@mod("pkg") { "handler" {{}} }
# Module pkg, functions named "handler" within it, and their call graph
```

### Filter by Project and Type

```askl
@project("backend") @mod("api") { @func("Create") }
# Create functions in api module of backend project
```

### Exclude Test Code

```askl
@preamble @ignore("test") @ignore("mock");
"main" {{}}  # Call graph excluding test/mock code
```

## Scope Types

### Callee Scope

Shows what the parent symbol calls:

```askl
"parent" { "child" }  # parent calls child
```

### Caller Scope

Shows what calls the nested symbol:

```askl
{ "target" }  # Functions that call target
```

### Nested Scopes

Multi-level relationships:

```askl
"a" { "b" { "c" } }  # a calls b, b calls c
```

## Query Rules and Validation

### Required Elements

Every global statement must contain at least one selection verb at some nesting level.

**Valid examples:**
```askl
"foo" {};           # Direct selection
{"foo"};            # Selection in scope
{{{"foo"}}}         # Deeply nested selection
```

**Invalid example:**
```askl
{{{{}}}}            # No selection anywhere
```

### Scopes Filter Results

Scopes only show results if the actual relationship exists. If `parent` doesn't call `child`, no results are displayed:

```askl
"parent" { "child" }  # Empty if parent doesn't call child
```
