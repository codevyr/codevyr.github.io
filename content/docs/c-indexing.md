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
