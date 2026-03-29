---
title: "Composition Model"
description: "Understanding containment relationships between symbols"
weight: 250
---

Askl's composition model allows you to query **containment relationships** between symbols. This is distinct from reference relationships (function calls) and enables powerful architectural queries.

## Symbol Hierarchy

Symbols are organized in a hierarchy based on their type:

```
directory (level 4)
    └── module (level 3)
        └── file (level 2)
            ├── function (level 1)
            ├── type (level 1)
            ├── data (level 1)
            └── macro (level 1)
```

A higher-level symbol **contains** a lower-level symbol if:
1. They share the same source file (object)
2. The higher-level symbol's byte range encompasses the lower-level symbol's range
3. The type levels are compatible (higher > lower)

## Two Relationship Types

### References (`refs`)

The default relationship type. Shows what symbols **call** or **reference** each other.

```askl
"foo" { "bar" }        /* foo calls bar */
"foo" refs { "bar" }   /* Same, explicit syntax */
```

### Containment (`has`)

Shows what symbols **contain** other symbols.

```askl
mod("pkg") has { "handler" }   /* handler is IN pkg */
file("/main.go") { func }      /* Functions IN main.go (implicit has from file) */
```

## Container Types: Implicit Relationships

Container type selectors (`dir`, `file`, `mod`) automatically set both containment and reference relationships for their children. This means you don't need explicit `has` when using them:

```askl
/* These are equivalent: */
mod("pkg") { func }
mod("pkg") has { func }

/* dir also sets implicit refs+has: */
dir("/src") { file }           /* Files in /src (no has needed) */
dir("/") {}                    /* Shows directories and files */

/* func overrides back to refs-only: */
dir("/") { func("main") { "bar" } }
/* "bar" found via call graph (REFS), not containment */
```

Each container type also sets default child types:

| Container | Default Child Types |
|-----------|-------------------|
| `dir` | directories, files |
| `file` | functions, modules |
| `mod` | modules, functions |
| `type` | types |
| `data` | data |
| `macro` | macros, functions |

## Practical Examples

### Find Functions in a Module

```askl
mod("api/handlers") { func }
```

Returns the module and all functions physically located within it.

> **Performance Note:** Inside scopes, bare type selectors like `func` use efficient filter mode—they derive from the parent instead of querying all symbols. To explicitly select all functions regardless of context, use `func(filter="false")`.

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
2. Finds "ServeHTTP" functions **contained** in that module (implicit refs+has from `mod`)
3. Shows the call graph of those functions (two levels deep)

### Compare Container vs Explicit Relationship

```askl
/* Containment: what functions are IN the module? */
mod("pkg") { func }

/* References only: what functions does the module CALL? */
mod("pkg") derive(type="refs") { func }
```

These return very different results:
- The first uses the implicit refs+has from `mod` — functions found via containment
- The second explicitly overrides to refs-only — functions found via call references

## Relationship Inheritance

`has`, `refs`, and `derive` all inherit by default — their relationship type propagates to all descendants until explicitly overridden.

```askl
has {              /* HAS for all descendants */
    "foo" {        /* Still uses HAS (inherited) */
        "bar"      /* Still uses HAS (inherited) */
    }
}

has {              /* HAS for descendants */
    "foo" refs {   /* Override to REFS for this scope and below */
        "bar"      /* Uses REFS */
    }
}
```

Container type selectors participate in this inheritance:
- `func`, `type`, `data`, `macro` explicitly set **REFS**, overriding any inherited refs+has
- `mod`, `file`, `dir` set **refs+has** with inheritance

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

No explicit `has` needed — each container type sets it implicitly.

## Multi-Instance Symbols

Some symbols have multiple instances:

- A **module** has one instance per file it spans
- A **directory** has one instance per contained file

When clicking on such nodes in the UI, a popup shows all instances, allowing you to navigate to specific files.

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

This finds functions in "myapp" that reference the "external" module. Note: `refs` explicitly overrides the inherited refs+has from `mod`, so only call references are shown.

### Refactoring Support

Find functions that should be moved:

```askl
mod("utils") { func("http") }
```

If you have HTTP-related functions in a "utils" module, they might belong elsewhere.
