# FreeLinX/toolchain

The Clang/LLVM + musl cross-toolchain every other FreeLinX repo builds
against. This repository is the root of the dependency chain — the kernel,
ports, and src rootfs staging all assume a working sysroot and compiler
wrappers produced here.

This repository is **not** a package manager, not a GCC/glibc toolchain, and
not the OS itself. It is the compiler, linker, C library, and C++ library
FreeLinX targets: `x86_64-linux-musl`, built with **zero GNU components** in
the target output (GNU tools may be used on the *host* only, during early
bootstrap — see `product notes` below).

---

## What this is, and why it exists

FreeLinX runs the Linux kernel but rejects glibc and the GNU toolchain
entirely. That means nothing can be compiled against the host's default
`cc`/`libc` — every binary in the OS (kernel, ports, userland) needs to be
built against a musl sysroot using Clang and LLD instead of GCC and GNU
binutils. This repository produces that sysroot and the wrapper scripts
that make using it painless.

```
musl source (git.musl-libc.org)
        │
        ▼
musl sysroot            (headers + libc.a/libc.so/crt*.o, built with
                          clang --target=x86_64-linux-musl + llvm-ar/ranlib)
        │
        ▼
LLVM libcxx/libcxxabi/libunwind
        │              (cross-built via CMake+Ninja against the musl
        │               sysroot, using the wrapper scripts below)
        ▼
freelinix-cc / freelinix-c++ / freelinix-ar / freelinix-ranlib / ...
        │              (thin wrapper scripts baking in --target,
        │               --sysroot, -fuse-ld=lld, -rtlib=compiler-rt,
        │               -unwindlib=none)
        ▼
Used by FreeLinX/kernel, FreeLinX/ports, and anything else needing
a compiler
```

---

## Repository layout

```
toolchain/
├── README.md              this file
├── BUILD.md                the musl sysroot build recipe, step by step
├── env.sh                  sources PATH/CC/CXX/AR/RANLIB/... into your shell
├── bin/
│   ├── clang, clang++, ld.lld, llvm-ar, llvm-ranlib, ...   (real LLVM tools)
│   ├── freelinix-cc         C compiler wrapper
│   ├── freelinix-c++        C++ compiler wrapper (libc++/libc++abi)
│   ├── freelinix-ar          -> llvm-ar
│   ├── freelinix-ranlib      -> llvm-ranlib
│   ├── freelinix-nm          -> llvm-nm
│   └── freelinix-strip       -> llvm-strip
└── x86_64-linux-musl/      the sysroot: headers, libc.a/.so, crt*.o,
                            libc++.a/.so, libc++abi.a/.so
```

---

## Building it from scratch

Full step-by-step recipe: see `BUILD.md`. Summary of what it does:

### 1. musl sysroot

musl's own build system assumes a GNU cross-prefixed toolchain
(`x86_64-linux-musl-gcc`, `x86_64-linux-musl-ar`, ...), which doesn't exist
when using Clang/LLVM. The fix is overriding `AR`/`RANLIB` explicitly and
folding the target triple into `CC` itself, rather than passing
`--target=` to musl's `configure`:

```bash
cd musl-src
make clean

CC="clang --target=x86_64-linux-musl" \
AR=llvm-ar \
RANLIB=llvm-ranlib \
CFLAGS="-fuse-ld=lld" \
./configure --prefix=$HOME/freelinix/toolchain/x86_64-linux-musl

make -j$(nproc)
make install
```

The install step tries to symlink musl's dynamic loader into the *host's*
`/lib/ld-musl-x86_64.so.1`. This fails without root and is expected/
harmless — FreeLinX never wants that symlink on the build host, only in
the sysroot itself.

### 2. Verifying the sysroot

A static binary compiled against it must run natively (the WSL2/Linux
kernel executes it regardless of libc, since libc is a userspace concern):

```bash
clang --target=x86_64-linux-musl \
      --sysroot=$HOME/freelinix/toolchain/x86_64-linux-musl \
      -fuse-ld=lld -rtlib=compiler-rt -unwindlib=none -static \
      -o /tmp/hello /tmp/hello.c
/tmp/hello
```

Bare `clang --target=... --sysroot=... -static` alone is not enough: Clang's
Linux driver defaults to GCC's runtime (`libgcc`, `crtbeginT.o`/`crtend.o`),
which the musl sysroot doesn't provide (musl has no GCC-style crt objects
or libgcc). `-rtlib=compiler-rt -unwindlib=none` tells Clang to use LLVM's
own runtime instead — this is required on every compile/link against this
sysroot, which is why it's baked into the wrapper scripts below rather than
left for each caller to remember.

### 3. libc++ / libc++abi / libunwind

The musl sysroot only provides the C library — no C++ standard library.
`freelinix-c++` doesn't work until libc++ is cross-built against musl:

```bash
cmake -G Ninja llvm-project/runtimes \
  -DCMAKE_C_COMPILER=$HOME/freelinix/toolchain/bin/freelinix-cc \
  -DCMAKE_CXX_COMPILER=$HOME/freelinix/toolchain/bin/freelinix-c++ \
  -DCMAKE_C_COMPILER_TARGET=x86_64-linux-musl \
  -DCMAKE_CXX_COMPILER_TARGET=x86_64-linux-musl \
  -DCMAKE_SYSROOT=$HOME/freelinix/toolchain/x86_64-linux-musl \
  -DCMAKE_TRY_COMPILE_TARGET_TYPE=STATIC_LIBRARY \
  -DLLVM_ENABLE_RUNTIMES="libcxx;libcxxabi;libunwind" \
  -DLIBCXXABI_USE_LLVM_UNWINDER=ON \
  -DLIBCXX_HAS_MUSL_LIBC=ON \
  -DLIBCXX_USE_COMPILER_RT=ON \
  -DLIBCXXABI_USE_COMPILER_RT=ON \
  -DLIBUNWIND_USE_COMPILER_RT=ON \
  -DCMAKE_INSTALL_PREFIX=$HOME/freelinix/toolchain/x86_64-linux-musl \
  -DCMAKE_BUILD_TYPE=Release

ninja cxx cxxabi unwind
ninja install-cxx install-cxxabi install-unwind
```

Two non-obvious things this needed:

- **`-DCMAKE_TRY_COMPILE_TARGET_TYPE=STATIC_LIBRARY`** — CMake's default
  compiler sanity check tries to *link* a full test program against
  `-lc++`, which doesn't exist yet (it's what's being built). This flag
  makes the sanity check stop at compiling an object file instead, which
  is the standard workaround for bootstrapping libc++ cross-builds.
- **Pointing `CMAKE_C_COMPILER`/`CMAKE_CXX_COMPILER` at the `freelinix-cc`/
  `freelinix-c++` wrapper scripts, not bare `clang`/`clang++`** — bare
  clang with only `--target`/`--sysroot` still falls back to GCC's runtime
  (see the crt/libgcc issue above); the wrappers already carry
  `-rtlib=compiler-rt -unwindlib=none`.

---

## Wrapper scripts (`bin/freelinix-*`)

Every FreeLinX repo compiles through these, not bare `clang`:

```bash
#!/bin/sh
# freelinix-cc
exec clang --target=x86_64-linux-musl \
    --sysroot="$HOME/freelinix/toolchain/x86_64-linux-musl" \
    -fuse-ld=lld \
    -rtlib=compiler-rt \
    -unwindlib=none \
    "$@"
```

`freelinix-c++` is the same with `clang++` and `-stdlib=libc++` added.
`freelinix-ar`/`freelinix-ranlib`/`freelinix-nm`/`freelinix-strip` are thin
`exec llvm-*` wrappers — musl/Clang builds need `llvm-ar`/`llvm-ranlib`
specifically, since there's no GNU-cross-prefixed `ar`/`ranlib` in an
LLVM-only toolchain.

`env.sh` sets `PATH`, `CC`, `CXX`, `AR`, `RANLIB`, `NM`, `STRIP` to these
wrappers in one shot:

```bash
. ~/freelinix/toolchain/env.sh
```

---

## Verifying the toolchain reports musl, not GNU

A bare `clang --version` with no flags reports Clang's compiled-in default
target (usually the host's own triple — `x86_64-unknown-linux-gnu` on a
typical glibc-based Linux host). **This is cosmetic and does not reflect
what gets built** — it's only the default used when no `--target` is
passed, and every real invocation in FreeLinX always passes one explicitly.

To see the real target FreeLinX actually builds for:

```bash
~/freelinix/toolchain/bin/clang \
  --target=x86_64-linux-musl \
  --sysroot=$HOME/freelinix/toolchain/x86_64-linux-musl \
  --version
# Target: x86_64-unknown-linux-musl
```

Similarly, `ld.lld --version` prints `"(compatible with GNU linkers)"` —
this describes command-line *flag* compatibility only (so existing
Makefiles/build systems don't need rewriting), not GNU code or lineage.
LLD is LLVM's own linker, written from scratch.

The real proof is in the compiled output, not tool banners:

```bash
file <binary>   # look for "statically linked", no GNU interpreter path
ldd <binary>    # expect: "not a dynamic executable" for static builds
```

---

## Status

- musl sysroot: **built and verified** — static hello-world compiles,
  links, and runs correctly.
- `freelinix-cc`/`freelinix-ar`/`freelinix-ranlib`/`freelinix-nm`/
  `freelinix-strip`: **built and verified.**
- `freelinix-c++` / libc++ / libc++abi / libunwind: **built and verified**
  — cross-compiled against the musl sysroot via CMake + Ninja.
- Used successfully to build: `FreeLinX/kernel` (Linux 6.6.21, LLVM=1,
  zero GNU binutils), and every port in `FreeLinX/ports` including the
  multi-binary `sysutils/runit` suite — all confirmed statically linked
  with no glibc/GNU dependency (`ldd` reports "not a dynamic executable").

---

## Related repositories

- `kernel` — Linux 6.6.21, built entirely with this toolchain (`LLVM=1
  LLVM_IAS=1 CC=clang`).
- `ports` — NetBSD-derived userland + runit, compiled and linked against
  this sysroot via `FREELINX_CC`/`FREELINX_LD`/`FREELINX_SYSROOT`.
- `src` — root filesystem assembly; consumes `ports`' staged output.
- `iso` — final bootable image, consumes `src`'s output.
