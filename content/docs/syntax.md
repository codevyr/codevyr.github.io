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

## Type Selectors

Type selectors filter results to specific symbol types. They can optionally include a name pattern.

### @function

Selects function symbols.

```askl
@function                    # All functions
@function("handler")         # Functions matching "handler"
@function("http.Handler")    # Functions matching both "http" and "Handler"
```

### @module

Selects module/package symbols.

```askl
@module                      # All modules
@module("util")              # Modules matching "util"
@module("k8s.io/api")        # Modules matching the pattern
```

### @file

Selects file symbols.

```askl
@file                        # All files
@file("main.go")             # Files matching "main.go"
```

### @directory

Selects directory symbols.

```askl
@directory                   # All directories
@directory("cmd")            # Directories matching "cmd"
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
