# Patches for python-pydantic-coreport

## Overview

`pydantic-core` is a Rust extension. It cannot be compiled natively on z/OS because
there is no native Rust toolchain yet. No source patches are applied by `zopen-build`.

## Cross-compilation approach

Wheels are cross-compiled from Linux-on-Power using the IBM Rust cross-compiler
(`s390x-ibm-zos` target) and published as release assets on this repository.

Key patches applied during the cross-compilation process (not via zopen-build):

1. **pyo3 0.26 fork** (`github.ibm.com/itodorov/pyo3`, branch `itodorov/zos-support-0.26`):
   - `is_linking_libpython_for_target()`: returns true for z/OS (XPLINK requires explicit link)
   - `print_libpython_rpath_link_args()`: skips rpath, uses LIBPATH instead

2. **libc fork** (`github.ibm.com/compiler/rust-libc`, branch `zOS.0.2.169`):
   - z/OS type definitions and syscall bindings

3. **getrandom 0.3.4 fork**: custom backend reading `/dev/urandom`

4. **Linker workaround**: `libpython3.12.x` side-deck passed directly as
   `-C link-arg=` (IBM binder uses `.x` import libraries, not `.a` archives)

## Proc-macro workaround

Cargo 1.86 does not pass proc-macro crates as `--extern` to cross-target compilations.
Nine proc-macro `.so` files are pre-built natively and injected via `rustflags`
in `.cargo/config.toml`.

## Test results

- SchemaValidator smoke tests: 5 per interpreter
- Validates: str/int schemas, JSON round-trip, ValidationError, typed_dict
