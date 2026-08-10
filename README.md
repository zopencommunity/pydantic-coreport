# python-pydantic-coreport

z/OS port of [pydantic-core](https://github.com/pydantic/pydantic-core) — the Rust-backed
validation and serialization engine powering [pydantic](https://docs.pydantic.dev/) v2.

## Status

[![Build Status](https://img.shields.io/badge/z%2FOS-passing-brightgreen)](https://github.com/zopencommunity/python-pydantic-coreport)

| Python | Wheel |
|--------|-------|
| 3.12 | `pydantic_core-2.41.5-cp312-none-any.whl` |
| 3.13 | `pydantic_core-2.41.5-cp313-none-any.whl` |
| 3.14 | `pydantic_core-2.41.5-cp314-none-any.whl` |

## How It Works

`pydantic-core` is a Rust extension. There is no native Rust toolchain for z/OS, so the
`.so` is cross-compiled from Linux-on-Power (LoP) using IBM's Rust cross-compiler targeting
`s390x-ibm-zos`, then packaged as a standard Python wheel and published as a release asset.

The `buildenv` downloads the prebuilt wheel for each declared Python version, verifies it,
and passes it through the standard `ZOPEN_BUILD_SYSTEM=Python` pipeline so it lands in the
zopen wheel index at `https://repo.zopen.community/pypi/wheels/simple/`.

## Installation

```sh
pip install pydantic-core \
  --index-url https://repo.zopen.community/pypi/wheels/simple/ \
  --only-binary pydantic-core
```

Or via zopen:
```sh
zopen install python-pydantic-core
```

## Cross-compilation

Wheels are built on LoP using:
```sh
# cross/build_zos_wheel.py in rust-scripts
python3 cross/build_zos_wheel.py pydantic-core
```

Source: https://github.ibm.com/compiler/rust-scripts (branch `itodorov/zos-cross-compile-setup`)

## Note on versions

`pydantic-core 2.41.5` is the newest release buildable with rustc 1.86 (`rust-version = "1.75"`).
Version 2.42.0+ requires rustc 1.88. When the z/OS cross-compiler toolchain is updated,
this port will be updated to match the latest pydantic release.
