# Patches for pydantic-core on z/OS (s390x-ibm-zos)

## Overview

`pydantic-core` is a high-performance Python extension module written in Rust using PyO3.
Cross-compilation targeting `s390x-ibm-zos` requires handling:
1. **Rust 1.86 `let-chains` and MSRV** (`pydantic-core 2.46.4` and `jiter 0.14.0`)
2. **`getrandom 0.3.3` on z/OS** (`/dev/urandom` backend + `libc::__errno`)
3. **ICU 2.3.0 feature gates** (`let_chains`, `inherent_str_constructors`, `unsigned_is_multiple_of`)
4. **Host Proc-Macro Resolution** (pre-building host proc-macros for cargo cross-compilation)
5. **z/OS Side-deck Linking** (`.x` import files for CPython runtime)

---

## 1. The `let-chains` Issue & Fix (Rust 1.86)

### Symptom
When compiling `pydantic-core 2.46.4` or `jiter 0.14.0` with `rustc 1.86`, the compiler rejects `if let ... && ...` statements with:
```text
error[E0658]: `let` expressions in this position are unstable
  --> src/...
   = note: see issue #53667 <https://github.com/rust-lang/rust/issues/53667> for more information
   = help: add `#![feature(let_chains)]` to the crate attributes to enable
```
Additionally, `cargo` may refuse to build if `rust-version = "1.88"` is declared in `Cargo.toml`.

### Solution
1. **Add `#![feature(let_chains)]`** to the root of `src/lib.rs` in `pydantic-core`:
   ```rust
   #![feature(let_chains)]
   ```
2. **Add `#![feature(let_chains)]`** to `jiter-0.14.0/src/lib.rs`.
3. **Add `#![feature(let_chains, inherent_str_constructors, unsigned_is_multiple_of)]`** to ICU 2.3.0 crates (`icu_locale_core`, `icu_collections`, `icu_normalizer`, `icu_properties`, `icu_provider`).
4. Pass `--ignore-rust-version` to `cargo` invocations to bypass MSRV checks.

---

## 2. Included Patches

| Patch File | Target | Description |
|---|---|---|
| `pydantic-core-2.46.4-zos.patch` | `pydantic-core 2.46.4` | Enables `#![feature(let_chains)]` in `src/lib.rs`. |
| `jiter-0.14.0-let-chains.patch` | `jiter 0.14.0` | Enables `#![feature(let_chains)]` in `src/lib.rs`. |
| `getrandom-0.3.3-zos.patch` | `getrandom 0.3.3` | Maps `target_os = "zos"` to `use_file` (`/dev/urandom`) and `libc::__errno`. |
| `cargo-config.toml` | `.cargo/config.toml` | Target configuration, proc-macro `--extern` mappings, and z/OS CPython side-deck link flags. |

---

## 3. Step-by-Step Build Instructions

### Step 1: Pre-build Host Proc-Macros
```bash
# 1. Run cargo check on host to compile proc-macros for the build host
export PYO3_NO_PYTHON=1
export PYO3_CONFIG_FILE=/tmp/pyo3-config-cp312.txt
cargo check --target-dir /tmp/pydc-host --ignore-rust-version

# 2. Symlink host proc-macro .so files for easy reference in .cargo/config.toml
for m in serde_derive strum_macros pyo3_macros yoke_derive zerofrom_derive zerovec_derive enum_dispatch displaydoc; do
    so=$(find /tmp/pydc-host/ -name "lib${m}-*.so" | head -1)
    ln -sf "$so" "/tmp/lib${m}_pydc.so"
done
```

### Step 2: Configure `.cargo/config.toml`
Place the following in `.cargo/config.toml`:
```toml
[target.s390x-ibm-zos]
linker = "s390x-ibm-zos-cc"
rustflags = [
  "-C", "target-feature=-vector",
  "-L", "/tmp/pydc-host/debug/deps",
  "--extern", "serde_derive=/tmp/libserde_derive_pydc.so",
  "--extern", "strum_macros=/tmp/libstrum_macros_pydc.so",
  "--extern", "pyo3_macros=/tmp/libpyo3_macros_pydc.so",
  "--extern", "yoke_derive=/tmp/libyoke_derive_pydc.so",
  "--extern", "zerofrom_derive=/tmp/libzerofrom_derive_pydc.so",
  "--extern", "zerovec_derive=/tmp/libzerovec_derive_pydc.so",
  "--extern", "enum_dispatch=/tmp/libenum_dispatch_pydc.so",
  "--extern", "displaydoc=/tmp/libdisplaydoc_pydc.so",
  "-C", "link-arg=/usr/lpp/IBM/cyp/v3r12/pyz/lib/libpython3.12.x",
]

[profile.zos]
inherits = "release"
opt-level = "z"
debug = 0
codegen-units = 1
lto = false
strip = true
```

### Step 3: Cross-Compile
```bash
export CC_s390x_ibm_zos=s390x-ibm-zos-cc
export AR_s390x_ibm_zos=s390x-ibm-zos-ar

cargo build --profile zos --target s390x-ibm-zos --ignore-rust-version
```

### Step 4: Package Wheel
The compiled shared library `lib_pydantic_core.so` is packaged into a wheel as:
`pydantic_core/_pydantic_core.cpython-312.so` along with the pure Python files from `python/pydantic_core/`.
