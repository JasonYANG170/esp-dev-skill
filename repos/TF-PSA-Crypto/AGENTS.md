# AGENTS.md — Supplementary Agent Guide

> Core principles, pitfalls, recipes index, and execution workflow are all in `SKILL.md`.
> This file covers **only** conventions and tooling guidance not present in `SKILL.md`. Do not duplicate content.

## Project Context

**Language**: C99 · **Library**: TF-PSA-Crypto 1.0.0 (PSA Cryptography API v1.2) · **Toolchain**: CMake 3.20.2+ with GCC 5.4+/Clang 3.8+/Arm Compiler 6.21+/MSVC 2019+ · **Python**: 3.8+ (for code/test generation) · **No fixed chip target** — portable C99 with 8-bit bytes, two's-complement ints, `int`/`size_t` ≥ 32 bits.

## Header & Include Conventions

### The single application header

Applications only need one header for all PSA crypto mechanisms:

```c
#include <psa/crypto.h>          /* all psa_* public APIs */
```

`crypto.h` pulls in `crypto_types.h`, `crypto_values.h`, `crypto_sizes.h`, `crypto_struct.h`, and `crypto_platform.h` automatically.

When you also need platform/build info (version macros, config version checks):

```c
#include "tf-psa-crypto/build_info.h"   /* TF_PSA_CRYPTO_VERSION_*, pulls crypto_config.h */
```

### Helper headers (only when needed)

- `mbedtls/platform.h` — for `mbedtls_printf`, `mbedtls_setbuf` (see `programs/psa/psa_hash.c`, `programs/psa/key_ladder_demo.c`)
- `mbedtls/platform_util.h` — `mbedtls_platform_zeroize` to wipe secret buffers (see `programs/psa/hmac_demo.c`)
- `psa/crypto_extra.h` — `mbedtls_psa_crypto_free()` and non-public extension types
- `psa/crypto_compat.h` — backward-compatibility shims

### Never `#include` internal core headers in applications

`core/psa_crypto.c`, `core/psa_crypto_core.h`, `core/psa_crypto_storage.*`, `core/psa_crypto_driver_wrappers.h` are internal. The public surface is everything under `include/psa/` and `include/tf-psa-crypto/`.

## Standard Application Layout

A typical PSA-crypto-using program looks like:

```
my_app/
├── CMakeLists.txt
├── src/
│   └── main.c              /* #include <psa/crypto.h>, psa_crypto_init(), ... */
├── psa_config/             /* optional custom crypto_config.h */
│   └── crypto_config.h
└── ...
```

When consuming TF-PSA-Crypto via CMake `find_package`:

```cmake
# Point CMake at the built TF-PSA-Crypto, then:
find_package(TF-PSA-Crypto REQUIRED)
add_executable(my_app src/main.c)
target_link_libraries(my_app PRIVATE TF-PSA-Crypto::tfpsacrypto)
```

## Canonical `main()` Pattern

The repo's example programs (`programs/psa/*.c`) all share this shape — reproduce it:

```c
#include <psa/crypto.h>
#include "tf-psa-crypto/build_info.h"
#include <stdio.h>
#include <stdlib.h>

/* Bail-out macro used across all programs/psa examples */
#define PSA_CHECK(expr)                                       \
    do {                                                      \
        status = (expr);                                      \
        if (status != PSA_SUCCESS) {                          \
            printf("Error %d at line %d: %s\n",               \
                   (int) status, __LINE__, #expr);            \
            goto exit;                                        \
        }                                                     \
    } while (0)

int main(void)
{
    psa_status_t status = PSA_SUCCESS;
    /* ...key/operation declarations... */

    PSA_CHECK(psa_crypto_init());

    /* ...do crypto work, each call wrapped in PSA_CHECK... */

exit:
    /* Always abort operations and destroy keys, even on success */
    /* psa_*_abort(...); psa_destroy_key(key); */
    mbedtls_psa_crypto_free();   /* from psa/crypto_extra.h */
    return status == PSA_SUCCESS ? EXIT_SUCCESS : EXIT_FAILURE;
}
```

Notes specific to this library:

- `psa_crypto_init()` returns `psa_status_t`, **not** `int`. Do not write `if (!psa_crypto_init())`.
- `mbedtls_psa_crypto_free()` (declared in `psa/crypto_extra.h`) frees all PSA resources and destroys volatile keys. Persistent keys in storage remain.
- The `PSA_CHECK` / `ASSERT_STATUS` goto-exit pattern is idiomatic — it guarantees cleanup runs on every error.

## Code Generation Conventions

### File & symbol naming

- Source files: `*.c`, headers: `*.h` (no C++).
- All public symbols are prefixed `psa_` (functions/types) or `PSA_` (macros/enums).
- Helper/internal symbols from this repo: `tf_psa_crypto_*`, `mbedtls_psa_*`.
- Applications: use your own prefix; never define symbols starting with `psa_` or `PSA_` (reserved).

### Variable naming idioms (from the examples)

- `status` — the `psa_status_t` return value (always checked).
- `attributes` / `attr` — a `psa_key_attributes_t`.
- `key` — a `psa_key_id_t` (an identifier, NOT a pointer).
- `op` / `operation` — a multi-part operation object (`psa_*_operation_t`).
- `*_len` / `output_length` — actual-output size out-params (e.g. `&hash_length`, `&signature_length`).
- `*_size` — buffer capacity in-params (e.g. `signature_size`, `hash_size`).

### Error handling

```c
psa_status_t status = some_psa_call(...);
if (status != PSA_SUCCESS) {
    /* to print the symbolic name: programs/psa/psa_constant_names status <value> */
    return status;
}
```

Never swallow a non-success status silently. On error inside a multi-part operation, call the matching `*_abort()` and (for keys) `psa_destroy_key()` before returning.

## Build Workflow (CMake)

From the TF-PSA-Crypto source root (or referencing it):

1. **One-time**: `git submodule update --init` (initializes the `framework/` submodule — required on the dev branch; release tarballs already include it).
2. **Generate build files**:
   ```
   cmake -B build -S .
   ```
   For a specific compiler or flags, set them on the first invocation only (CMake caches them):
   ```
   CC=arm-none-eabi-gcc cmake -B build -S . -DENABLE_TESTING=Off
   ```
3. **Build**:
   ```
   cmake --build build
   ```
   This produces `libtfpsacrypto` (static by default; `-DUSE_SHARED_TF_PSA_CRYPTO_LIBRARY=On` for a shared lib).
4. **Run the test suite** (needs Python): `cd build && ctest`
5. **Disable tests** (no Python): `cmake -DENABLE_TESTING=Off -S .`
6. **Select a build mode**: `cmake -DCMAKE_BUILD_TYPE=Debug|Release|ASan|Check|TSan ...` (modes: Release, Debug, ASan, ASanDbg, MemSan, MemSanDbg, Check, TSan, TSanDbg).
7. **Generate API docs**: `cmake --build build --target tfpsacrypto-apidoc` (needs Doxygen ≥ 1.8.14) → `apidoc/index.html`.

### Using a custom config

- Replace `include/psa/crypto_config.h` directly, **or**
- Keep your config outside the tree and pass it at configure time:
  ```
  cmake -DTF_PSA_CRYPTO_CONFIG_FILE="/abs/path/my_crypto_config.h" -S . -B build
  ```
- Use a `configs/` preset (`crypto-config-ccm-aes-sha256.h`, `crypto-config-symmetric-only.h`) as a starting point.
- Programmatic edits: `python3 scripts/config.py --help` (set/get/unset `PSA_WANT_*` symbols).

## Code Generation Checklist

Before presenting PSA-crypto code to the user, verify:

- [ ] `#include <psa/crypto.h>` present (and `"tf-psa-crypto/build_info.h"` if version macros are needed)
- [ ] `psa_crypto_init()` called and its `psa_status_t` return checked before any other `psa_*` call
- [ ] Every key has `psa_set_key_type`, `psa_set_key_bits`, `psa_set_key_usage_flags`, `psa_set_key_algorithm` set before `psa_import_key` / `psa_generate_key`
- [ ] Usage flags include every operation actually performed (`ENCRYPT|DECRYPT`, `SIGN_HASH|VERIFY_HASH`, etc.)
- [ ] Algorithm passed to the operation matches the key's `psa_set_key_algorithm`
- [ ] Every multi-part operation object zero-initialized with the matching `PSA_*_OPERATION_INIT`
- [ ] Every `psa_status_t` return checked against `PSA_SUCCESS`
- [ ] On any error path: matching `psa_*_abort()` + `psa_destroy_key()`
- [ ] Output buffers sized with `PSA_HASH_LENGTH`/`PSA_MAC_MAX_SIZE`/`PSA_AEAD_TAG_LENGTH`/`PSA_SIGN_OUTPUT_SIZE`/`PSA_EXPORT_KEY_OUTPUT_SIZE` — no literals
- [ ] Required `PSA_WANT_*` symbols enabled in `crypto_config.h` for every key type, algorithm, and curve/group used
- [ ] Secrets wiped with `mbedtls_platform_zeroize` if exported into application memory
- [ ] `mbedtls_psa_crypto_free()` called at shutdown (optional; frees resources and volatile keys)

## Do Not Modify

- `include/psa/*` and `include/tf-psa-crypto/*` — public API headers (edit only `crypto_config.h` to toggle mechanisms)
- `core/*` — internal implementation
- `drivers/builtin/*`, `drivers/everest/*`, `drivers/p256-m/*` — bundled third-party / built-in driver code
- `framework/` — git submodule (upstream `TF-PSA-Crypto/mbedtls-framework`); never hand-edit
- `tests/*` — the upstream test suite
- Build-generated files (anything produced by `framework/scripts/make_generated_files.py`) — regenerate, do not patch

When integrating a **PSA cryptoprocessor driver**, follow `docs/psa-driver-example-and-guide.md` and `docs/proposed/psa-driver-developer-guide.md` — do **not** hand-edit the auto-generated `psa_crypto_driver_wrappers.h` for entry points that support auto-generation (`import_key`, `export_public_key`, opaque `export_key`/`copy_key`/`get_builtin_key`).
