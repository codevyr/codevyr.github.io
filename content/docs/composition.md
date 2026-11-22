---
title: "Composition Model"
description: "Understanding containment relationships between symbols"
weight: 250
---

Askl's composition model allows you to query **containment relationships** between symbols. This is distinct from reference relationships (function calls) and enables powerful architectural queries.

## The containment tree

Containment is a tree the indexer writes down for every file: the module and
directory the file belongs to, the file itself, its declarations, and whatever
those declare in turn — a struct's fields, a closure inside a function, the
declarations a macro generated. The common shape:

```
directory
    └── module
        └── file
            ├── function
            │     └── nested declarations
            ├── type
            │     └── field
            ├── data
            └── macro
```

The tree is not derived from symbol types or byte ranges — a function can
declare a struct, a struct can hold a function-pointer field, and everything a
macro generates shares one byte range — so the indexer records a depth-first
interval per declaration and the engine reads containment from that. The shape
above is only the usual case; [Containment](/docs/design/containment/) has the
model. `type("file_operations") has { field }` works because the struct's
fields sit inside it in that tree.

## Containment and References

A scope can ask for four relationships
([Relationship Words](/docs/syntax/#relationship-modifiers)). This page is
about the two that every index has: containment and references.

### References (`refs`)

Shows what symbols **call** or **reference** each other.

```askl
"foo" refs { "bar" }   /* foo references bar */
```

### Containment (`has`)

Shows what symbols **contain** other symbols.

```askl
mod("pkg") has { "handler" }   /* handler is IN pkg */
file("/main.go") has { func }  /* Functions IN main.go */
```

### The default: any relationship

A scope with no relationship word accepts any of them: `"foo" { "bar" }` keeps
`bar` if `foo` references it or contains it, and `file("/main.go") { func }`
finds the file's functions without `has`. Writing `has` or `refs` says which
one that scope asks for, and only that scope: the scopes nested below are back
to the default ([Relationship Inheritance](#relationship-inheritance)).

## Containers need no `has`

Because a bare scope accepts any relationship, container selectors (`dir`, `file`,
`mod`) list their contents directly. Nothing is set implicitly, and type
selectors never change the relationship:

```askl
/* Functions the module contains — or references; `has` keeps only the contents */
mod("pkg") { func }
mod("pkg") has { func }

dir("/src") { file }           /* Files in /src */
dir("/") {}                    /* Directories and files (default child types) */

/* "bar": referenced by main, or declared inside it */
dir("/") { func("main") { "bar" } }
```

A type word on the parent says nothing about the child: a scope with no
type word accepts every type. `dir("/src") { }` is everything under the
directory, and `dir("/src") { file }` is its files
([a type word and the scopes around it](/docs/syntax/#type-word-scope)).

## Several Children in One Scope

A scope can hold more than one statement, and the parent has to satisfy **all**
of them. Sibling statements — separated by `;` or a newline — **conjoin**:

```askl
mod("api") has { "Encode" ; "Decode" }
/* modules containing BOTH Encode and Decode */
```

Writing the two selectors in a **single** statement disjoins them instead:

```askl
mod("api") has { "Encode" "Decode" }
/* modules containing EITHER Encode or Decode */
```

The conjunction is genuine, so a sibling that matches nothing removes the
parent: `mod("api") has { "Encode" ; "Typo" }` returns nothing.

A `;` between **top-level** statements is a different separator — it ends the
statement instead of adding a condition, and the two statements' results are
unioned:

```askl
mod("api") has { "Encode" } ; mod("api") has { "Decode" }
/* modules containing Encode, plus modules containing Decode */
```

The same contrast in the reference direction, and the rest of the scope
syntax, is in the
[Syntax Reference](/docs/syntax/#sibling-statements-in-a-scope).

## Practical Examples

### Find Functions in a Module

```askl
mod("api/handlers") { func }
```

Returns the module and all functions physically located within it.

> **Performance Note:** Inside scopes, bare type selectors like `func` are pure constraints—the engine derives their rows from the neighbouring statements instead of querying all symbols, choosing the cheapest direction from measured cardinality. To explicitly enumerate all functions regardless of context, use `func select` (budget-bounded).

### Find What Module Contains a Function

```askl
mod { "processRequest" }
```

Returns the function "processRequest" and any modules that contain it.

### Mix Containment and References

```askl
mod("handlers") { "ServeHTTP" {{}} }
```

This query:
1. Selects the "handlers" module
2. Finds "ServeHTTP" functions **contained** in that module (a bare scope accepts containment)
3. Shows the call graph of those functions (two levels deep)

### Compare Container vs Explicit Relationship

```askl
/* Either kind: functions the module contains or references */
mod("pkg") { func }

/* Containment only: what functions are IN the module? */
mod("pkg") has { func }

/* References only: what functions does the module CALL? */
mod("pkg") refs { func }
```

The first is the union of the other two. The worked example below shows,
on the kernel, how far they can drift apart.

### Worked Example: "Defined In" vs "Referenced By"

A real query against the Linux kernel that shows why the distinction matters.
Ask for functions related to the `amdgpu` module that call `drm_dev_enter`:

```askl
mod("amdgpu") { func { "drm_dev_enter" } }
```

With the default relationship (any of them), the middle statement matches
any function the module *references* — which pulls in DRM-core helpers that
amdgpu merely calls or wires up, not functions amdgpu defines:

- `drm_dev_is_unplugged` (a header inline in `include/drm/drm_drv.h` whose
  body is a `drm_dev_enter` call) matches because amdgpu code calls it;
- `drm_show_fdinfo` (in `drm_file.c`) matches because `amdgpu_drv.c`
  references it in a fops initializer (`.show_fdinfo = drm_show_fdinfo`) —
  a function-pointer reference is still a reference.

To ask for functions **defined in** the module instead, make the outer
relationship containment. Each scope says what it asks for, so the inner one
is written as references:

```askl
mod("amdgpu") has { func refs { "drm_dev_enter" } }
```

This keeps only `amdgpu_*` functions whose bodies call `drm_dev_enter` and
drops the two DRM-core helpers. `has` is about the scope it is written on and
no other: leaving `refs` out gives the same answer here, because a function
cannot contain `drm_dev_enter`, and carrying `has` inward with
`inherit(relationship)` would ask for functions *containing* it and find none.

## Relationship Inheritance

**A relationship word governs the statement it is written in;
`inherit(relationship)` carries it** into every scope below, until a statement
writes its own.

```askl
has {              /* HAS for this scope */
    "foo" {        /* the default again: any relationship */
        "bar"
    }
}

has inherit(relationship) {   /* HAS for this scope and the ones below */
    "foo" {                   /* still HAS */
        "bar"
    }
}

has inherit(relationship) {   /* HAS for descendants */
    "foo" refs {              /* this scope asks for REFS instead */
        "bar"
    }
}
```

Type selectors (`func`, `mod`, `file`, …) take no part in it: they constrain
the symbol type and leave the relationship as they found it. A scope with no
word on it, and none carried down to it, accepts any relationship. The full
rule, and `inherit(type)` for type words, is in the
[Syntax Reference](/docs/syntax/#inherit).

### Nested Containment

Use nested containers for multi-level containment:

```askl
dir("/src") {
    mod {
        func("handler")
    }
}
```

This finds:
1. The `/src` directory
2. Modules contained in that directory
3. Functions named "handler" contained in those modules

No explicit `has` needed — a bare scope accepts containment. To ask for
containment and nothing else at every level, carry the word:

```askl
dir("/src") has inherit(relationship) {
    mod {
        func("handler")
    }
}
```

## Multi-Instance Symbols

Some symbols have multiple instances:

- A **module** has one instance per file it spans
- A **directory** has one instance per contained file

When clicking on such nodes in the UI, a popup lists the instances,
allowing you to navigate to specific files. The graph itself carries only
a count of the instances the result holds; the popup pages the instances
in from the server — in file order, with an exact total, and scoped to the
node's own containment context when a symbol is drawn once per container —
so a kernel module spanning hundreds of files opens as quickly as a
function, and fetching more never re-lays-out the graph. See the
[HTTP API](/docs/api#instances) for the underlying endpoint.

## Use Cases

### Architectural Queries

Find all handlers in a specific package:

```askl
mod("api/v1") { func("Handler") }
```

### Code Organization

See what's in a directory:

```askl
dir("/cmd") { mod { func("main") } }
```

### Dependency Scope

Find external dependencies used by a module:

```askl
mod("myapp") { func refs { mod("external") } }
```

This finds functions in "myapp" that reference the "external" module. Note: `refs` makes the inner scope ask for references only; a bare `{ }` would accept containment too.

### Refactoring Support

Find functions that should be moved:

```askl
mod("utils") { func("http") }
```

If you have HTTP-related functions in a "utils" module, they might belong elsewhere.
