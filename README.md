# FreeLinX Toolchain

The LLVM and musl toolchain used to build every other component of
FreeLinX: the kernel, the ports, and the root filesystem.

## Overview

FreeLinX does not use GCC, glibc, or GNU binutils anywhere in its build
or runtime. This repository provides the alternative: a Clang/LLVM
compiler toolchain targeting musl libc, along with the wrapper scripts
that make it usable as a drop-in replacement for a traditional cc/c++
toolchain.

- Target triple: x86_64-linux-musl
- Compiler: Clang 21.1.8
- Linker: ld.lld
- C runtime: musl libc
- Compiler runtime: compiler-rt

## Layout

- bin/freelinix-cc, bin/freelinix-c++ — the canonical compiler entry
  points. Both wrap clang/clang++ with the correct --target, --sysroot,
  linker, and runtime library flags baked in, so other repositories can
  build against FreeLinX without repeating that configuration.
- x86_64-linux-musl/ — the musl sysroot: headers and libraries the
  compiler builds against.
- lib/clang/21/ — Clang's resource directory, including
  libclang_rt.builtins.a for the musl target.

Both bin/freelinix-cc and bin/freelinix-c++ must point at the real
wrapper scripts, not directly at clang/clang++. A bare symlink to clang
skips the --target and --sysroot flags entirely and silently falls back
to the host's default toolchain, which has caused real build failures
in this project before. Always confirm with:

    freelinix-cc --version

which should report Target: x86_64-unknown-linux-musl. If it reports
a gnu target instead, the wrapper is broken.

## Building the toolchain

The toolchain is built in two stages, bootstrapped from the host's
system compiler:

1. Stage 1: build LLVM (clang, lld) and the musl-targeted compiler-rt
   and libc++ runtimes using the host compiler, with
   CMAKE_TRY_COMPILE_TARGET_TYPE=STATIC_LIBRARY to avoid a circular
   dependency on libc++ during the CMake compiler check, and
   COMPILER_RT_BUILD_SANITIZERS=OFF, since the sanitizer runtimes
   assume glibc-only headers that musl does not provide.
2. Stage 2: once libc++ exists, rebuild self-hosted using the
   FreeLinX compiler wrappers themselves.

See docs/ (or the build scripts, once tracked here) for the exact CMake
configuration.

## Status

The toolchain builds successfully and has been used to build:

- The Linux 6.6.21 kernel (LLVM=1, zero GNU binutils)
- A statically linked BusyBox userland
- Individually built static userland tools (coreutils-equivalents, git,
  ninja, openssh) staged in FreeLinX/src

## Related repositories

- kernel — Linux kernel configuration, built with this toolchain
- ports — userland packages built with this toolchain
- src — root filesystem assembly, consumes toolchain and ports output
- iso — bootable image packaging
