# Consolidated Pitfalls (TF-PSA-Crypto / PSA Crypto)

A single-page index of the most common mistakes, cross-referenced from `SKILL.md`. Every entry is grounded in the API contracts in `include/psa/crypto.h` and the behavior shown in `programs/psa/*.c`.

## Lifecycle

### P1. Calling `psa_*` before `psa_crypto_init()`

- **Symptom**: `PSA_ERROR_BAD_STATE` or undefined behavior.
- **Cause**: The library must be initialized before any other PSA call; the RNG is seeded here.
- **Fix**: Call `psa_crypto_init()` at startup and check `status != PSA_SUCCESS` (note: `PSA_SUCCESS == 0`, so `if (!psa_crypto_init())` is wrong — that treats success as false).
- **Ref**: `include/psa/crypto.h` docstring of `psa_crypto_init`.

### P2. Confusing `psa_status_t` with Mbed TLS `int`

- **Symptom**: Wrong error handling; comparing against `MBEDTLS_ERR_*`.
- **Cause**: PSA returns `psa_status_t`; success is `PSA_SUCCESS == 0`, errors are negative `PSA_ERROR_*`.
- **Fix**: Always `if (status != PSA_SUCCESS)`. To decode a numeric value, run `programs/psa/psa_constant_names status <n>`.
- **Ref**: `docs/psa-transition.md` "Error codes" section; `include/psa/crypto_values.h`.

## Keys

### P3. Passing raw key bytes to an operation

- **Symptom**: Type error / compile failure, or passing a buffer where a `psa_key_id_t` is expected.
- **Cause**: All PSA crypto operations take a key *identifier*, not key bytes.
- **Fix**: `psa_import_key(&attributes, key_bytes, len, &key)` first, then pass `key`.
- **Ref**: `programs/psa/hmac_demo.c`, `programs/psa/crypto_examples.c`.

### P4. Missing usage flag → `PSA_ERROR_NOT_PERMITTED`

- **Symptom**: Operation returns `PSA_ERROR_NOT_PERMITTED`.
- **Cause**: The key's `psa_set_key_usage_flags` doesn't include the flag for the operation (`SIGN_HASH`, `ENCRYPT`, `DERIVE`, …).
- **Fix**: Combine flags with `|` when creating the key, e.g. `PSA_KEY_USAGE_ENCRYPT | PSA_KEY_USAGE_DECRYPT`.
- **Ref**: `include/psa/crypto_values.h` `PSA_KEY_USAGE_*`.

### P5. Algorithm in the operation differs from the key's algorithm

- **Symptom**: `PSA_ERROR_NOT_PERMITTED` even with the right usage flag.
- **Cause**: `psa_set_key_algorithm` pins the key's algorithm; using a different one is rejected.
- **Fix**: Keep them consistent (especially for AEAD short-tag variants — use the same `PSA_ALG_AEAD_WITH_SHORTENED_TAG(...)` on both the key and the operation).
- **Ref**: `programs/psa/aead_demo.c`.

### P6. Forgetting `psa_destroy_key` → leaked key-store slot

- **Symptom**: Over time, key creation starts failing (slots exhausted).
- **Cause**: Each key occupies a key-store entry until destroyed or `mbedtls_psa_crypto_free()` runs.
- **Fix**: Destroy in a cleanup label; `psa_destroy_key(PSA_KEY_ID_NULL)` is a safe no-op.
- **Ref**: `docs/psa-transition.md` "Key management".

### P7. Persistent key id out of range / collision

- **Symptom**: `PSA_ERROR_INVALID_ARGUMENT` or `PSA_ERROR_ALREADY_EXISTS`.
- **Cause**: id 0 is `PSA_KEY_ID_NULL`; ids > `PSA_KEY_ID_USER_MAX` (0x3fffffff) are reserved; reusing an existing id with different attributes collides.
- **Fix**: Use a stable id in `[PSA_KEY_ID_USER_MIN (0x1), PSA_KEY_ID_USER_MAX]`; destroy before recreating.
- **Ref**: `include/psa/crypto_values.h`.

## Operations (multi-part)

### P8. Operation object not zero-initialized

- **Symptom**: `PSA_ERROR_BAD_STATE` or memory garbage.
- **Cause**: Stack-allocated operation structs contain garbage until zeroed.
- **Fix**: Use the matching `PSA_*_OPERATION_INIT` initializer (or `memset(&op, 0, sizeof(op))`).
- **Ref**: `programs/psa/crypto_examples.c` (`PSA_CIPHER_OPERATION_INIT`), `include/psa/crypto_struct.h`.

### P9. Reusing an operation after an error without abort

- **Symptom**: `PSA_ERROR_BAD_STATE` on subsequent calls.
- **Cause**: Any non-success (with rare exceptions) puts the operation in an error state.
- **Fix**: Always call the matching `psa_*_abort` on the error path. It is harmless on the success path, so call it unconditionally in cleanup.
- **Ref**: `programs/psa/hmac_demo.c` `psa_mac_abort` in `exit:`.

### P10. Wrong multi-part call order

- **Symptom**: `PSA_ERROR_BAD_STATE`.
- **Cause**: Each operation class has a required phase order.
- **Fix**:
  - Hash: `setup → update* → finish|verify`
  - MAC: `sign_setup|verify_setup → update* → sign_finish|verify_finish`
  - Cipher: `encrypt_setup|decrypt_setup → generate_iv|set_iv → update* → finish`
  - AEAD: `*_setup → set_nonce|generate_nonce → (set_lengths) → update_ad → update* → finish|verify`
  - KDF: `setup → input* (all steps before any output) → output_bytes|output_key`
- **Ref**: each function's docstring in `include/psa/crypto.h`.

## Buffer sizing

### P11. Output buffer too small → `PSA_ERROR_BUFFER_TOO_SMALL`

- **Symptom**: `PSA_ERROR_BUFFER_TOO_SMALL`.
- **Cause**: Output buffer is a hard-coded literal that's smaller than the algorithm's output.
- **Fix**: Use the size macros — `PSA_HASH_LENGTH(alg)`, `PSA_MAC_MAX_SIZE`, `PSA_AEAD_TAG_LENGTH(type,bits,alg)`, `PSA_SIGN_OUTPUT_SIZE(type,bits,alg)`, `PSA_EXPORT_KEY_OUTPUT_SIZE(type,bits)`, `PSA_AEAD_ENCRYPT_OUTPUT_SIZE(...)`.
- **Ref**: `programs/psa/psa_hash.c`, `programs/psa/aead_demo.c`.

## Specific algorithm pitfalls

### P12. CBC no-padding with non-block-aligned input

- **Symptom**: `PSA_ERROR_INVALID_ARGUMENT`.
- **Cause**: `PSA_ALG_CBC_NO_PADDING` requires input length to be a multiple of the block size (16 for AES).
- **Fix**: Use `PSA_ALG_CBC_PKCS7`, or pad manually.
- **Ref**: `include/psa/crypto_values.h`.

### P13. AEAD nonce reuse with the same key

- **Symptom**: Catastrophic loss of confidentiality (GCM/CCM).
- **Cause**: Reusing a (key, nonce) pair reveals the keystream and can forge tags.
- **Fix**: Generate a fresh nonce per encryption (`psa_aead_generate_nonce`) or use a counter that never wraps under the same key.
- **Ref**: `programs/psa/key_ladder_demo.c` uses `psa_generate_random(header.iv, ...)` per wrap.

### P14. Raw ECDH output used directly as a symmetric key

- **Symptom**: Insecure shared secret; non-uniform key.
- **Cause**: `psa_raw_key_agreement` outputs the raw DH shared secret, which should be fed to a KDF.
- **Fix**: Use `psa_key_derivation_key_agreement` to feed the secret into HKDF, then `psa_key_derivation_output_key`.
- **Ref**: `include/psa/crypto.h`; `recipes/key_derivation.md`.

### P15. HKDF/PBKDF2 capacity exhausted

- **Symptom**: `PSA_ERROR_INSUFFICIENT_DATA`.
- **Cause**: Requested output exceeds the operation's capacity.
- **Fix**: Call `psa_key_derivation_set_capacity(op, PSA_KEY_DERIVATION_UNLIMITED_CAPACITY)` (== `(size_t)-1`) or a value ≥ output length, before reading.
- **Ref**: `include/psa/crypto.h` `psa_key_derivation_output_bytes` docstring.

## Build / config

### P16. `PSA_ERROR_NOT_SUPPORTED` for an algorithm/key

- **Symptom**: Code compiles but runtime returns `PSA_ERROR_NOT_SUPPORTED`.
- **Cause**: The matching `PSA_WANT_*` is not defined in `crypto_config.h`.
- **Fix**: Enable the symbol (key type + algorithm + curve/group) and rebuild; verify with `programs/psa/psa_constant_names` if unsure.
- **Ref**: `include/psa/crypto_config.h`; `resources/config_reference.md`.

### P17. Changing `CC`/`CFLAGS` after the first CMake call has no effect

- **Symptom**: Build still uses the old compiler/flags.
- **Cause**: CMake caches compiler settings on first invocation.
- **Fix**: Delete the build directory and re-run `cmake`, or re-run the configuration phase with the new settings.
- **Ref**: `README.md` "CMake" section.

### P18. Missing generated source files (dev branch)

- **Symptom**: Build fails on files that should exist.
- **Cause**: The dev branch omits configuration-independent generated files; release tarballs include them.
- **Fix**: `git submodule update --init` then `python3 framework/scripts/make_generated_files.py` (or let CMake generate them on non-Windows non-cross builds).
- **Ref**: `README.md` "Generated source files in the development branch".

### P19. `mbedtls_psa_crypto_free` undeclared

- **Symptom**: Compile error.
- **Cause**: This helper is declared in `psa/crypto_extra.h`, not `psa/crypto.h`.
- **Fix**: `#include "psa/crypto_extra.h"`.
- **Ref**: `include/psa/crypto_extra.h`.

## Quick decision table

| You see … | First check |
|---|---|
| `PSA_ERROR_BAD_STATE` | (a) `psa_crypto_init()` done? (b) operation object initialized? (c) call order? (d) abort after prior error? |
| `PSA_ERROR_NOT_PERMITTED` | usage flags? algorithm matches key's algorithm? |
| `PSA_ERROR_NOT_SUPPORTED` | `PSA_WANT_*` enabled in `crypto_config.h`? |
| `PSA_ERROR_BUFFER_TOO_SMALL` | using `PSA_*_SIZE`/`PSA_*_LENGTH` macros? |
| `PSA_ERROR_INVALID_ARGUMENT` | input length valid for the mode (e.g. CBC block alignment)? key bits match import length? |
| `PSA_ERROR_INVALID_SIGNATURE` | for verify: tag/MAC/signature/hash genuinely mismatched (treat as data-integrity failure). |
| `PSA_ERROR_INSUFFICIENT_ENTROPY` | RNG could not be seeded — port/provide an entropy source. |
| `PSA_ERROR_INSUFFICIENT_DATA` | KDF capacity too small for requested output. |
