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

Since `dot_is_separator` is true, `"read"` matches any field named `read` across all structs (`file_operations.read`, `address_space_operations.read`, etc.).

### Full dispatch chain from a caller

```askl
func("vfs_read") { field("read") { func } }
```

Traces: `vfs_read` &rarr; calls through `file_operations.read` &rarr; implementing functions.

### All fields of a struct (containment)

```askl
type("file_operations") has { field }
```

Returns all function pointer fields declared in `file_operations`. Works because `field` (level 1) is below `type` (level 2) in the hierarchy.

### Who calls through a field? (reverse)

```askl
{ field("file_operations.read") }
```

Returns functions that reference `file_operations.read` — i.e., functions that call through the `read` dispatch point.

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

Both use `dot_is_separator=true`, so token matching works uniformly:
- `field("read")` or `method("Read")` matches the last component
- `field("file_operations.read")` or `method("io.Reader.Read")` matches precisely

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
