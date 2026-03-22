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
@module("mypackage") @has { @function }  # Functions contained in module
@file("main.go") @has { "handler" }      # handler function in main.go
```

### 4. Pattern Matching

Askl uses exact, case-sensitive token matching on fully qualified names:

- `"cli.Run"` matches symbols containing both "cli" and "Run" tokens
- `"cli"` will NOT match "click" (exact matching)
- Matching works on the complete package path + symbol name

## Verb Types

Verbs in Askl fall into three categories:

### Selectors

**Selectors add symbols to the result.** They query the database and produce nodes in the graph.

| Verb | Description |
|------|-------------|
| `"name"` / `@select(name="...")` | Select symbols matching a name pattern |
| `@function("name")` | Select functions matching a name |
| `@module("name")` | Select modules matching a name |
| `@file("name")` | Select files matching a name |
| `@directory("name")` | Select directories matching a name |
| `@use("label")` | Select symbols from a labeled statement |

Multiple selectors in a statement combine—all must match for a symbol to be included.

### Filters

**Filters constrain the selection without adding symbols.** They remove symbols that don't match criteria.

| Verb | Description |
|------|-------------|
| `@ignore("pattern")` | Exclude symbols matching the pattern |
| `@project("name")` | Only include symbols from a specific project |
| `@function` (no name) | Only include function symbols |
| `@module` (no name) | Only include module symbols |
| `@file` (no name) | Only include file symbols |
| `@directory` (no name) | Only include directory symbols |

> **Note:** Type selectors like `@function` behave differently based on arguments:
> - With a name (`@function("foo")`) → **Selector** (queries matching symbols)
> - Without a name (`@function`) → **Filter** (constrains to type, derives from parent)
> - Explicit: `@function(filter="false")` forces selector mode

### Modifiers

**Modifiers change context or behavior** for the current statement or scope.

| Verb | Description |
|------|-------------|
| `@has` | Use containment relationships instead of references |
| `@refs` | Use reference relationships (default) |
| `@preamble` | Apply subsequent verbs to the global scope |
| `@label("name")` | Label this statement for reuse |
| `!` (forced) | Force display of relationships |
| `?` (weak) | Make statement non-constraining |

## Type Selectors

Type selectors target specific symbol types. As explained above, they act as **selectors** (with a name) or **filters** (without a name).

### Selector vs Filter Behavior

| Syntax | Role | Behavior |
|--------|------|----------|
| `@function("name")` | Selector | Queries functions matching "name" |
| `@function` | Filter | Constrains to functions; derives from parent |
| `@function(filter="false")` | Selector | Queries ALL functions |
| `@function(filter="true")` | Filter | Explicit filter (same as bare `@function`) |

**Why this default?** Querying all symbols of a type is expensive. Inside `@has { }` scopes, filter mode is much more efficient—it derives from the parent's contained symbols instead of querying the entire database.

### @function

Selects function symbols.

```askl
@function("handler")         # Functions matching "handler"
@function("http.Handler")    # Functions matching both "http" and "Handler"
@function(filter="false")    # All functions (explicit selector mode)
@file("main.go") @has { @function }  # Functions in main.go (filter mode, efficient)
```

### @module

Selects module/package symbols.

```askl
@module("util")              # Modules matching "util"
@module("k8s.io/api")        # Modules matching the pattern
@module(filter="false")      # All modules
```

### @file

Selects file symbols.

```askl
@file("main.go")             # Files matching "main.go"
@file(filter="false")        # All files
@directory("/src") @has { @file }  # Files in /src directory
```

### @directory

Selects directory symbols.

```askl
@directory("cmd")            # Directories matching "cmd"
@directory(filter="false")   # All directories
```

### Default Type Inheritance

When you use a type selector, child scopes inherit sensible defaults:

```askl
@module("mypackage") {}      # Children include modules AND functions
@module("mypackage") { @function }  # Children explicitly filtered to functions only
```

This prevents seeing every symbol type in results and shows only relevant relationships.

## Relationship Modifiers

### @refs (References)

Explicitly use reference-based relationships. This is the default, so it's optional.

```askl
"foo" @refs { "bar" }  # foo calls bar
"foo" { "bar" }        # Same - @refs is default
```

Use `@refs` to override an inherited `@has`:

```askl
@module("pkg") @has { "foo" @refs { "bar" } }
# Module contains foo, foo CALLS bar (not contains)
```

### @has (Containment)

Use containment-based relationships.

```askl
@module("pkg") @has { @function }    # Functions IN the module
@directory("/src") @has { @file }    # Files IN the directory
```

Containment is determined by source code byte ranges: if symbol A's range contains symbol B's range, and A's type level is higher than B's, then A contains B.

## Core Verbs

### @select (Function Selection)

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
@module("mypackage") @has { @function }  # All functions in module
```

### Mixed Relationships

```askl
@module("pkg") @has { "handler" @refs {{}} }
# Module pkg, functions named "handler" within it, and their call graph
```

### Filter by Project and Type

```askl
@project("backend") @module("api") @has { @function("Create") }
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
