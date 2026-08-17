---
title: "Composition Model"
description: "Understanding containment relationships between symbols"
weight: 250
---

Askl's composition model allows you to query **containment relationships** between symbols. This is distinct from reference relationships (function calls) and enables powerful architectural queries.

## Symbol Hierarchy

Symbols are organized in a hierarchy based on their type:

```
directory (level 5)
    └── module (level 4)
        └── file (level 3)
            ├── function (level 2)
            ├── type (level 2)
            ├── data (level 2)
            ├── macro (level 2)
            └── field (level 1)
```

A higher-level symbol **contains** a lower-level symbol if:
1. They share the same source file (object)
2. The higher-level symbol's byte range encompasses the lower-level symbol's range
3. The type levels are compatible (higher > lower)

Fields sit at the lowest level, so they can be contained by types (e.g., `type("file_operations") has { field }`), files, modules, and directories.

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
| `field` / `method` | functions |

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

### Worked Example: "Defined In" vs "Referenced By"

A real query against the Linux kernel that shows why the distinction matters.
Ask for functions related to the `amdgpu` module that call `drm_dev_enter`:

```askl
mod("amdgpu") { func { "drm_dev_enter" } }
```

With the default relationship (refs **or** has), the middle statement matches
any function the module *references* — which pulls in DRM-core helpers that
amdgpu merely calls or wires up, not functions amdgpu defines:

- `drm_dev_is_unplugged` (a header inline in `include/drm/drm_drv.h` whose
  body is a `drm_dev_enter` call) matches because amdgpu code calls it;
- `drm_show_fdinfo` (in `drm_file.c`) matches because `amdgpu_drv.c`
  references it in a fops initializer (`.show_fdinfo = drm_show_fdinfo`) —
  a function-pointer reference is still a reference.

To ask for functions **defined in** the module instead, make the outer
relationship containment — and pin the inner one back to references, because
relationship context flows inward and would otherwise ask for functions
*containing* `drm_dev_enter`:

```askl
mod("amdgpu") has { func refs { "drm_dev_enter" } }
```

This keeps only `amdgpu_*` functions whose bodies call `drm_dev_enter` and
drops the two DRM-core helpers.

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
- `func`, `type`, `data`, `macro`, `field`/`method` explicitly set **REFS**, overriding any inherited refs+has
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
