---
title: "C/C++ Indexing"
description: "Index C and C++ projects using the clang-based indexer and a compilation database"
weight: 130
---

The C/C++ indexer (`askl-clang-indexer`) uses `compile_commands.json` — also known as a [compilation database](https://clang.llvm.org/docs/JSONCompilationDatabase.html) — to find every source file and its exact compiler flags. Only files that appear in the compilation database are indexed, so the key to maximizing coverage is configuring the project with as many optional features enabled as possible before building.

## How it works

1. **Configure** the project with its build system (CMake, autotools, Meson, etc.), enabling every optional feature you can.
2. **Build** the project (or at least generate the compilation database).
3. **Run the indexer** pointing it at the directory containing `compile_commands.json`.

The indexer parses each translation unit with libclang, extracts symbols, and writes the index to a protobuf file.

## Generating a compilation database

The method depends on the project's build system.

### CMake

CMake generates `compile_commands.json` natively:

```sh
cmake -B build -DCMAKE_EXPORT_COMPILE_COMMANDS=1 [other flags...]
cmake --build build -j$(nproc)
```

The file is written to the build directory (e.g., `build/compile_commands.json`). Point the indexer at that directory.

### Autotools (configure/make)

Autotools does not generate a compilation database on its own. Use [Bear](https://github.com/rizsotto/Bear) to intercept the build and capture compile commands:

```sh
./configure [flags...]
bear -- make -j$(nproc)
```

Bear writes `compile_commands.json` in the current directory.

### Meson

Meson generates `compile_commands.json` automatically during setup:

```sh
meson setup build [flags...]
ninja -C build
```

The file is in the build directory.

### Other build systems

For any build system, Bear can wrap the build command:

```sh
bear -- make -j$(nproc)
bear -- ninja -C build
```

## Maximizing coverage

**Only compiled code gets indexed.** If a feature is disabled at configure time, its source files never appear in the compilation database and the indexer cannot see them.

To index as much of the codebase as possible:

- **Enable all optional features**, even if you do not have the hardware. Install the development headers for every optional dependency. The code only needs to compile — it never runs.
- **Install stub libraries** for dependencies that have no Debian/RPM package. For example, [gdrcopy](https://github.com/NVIDIA/gdrcopy) has no distro package but you can build its library from source to satisfy the linker.
- **Use `allmodconfig` or equivalent** for projects that support it (e.g., the Linux kernel's `make allmodconfig`).
- **Prefer in-place or development builds.** Flags like CMake's `-DIN_PLACE=1` or autotools' `--enable-devel-headers` often compile additional code that a production build would skip.

### Example: CMake project (rdma-core)

rdma-core uses CMake. Install all optional dependencies, then configure with everything enabled:

```sh
cmake -B build -GNinja \
  -DCMAKE_EXPORT_COMPILE_COMMANDS=1 \
  -DCMAKE_BUILD_TYPE=RelWithDebInfo \
  -DENABLE_RESOLVE_NEIGH=1 \
  -DENABLE_VALGRIND=1 \
  -DENABLE_IBDIAGS_COMPAT=True \
  -DIN_PLACE=1

ninja -C build -j$(nproc)
```

This enables libnl neighbor resolution, valgrind annotations, obsolete diagnostic scripts, pyverbs, all hardware providers, and all daemon code — maximizing the number of source files in the compilation database.

### Example: Autotools project (UCX)

UCX uses autotools. Run `autogen.sh`, configure with all features, and wrap the build with Bear:

```sh
./autogen.sh

./contrib/configure-release \
  --enable-cma \
  --enable-mt \
  --enable-experimental-api \
  --enable-devel-headers \
  --enable-examples \
  --enable-test-apps \
  --with-valgrind \
  --with-verbs \
  --with-rdmacm \
  --with-rc --with-ud --with-dc \
  --with-mlx5 --with-ib-hw-tm \
  --with-dm --with-devx \
  --with-cuda=/usr \
  --with-rocm=/usr \
  --with-gdrcopy=/usr/local \
  --with-fuse3 \
  --with-bfd \
  --with-efa

bear -- make -j$(nproc)
```

This enables CUDA, ROCm, GDR copy, all InfiniBand transports, FUSE, debug symbols, and test applications.

## Running the indexer

Point the indexer at the directory containing `compile_commands.json`:

```sh
askl-clang-indexer /path/to/project/build \
  --project my-project \
  -j $(nproc) \
  --progress \
  --include-git-files \
  -o /path/to/output
```

| Flag | Purpose |
|------|---------|
| `--project` | Project name (used when uploading to askld) |
| `-j` | Number of parallel indexing threads |
| `--progress` | Show a progress indicator |
| `--include-git-files` | Add all git-tracked files to the project file list |
| `-o` | Output directory for the index protobuf file |

The first positional argument is the project directory. It must contain `compile_commands.json` (either directly or via the build directory).

## What the indexer records

Beyond symbols and references, the indexer writes down the **containment
tree** itself: every instance carries a depth-first interval and its nesting
level. Ranges alone cannot express this in C — everything
`PMU_FORMAT_ATTR(...)` generates shares one extent, and a `##`-pasted name has
no source location at all — so the walk that libclang already performs is what
states it. See [Containment](/docs/design/containment/) for the model and
[Traversal modes](/docs/syntax/#traversal-modes) for what it buys a query.

**Using a macro is a reference, not an instance.** A macro is declared once,
at its `#define`; every use is one site naming it, recorded as a reference in
the enclosing function. The indexer emits no *expansion* instances (the
`EXPANSION` instance type stays in the wire format so older indices still
parse); dropping them removed 37.8% of criu's instances and changed no
reference. What a macro *generates* — declarations that exist only because it
expanded — is recorded as ordinary instances, placed where the invocation was
written.

### Where things sit {#placement}

**An instance or a reference sits in the innermost instance whose extent
contains its text.** Nothing else decides it, and in particular not whether a
macro was involved:

```c
UCS_PROFILE_FUNC(ucs_status_ptr_t, ucp_tag_send_nbx, (ep, buffer), ucp_ep_h ep, const void *buffer)
{
    return ucp_tag_send_req(ep, buffer);     /* sits in ucp_tag_send_nbx */
}
```

The function's *name* passes through a macro, but its body is text the author
typed, and the extent runs to the closing brace. A declaration is *generated*
only when its whole extent is one macro invocation, as with `DEFINE_SHOW(event)`
producing `event_show` — and such a declaration contains what is typed in the
invocation's arguments, like any other instance contains its text.

When one invocation generates several declarations they share an extent, and
text cannot say which of them a reference is in. The parse can: the reference
sits in the one it was parsed inside. A reference has one position, though, so
when two generated functions refer to the same symbol through the same
argument token, one of them has the edge.

**A reference sits where its caller completed its name.** Usually that is the
token that spells it. Three cases need saying:

- *A name used as a value is a reference too* — `register_cb(handler)`,
  `return handler;`, `.open = my_open`, and `module_init(my_init)` where the
  macro stores its argument.
- *A pasted name sits on the part its caller typed.* With
  `#define MCA_PML_CALL(a) mca_pml.pml_ ## a`, the call
  `MCA_PML_CALL(send(buf))` refers to the field `pml_send`, and no token spells
  that. The caller chose `send`, so the reference sits on `send` and the edge
  is `PMPI_Send -> pml_send`, direct; `MCA_PML_CALL` has no edge to any
  `pml_*` field. A name pasted from two halves the macro wrote itself was
  chosen by nobody and stays the macro's.
- *A name completed by a macro passed to a list sits on that macro.* In
  `int (*table[])(int) = { EACH(ENTRY) };` with
  `#define EACH(_m) _m(alpha) _m(beta)` and `#define ENTRY(_n) handle_ ## _n,`
  the list types `alpha`, but that names nothing until `ENTRY` is applied — the
  same list with another macro would declare, or stringify. So the reference
  sits on `ENTRY`, and the edge is `table -> handle_alpha`.

A name written whole inside a `#define` body is that body's reference, once,
however many places use the macro: each user reaches it through the macro, in
two hops.

### Folding {#folding}

Every symbol carries one flag, **folded**: *not listed by default*. It is what
keeps `mod("amdgpu") { func }` to the functions people work with, and it has
nothing to do with where a symbol sits.

**A symbol is folded unless normal code wrote it down**, where normal code is
any text outside a `#define` body. There are three ways to write a symbol down:

| | example | unfolds |
|---|---|---|
| declare it with text of its own | `int plain(int a) { … }`, and the `ucp_tag_send_nbx` above | the declared symbol |
| type its whole name where a macro declares it | `DEFINE_MUTEX(lock)` | `lock` |
| type its name, or the part of a pasted name a caller chooses, in a reference | `kh_put(tbl, h, k)` | `kh_put_tbl` |

Everything else is folded: `__compiletime_assert_N`, an initializer's
constructor, `event_show` when nothing calls it by name, the fields of a
generated struct that no code reads. The rule is the same at every depth — a
field of a generated struct is unfolded by a `h->n_buckets` in normal code and
folded otherwise, whatever its siblings are.

What counts is where the name was **typed**, which is not always where the
reference **sits**:

- In `EACH(ENTRY)` above, the reference to `handle_alpha` sits in normal code,
  but `alpha` was typed in the list's `#define`. The edge exists and the
  function stays folded.
- `KHASH_INIT(tbl, …)` generates a family of functions that call each other,
  all on the one `tbl`. A caller's part of a pasted name *inside a generated
  declaration* is the generator's plumbing, not a use, and unfolds nothing.

The flag is decided once per symbol, after every translation unit has been
merged, because one file's reference unfolds what another file's macro
generated. Macros, files, directories and modules are never folded.

**A header is the union of its parses.** A header compiled by several
translation units may show each a different set of declarations. The index
keeps the union: identical declarations (same symbol, kind and extent) merge
into one, and a declaration only some compilations saw is added — see
[Containment](/docs/design/containment/) for why the alternative loses
silently.

Each instance also carries up to three optional spans inside its extent: the
**name** token, the **signature**, and the **content**. They are what let the
UI highlight an identifier or show a multi-line C signature correctly, and they
are stored rather than derived, because K&R declarations, attributes between
the signature and the body, and trailing return types all break the
subtraction. References carry their own position in the same depth-first
order, their nesting level, and optionally a *full span* for the whole
referencing construct — `printk(foo)`, where the name span covers only
`printk`.

**If the upload is rejected.** askld validates each object's tree before it
stores anything: non-empty intervals at least one stride wide, unique
`dfs_enter`, nesting without straddling, a consistent `nesting_level`, every
reference inside an instance. A malformed object fails the
upload with HTTP 400 naming it. An index produced before the containment tree
existed fails the first check, since every interval is `[0,0)` — re-run the
indexer. There is no backfill, by design: the old data cannot express the
tree, so inventing one would bake in a wrong answer.

Files pulled in by `--include-git-files` that no compile command covers are
still indexed as file objects, so `search()` reaches documentation, scripts and
headers that no translation unit includes.

## Pre-built Docker images

Pre-built containers are available for several projects. Each container installs all build dependencies, configures the project, builds it, and runs the indexer in one step:

```sh
# Linux kernel
docker run -v /path/to/linux:/linux -v /tmp/out:/out \
  ghcr.io/codevyr/askl-clang-indexer-linux:latest

# rdma-core
docker run -v /path/to/rdma-core:/rdma-core -v /tmp/out:/out \
  ghcr.io/codevyr/askl-clang-indexer-rdma-core:latest

# UCX
docker run -v /path/to/ucx:/ucx -v /tmp/out:/out \
  ghcr.io/codevyr/askl-clang-indexer-ucx:latest
```

Mount the project source tree and an output directory. The container handles configuration, building, and indexing automatically.

## Special modules

The indexer supports a `--modules` flag for build-system-specific symbol extraction. Currently the only supported module is `kbuild` for the Linux kernel, which creates additional `MODULE` symbols from the kernel's Kbuild files.
