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
            └── function (level 1)
```

A higher-level symbol **contains** a lower-level symbol if:
1. They share the same source file (object)
2. The higher-level symbol's byte range encompasses the lower-level symbol's range
3. The type levels are compatible (higher > lower)

## Two Relationship Types

### References (`@refs`)

The default relationship type. Shows what symbols **call** or **reference** each other.

```askl
"foo" { "bar" }        # foo calls bar
"foo" @refs { "bar" }  # Same, explicit syntax
```

### Containment (`@has`)

Shows what symbols **contain** other symbols.

```askl
@module("pkg") @has { "handler" }   # handler is IN pkg
@file("main.go") @has { @function } # Functions IN main.go
```

## Practical Examples

### Find Functions in a Module

```askl
@module("api/handlers") @has { @function }
```

Returns the module and all functions physically located within it.

### Find What Module Contains a Function

```askl
@module @has { "processRequest" }
```

Returns the function "processRequest" and any modules that contain it.

### Mix Containment and References

```askl
@module("handlers") @has { "ServeHTTP" @refs {{}} }
```

This query:
1. Selects the "handlers" module
2. Finds "ServeHTTP" functions **contained** in that module (`@has`)
3. Shows the call graph of those functions (`@refs`, two levels deep)

### Compare Container vs Reference

```askl
# Containment: what functions are IN the module?
@module("pkg") @has { @function }

# References: what functions does the module CALL?
@module("pkg") @refs { @function }
```

These return very different results:
- `@has` returns functions whose source code is within the module
- `@refs` returns functions that the module's code references

## Relationship Scope

The relationship modifier affects the **immediate child scope**:

```askl
@module("pkg") @has {     # First child: containment
    "foo" @refs {         # Second level: references (explicit)
        "bar"
    }
}
```

This query:
1. Module "pkg" **contains** "foo" (`@has`)
2. "foo" **calls** "bar" (`@refs`)
3. "bar" doesn't need to be in module "pkg"

### Nested Containment

Use nested `@has` for multi-level containment:

```askl
@directory("/src") @has {
    @module @has {
        @function("handler")
    }
}
```

This finds:
1. The `/src` directory
2. Modules contained in that directory
3. Functions named "handler" contained in those modules

## Overriding Inherited Relationships

Child scopes inherit the parent's relationship type by default. Use explicit modifiers to override:

```askl
@module("pkg") @has {     # Children inherit @has
    "foo" {               # Still uses @has (inherited)
        "bar"
    }
}

@module("pkg") @has {     # Children inherit @has
    "foo" @refs {         # Override to @refs
        "bar"
    }
}
```

## Multi-Instance Symbols

Some symbols have multiple instances:

- A **module** has one instance per file it spans
- A **directory** has one instance per contained file

When clicking on such nodes in the UI, a popup shows all instances, allowing you to navigate to specific files.

## Use Cases

### Architectural Queries

Find all handlers in a specific package:

```askl
@module("api/v1") @has { @function("Handler") }
```

### Code Organization

See what's in a directory:

```askl
@directory("/cmd") @has { @module @has { @function("main") } }
```

### Dependency Scope

Find external dependencies used by a module:

```askl
@module("myapp") @has { @function } @refs { @module("external") }
```

This finds functions in "myapp" that reference the "external" module.

### Refactoring Support

Find functions that should be moved:

```askl
@module("utils") @has { @function("http") }
```

If you have HTTP-related functions in a "utils" module, they might belong elsewhere.
