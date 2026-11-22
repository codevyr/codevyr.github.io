---
title: "Function Pointer Dispatch"
description: "Tracing function pointer and interface method dispatch to implementations"
weight: 260
---

Many codebases use indirect function calls through struct fields (C) or interface methods (Go). Askl models these dispatch points as **field** symbols, enabling queries that trace from a caller through a dispatch point to the implementing function.

## The Problem

In C, a common pattern is the vtable struct:

```c
struct file_operations {
    int (*read)(struct file *, char *, size_t);
    int (*write)(struct file *, const char *, size_t);
};
```

A function calls through the struct field:

```c
ssize_t vfs_read(struct file *f, ...) {
    f->f_op->read(f, buf, count);  // indirect call
}
```

And somewhere else, a driver provides the implementation:

```c
static struct file_operations my_ops = {
    .read = my_driver_read,
    .write = my_driver_write,
};
```

Without dispatch modeling, the indexer sees `vfs_read` calling a field access but has no way to connect it to `my_driver_read`. The `field` symbol type bridges this gap.

## How It Works

The indexer creates three kinds of records for each dispatch point:

1. **Field symbol + instance** at the struct field declaration (`file_operations.read` in the header)
2. **Call-site ref** from each `fops->read(...)` expression to the `file_operations.read` field symbol
3. **Synthetic ref** from the field's declaration site to each implementing function (one per `.read = impl` assignment)

This creates a two-hop chain: caller &rarr; field &rarr; implementation.

The two hops are two different relationships. The caller **calls** the slot: a
reference whose token is the callee of a call. The slot **holds** the
implementation: no token of the field's declaration names `my_read`, the
assignment elsewhere put it there. The query language can ask for each
([Asking by relationship](#asking-by-relationship)); a scope with no
relationship word follows both, which is what the queries below rely on.

### Reference Chain

Given this source layout:

```c
// fs.h
struct file_operations {
    int (*read)(...);       // field instance here
};

// drivers/my.c
static int my_read(...) { ... }
static struct file_operations ops = {
    .read = my_read,        // assignment site
};

// fs/read_write.c
ssize_t vfs_read(...) {
    f->f_op->read(...);     // call site
}
```

The indexer produces:

| Ref | To symbol | From file | Attributed to |
|-----|-----------|-----------|---------------|
| Call-site | `file_operations.read` | `read_write.c` | `vfs_read` function |
| Assignment | `my_read` | `drivers/my.c` | enclosing function/data |
| Synthetic | `my_read` | `fs.h` | `file_operations.read` field |

The synthetic ref is placed at the field's declaration range in the header. Since the field's instance covers that same range, the ref is attributed to the field by offset containment.

## Querying Dispatch Chains

### Find implementations of a field

```askl
field("file_operations.read") { func }
```

Returns all functions assigned to the `read` field of `file_operations`. For Linux, this could return hundreds of driver implementations.

### Broad match across structs

```askl
field("read") { func }
```

Since `"read"` is a simple name (no separators), it matches any field whose last component is `read` across all structs (`file_operations.read`, `address_space_operations.read`, etc.).

### Full dispatch chain from a caller

```askl
func("vfs_read") { field("read") { func } }
```

Traces: `vfs_read` &rarr; calls through `file_operations.read` &rarr; implementing functions.

### All fields of a struct (containment)

```askl
type("file_operations") has { field }
```

Returns all function pointer fields declared in `file_operations`. Works because a struct's fields sit inside it in the containment tree.

### Who refers to a field? (reverse)

```askl
func { field("file_operations.read") }
```

Returns the functions that reference `file_operations.read`: the ones that call
through the `read` dispatch point, and the ones that assign it
(`ops->read = impl`, or an initializer inside a function). Telling the two
apart is [asking by relationship](#asking-by-relationship).

### Combine with other verbs

```askl
preamble ignore("test") ignore("mock")

func("vfs_read") {
    field("read") {
        func { func }
    }
}
```

From `vfs_read`, through the `read` dispatch, find implementations and their callees — excluding test and mock code.

## Asking by relationship {#asking-by-relationship}

The queries above use the default scope, which accepts every relationship. The
[relationship words](/docs/syntax/#relationship-modifiers) ask for one.

> **Today's indices.** The words in this section need an index that states
> what a reference does (`calls`, `writes`) and records what a slot `holds`.
> No published index does yet: on the kernel today each query below answers
> with a notice instead of symbols, quoted under it. The default-scope queries
> above are the ones to use until the indexers write them
> ([what today's indices state](/docs/syntax/#what-an-index-states)).

### Who calls through a slot, and who assigns it

```askl
func calls { field("platform_driver.probe") }    /* who calls through the slot */
func writes { field("platform_driver.probe") }   /* who assigns it */
func { field("platform_driver.probe") }          /* both */
```

On the kernel the third lists seven functions: `platform_probe`, which calls
through the slot, and six that assign it, among them
`__platform_driver_probe`. The first two are meant to split that list. Today
they say:

```
the `linux` index states no roles; `calls` cannot match there. `refs { … }` lists its references
```

A role that is not stated is unknown, never "no": a role word does not match a
reference that says nothing about what it does, and the answer says what it
passed over rather than coming back empty without a word.

### What a slot holds

```askl
field("file_operations.read") holds { func }   /* the functions stored in the slot */
method("Read") holds { func }                  /* Go: the methods that implement it */
```

`holds` is the slot's own relationship (`⇢` in the report), apart from
`refs`: a field's declaration may also *name* things, such as the types in its
signature, and `holds` leaves those out. Today:

```
the `linux` index has no `holds` rows: nothing in it records what a slot holds
```

Published indices record a stored function as an ordinary reference from the
field for now, which is why `field("file_operations.read") { func }` and
`refs { func }` list the implementations today. Once an index records `holds`,
`refs` stops listing them and the default scope keeps listing them.

### A call across an ops table

**`calls` is literal.** `vfs_read` calls the slot `file_operations.read`. It
does not call `my_read`, and `"vfs_read" calls { func }` will not list it. The
functions behind the slot are one more hop, and the hop is written:

```askl
"vfs_read" calls { field holds { func } }   /* calls a slot; the slot holds a function */
"vfs_read" calls { { func } }               /* a bare middle scope crosses the slot too */
"vfs_read" { field { func } }               /* default scopes: the same chain, today */
```

When a role word finds nothing because the target sits behind a slot, the
answer points at the hop. From the engine's test index:

```
nothing calls `"my_read"` directly; `slot` holds it (`⇢`): `calls { field holds { "my_read" } }` goes through the slot
```

The same from the caller's side: a listing that leaves a called slot out, such
as `"via" { func }`, says which slots it passed over and gives the query that
follows them
([notices](/docs/syntax/#relationship-notices)):

```
`func` passed over 1 slot that `"via"` calls through: `slot`; follow it: `"via" calls { field holds { func } }`
```

## Go Interface Methods

In Go, interface methods serve the same role as C function pointer fields. The indexer models them identically using `field` symbols.

### How It Works

For an interface declaration:

```go
type Reader interface {
    Read(p []byte) (n int, err error)
}
```

The indexer creates a field symbol `(io.Reader).Read` with an instance at the method signature in the interface definition.

When a concrete type is assigned to an interface:

```go
var r io.Reader = &MyFile{}
```

The indexer creates refs from the interface method's declaration to each matching concrete method (e.g., `(*MyFile).Read`).

### Querying Go interfaces

The `method` verb is an alias for `field`, provided for readability in Go/OOP contexts:

```askl
method("Read") { func }
```

Returns all concrete methods that implement a `Read` interface method.

```askl
type("Reader") has { method }
```

Returns all method signatures declared in the `Reader` interface.

```askl
func("processData") { method("Read") { func } }
```

Traces: `processData` calls through the `Read` interface method &rarr; concrete implementations.

## Naming Conventions

| Language | Field name format | Example |
|----------|------------------|---------|
| C | `struct_name.field_name` | `file_operations.read` |
| Go | `(pkg.Interface).Method` | `(io.Reader).Read` |

Both use dots as separators, so matching works uniformly:
- `field("read")` or `method("Read")` — simple name, matches the last component (leaf match)
- `field("file_operations.read")` or `method("io.Reader.Read")` — compound name, matches precisely (pattern match)

## Limitations

| Case | Status |
|------|--------|
| Direct assignment (`.read = impl`) | Supported |
| Binary assignment (`ops->read = impl`) | Supported |
| Compound literals (`(struct file_ops){ .read = impl }`) | Supported |
| Conditional (`ops->read = cond ? a : b`) | Both targets captured |
| Copied to local variable (`fn = ops->read; fn()`) | Not tracked |
| Whole-struct pass (`register_ops(&ops)` with later field call) | Not tracked (requires alias analysis) |
| Anonymous structs | Not tracked (no compound name) |
