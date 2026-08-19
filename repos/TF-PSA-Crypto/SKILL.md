---
name: tf-psa-crypto-skill
description: >-
  AI Skill for the TF-PSA-Crypto library — the reference implementation of the
  PSA Cryptography API (v1.2). Used when users need to create, modify, or debug
  applications that use PSA Crypto (psa_*) APIs for hashing, MAC, symmetric/AEAD
  cipher, asymmetric signatures, key agreement, key derivation, key management,
  build configuration (crypto_config.h), or PSA driver integration.
  Trigger words: "PSA Crypto", "PSA Cryptography", "TF-PSA-Crypto", "psa_crypto",
  "psa/crypto.h", "psa_sign", "psa_aead", "psa_key_derivation", "Mbed TLS crypto",
  "PSA加密", "PSA密码学", "密钥派生", "AEAD", "HMAC", "ECDSA", "RSA-PSS", "crypto_config.h"
license: Apache-2.0 OR GPL-2.0-or-later
metadata:
  author: Community
  version: "1.1.0"
---

# TF-PSA-Crypto-skill

AI Skill for the TF-PSA-Crypto library, the reference implementation of the **PSA Cryptography API version 1.2**. Provides scenario-driven recipes, complete `psa_*` API reference, `crypto_config.h` configuration guidance, and common pitfalls — all grounded in the repository's own headers (`include/psa/crypto.h`, `crypto_values.h`, `crypto_sizes.h`, `crypto_struct.h`), example programs (`programs/psa/`), and documentation (`docs/`).

## Core Principles

1. **Always call `psa_crypto_init()` first** — Any other `psa_*` call before a successful `psa_crypto_init()` is undefined behavior; implementations are encouraged to return `PSA_ERROR_BAD_STATE`. Initialize once; repeated successful calls are guaranteed to succeed.
2. **Keys are opaque identifiers, not buffers** — All crypto takes a `psa_key_id_t` (typedef of `mbedtls_svc_key_id_t`), never raw key bytes. Create with `psa_import_key()` / `psa_generate_key()`, then `psa_destroy_key()` when done. Key material can live entirely inside an isolation boundary.
3. **Attributes describe the key, set before creation** — Use a `psa_key_attributes_t` (init with `PSA_KEY_ATTRIBUTES_INIT`), set type/usage/algorithm/bits via `psa_set_key_*()`, pass to the creation function, then optionally `psa_reset_key_attributes()`. The same attributes object is NOT the operation handle.
4. **Usage flags gate every operation** — `PSA_KEY_USAGE_SIGN_HASH`/`SIGN_MESSAGE`, `ENCRYPT`/`DECRYPT`, `DERIVE`, `EXPORT`, `VERIFY_*`. A missing flag returns `PSA_ERROR_NOT_PERMITTED`. Combine with `|`.
5. **Algorithm is bound to the key, and must match the operation** — `psa_set_key_algorithm()` pins the key's permitted algorithm; an operation using a different algorithm returns `PSA_ERROR_NOT_PERMITTED`.
6. **Multi-part operations must be aborted on error** — `psa_hash_abort`, `psa_mac_abort`, `psa_cipher_abort`, `psa_aead_abort`, `psa_key_derivation_abort`. After any non-success status the operation object enters an error state; abort is required (harmless on success).
7. **Operation objects must be zero-initialized** — Use `PSA_HASH_OPERATION_INIT`, `PSA_MAC_OPERATION_INIT`, `PSA_CIPHER_OPERATION_INIT`, `PSA_AEAD_OPERATION_INIT`, `PSA_KEY_DERIVATION_OPERATION_INIT` (or `memset(&op, 0, sizeof(op))`).
8. **Size output buffers with PSA macros, not guesses** — `PSA_HASH_LENGTH(alg)`, `PSA_MAC_MAX_SIZE`, `PSA_AEAD_TAG_LENGTH(type,bits,alg)`, `PSA_SIGN_OUTPUT_SIZE(type,bits,alg)`, `PSA_EXPORT_KEY_OUTPUT_SIZE(type,bits)`. A too-small buffer yields `PSA_ERROR_BUFFER_TOO_SMALL`.
9. **Return type is `psa_status_t`** — `PSA_SUCCESS == 0`; all errors are negative (`PSA_ERROR_*`). Do not compare with `MBEDTLS_ERR_*` style; always check `status != PSA_SUCCESS`.
10. **PSA owns the RNG** — No RNG argument is passed to `psa_*` functions; the library maintains a global RNG seeded during `psa_crypto_init()`. `psa_generate_random()` reads from it. A bad seed surfaces as `PSA_ERROR_INSUFFICIENT_ENTROPY` from `psa_crypto_init()`.
11. **Mechanism availability is compile-time** — `include/psa/crypto_config.h` lists `PSA_WANT_KEY_TYPE_*`, `PSA_WANT_ALG_*`, `PSA_WANT_ECC_*`, `PSA_WANT_DH_*`. An unsupported algorithm/key returns `PSA_ERROR_NOT_SUPPORTED`.
12. **Persistent keys survive reboots** — `psa_set_key_id()` + `psa_set_key_lifetime(PSA_KEY_LIFETIME_PERSISTENT)` writes to storage on creation. IDs range `PSA_KEY_ID_USER_MIN` (0x1) to `PSA_KEY_ID_USER_MAX` (0x3fffffff). Volatile keys (default) die on `mbedtls_psa_crypto_free()`.

## When to Use

**Applicable:**
- Writing application code that calls the `psa_*` / `psa/crypto.h` API
- Implementing hashing (SHA-256/384/512, SHA-3), HMAC, CMAC
- Symmetric encryption (AES-CBC/CTR/CFB/OFB/ECB, ChaCha20)
- AEAD (AES-GCM, AES-CCM, ChaCha20-Poly1305)
- Asymmetric signatures (RSA-PSS/PKCS1v15, ECDSA, Ed25519)
- Key agreement (ECDH, FFDH) and key derivation (HKDF, PBKDF2, TLS12-PRF)
- Key management (generate, import, export, persistent keys)
- Configuring `crypto_config.h` / using `configs/` presets or `scripts/config.py`
- Building/integrating TF-PSA-Crypto via CMake (static/shared, subproject, `find_package`)
- Migrating code from legacy `mbedtls_*` crypto APIs to `psa_*` APIs

**Not applicable:**
- TLS/X.509/secure-boot protocol design (those are higher layers; TF-PSA-Crypto is the crypto primitives layer)
- The legacy `mbedtls_aes_*`, `mbedtls_md_*`, `mbedtls_rsa_*` APIs as an end-state (they are being removed; use PSA equivalents)
- Hardware/PCB design or RNG entropy source selection at the board level
- Writing a PSA cryptoprocessor driver from scratch without first reading `recipes/psa_driver_development.md` (which summarizes `docs/psa-driver-example-and-guide.md` and `docs/proposed/psa-driver-developer-guide.md`)

---

## Scenario Quick Reference (Recipes)

When the user intent matches a scenario below, **read the corresponding recipe first** — it contains the full call chain, step-by-step instructions, common errors, and copy-able code adapted from the repo's own `programs/psa/` examples.

### Initialization & Library Lifecycle

| recipe | scenario |
|---|---|
| `recipes/init_and_lifecycle.md` | Initialize the library, manage keys, and shut down cleanly (`psa_crypto_init` / `mbedtls_psa_crypto_free`) |

### Key Management

| recipe | scenario |
|---|---|
| `recipes/key_management.md` | Generate, import, export, and destroy keys; volatile vs. persistent lifetimes |

### Hashing

| recipe | scenario |
|---|---|
| `recipes/hashing.md` | One-shot `psa_hash_compute` and multi-part `psa_hash_setup/update/finish`, plus clone |

### MAC (HMAC / CMAC)

| recipe | scenario |
|---|---|
| `recipes/mac.md` | One-shot and multi-part HMAC (SHA-256) and AES-CMAC; sign vs. verify |

### Symmetric Cipher (AES / ChaCha20)

| recipe | scenario |
|---|---|
| `recipes/cipher.md` | AES-CBC/CTR one-shot and multi-part encrypt/decrypt, IV generation |

### AEAD (GCM / CCM / ChaCha20-Poly1305)

| recipe | scenario |
|---|---|
| `recipes/aead.md` | Multi-part AEAD with additional data, nonce handling, short tags |

### Asymmetric Signatures & Key Agreement

| recipe | scenario |
|---|---|
| `recipes/asymmetric.md` | ECDSA/RSA sign+verify, `psa_export_public_key`, ECDH raw key agreement |

### Key Derivation

| recipe | scenario |
|---|---|
| `recipes/key_derivation.md` | HKDF key ladder, `psa_key_derivation_output_key`, TLS12-PRF inputs |

### Build & Configuration

| recipe | scenario |
|---|---|
| `recipes/build_and_config.md` | CMake build modes, `crypto_config.h` editing, `configs/` presets, `TF_PSA_CRYPTO_CONFIG_FILE` |

### Migration & Driver Integration

| recipe | scenario |
|---|---|
| `recipes/migration_to_psa.md` | Migrate legacy `mbedtls_*` crypto APIs (AES/SHA/HMAC/RSA/ECDSA/ECDH/HKDF) to `psa_*`; RNG-callback removal, `MBEDTLS_xxx_C`→`PSA_WANT_*` config split, error-code merge, PK/PAKE changes |
| `recipes/psa_driver_development.md` | Write/integrate PSA cryptoprocessor drivers (transparent accelerator / opaque secure element); JSON driver description, manual 7-step wrapper integration, driver-only builds (`PSA_WANT_*`+`MBEDTLS_PSA_ACCEL_*`+`MBEDTLS_xxx_C`) |

---

## Key Type Quick Reference

All values from `include/psa/crypto_values.h`. Mechanism availability is gated by matching `PSA_WANT_*` in `include/psa/crypto_config.h`.

| Category | Macro | Notes |
|---|---|---|
| Raw/symmetric | `PSA_KEY_TYPE_RAW_DATA` (0x1001) |Opaque data, e.g. public key export |
| | `PSA_KEY_TYPE_HMAC` (0x1100) | Any-length HMAC key |
| | `PSA_KEY_TYPE_DERIVE` (0x1200) | Input to key derivation |
| | `PSA_KEY_TYPE_PASSWORD` (0x1203) | PBKDF2 input |
| | `PSA_KEY_TYPE_PASSWORD_HASH` (0x1205) | PBKDF2 input |
| | `PSA_KEY_TYPE_PEPPER` (0x1206) | PBKDF2 pepper |
| Block cipher | `PSA_KEY_TYPE_AES` (0x2400) | 128/192/256-bit |
| | `PSA_KEY_TYPE_ARIA` (0x2406) | Korean block cipher |
| | `PSA_KEY_TYPE_CAMELLIA` (0x2403) | |
| | `PSA_KEY_TYPE_CHACHA20` (0x2004) | Stream cipher |
| RSA | `PSA_KEY_TYPE_RSA_PUBLIC_KEY` (0x4001) / `PSA_KEY_TYPE_RSA_KEY_PAIR` (0x7001) | |
| ECC | `PSA_KEY_TYPE_ECC_KEY_PAIR(curve)` / `PSA_KEY_TYPE_ECC_PUBLIC_KEY(curve)` | curve is a `PSA_ECC_FAMILY_*` |
| DH (FFDH) | `PSA_KEY_TYPE_DH_KEY_PAIR(group)` / `PSA_KEY_TYPE_DH_PUBLIC_KEY(group)` | group is `PSA_DH_FAMILY_RFC7919_*` |

### ECC Families (`PSA_ECC_FAMILY_*`, from `crypto_values.h`)

| Family | Curves (PSA_WANT_ECC_*) |
|---|---|
| `SECP_R1` (0x12) | secp192r1*, secp256r1, secp384r1, secp521r1 |
| `SECP_K1` (0x17) | secp192k1*, secp256k1 |
| `SECP_R2` (0x1b) | (rare) |
| `SECT_*` (0x22/0x27/0x2b) | Binary curves (not enabled by default) |
| `BRAINPOOL_P_R1` (0x30) | 256/384/512 |
| `MONTGOMERY` (0x41) | Curve25519, Curve448 |
| `TWISTED_EDWARDS` (0x42) | Ed25519, Ed448 |

\* secp192r1/secp192k1 are not in the public default config (internal testing only).

---

## Algorithm Quick Reference (subset)

From `include/psa/crypto_values.h`. Builders `PSA_ALG_HMAC(hash)`, `PSA_ALG_RSA_PSS(hash)`, `PSA_ALG_RSA_PKCS1V15_SIGN(hash)`, `PSA_ALG_ECDSA(hash)`, `PSA_ALG_RSA_OAEP(hash)`, `PSA_ALG_HKDF(hash)`, `PSA_ALG_HKDF_EXPAND(hash)`, `PSA_ALG_TLS12_PRF(hash)` take a hash algorithm.

| Class | Macros |
|---|---|
| Hash | `PSA_ALG_MD5`, `PSA_ALG_SHA_1`, `PSA_ALG_SHA_224`, `PSA_ALG_SHA_256`, `PSA_ALG_SHA_384`, `PSA_ALG_SHA_512`, `PSA_ALG_SHA3_256`, `PSA_ALG_SHA3_512`, `PSA_ALG_RIPEMD160` |
| MAC | `PSA_ALG_HMAC(hash)` (over any hash), `PSA_ALG_CMAC` |
| Cipher | `PSA_ALG_STREAM_CIPHER`, `PSA_ALG_CTR`, `PSA_ALG_CBC_NO_PADDING`, `PSA_ALG_CBC_PKCS7`, `PSA_ALG_CFB`, `PSA_ALG_OFB`, `PSA_ALG_ECB_NO_PADDING` |
| AEAD | `PSA_ALG_GCM`, `PSA_ALG_CCM`, `PSA_ALG_CHACHA20_POLY1305`, `PSA_ALG_CCM_STAR_NO_TAG` |
| Sign | `PSA_ALG_RSA_PKCS1V15_SIGN(hash)`, `PSA_ALG_RSA_PSS(hash)`, `PSA_ALG_ECDSA(hash)`, `PSA_ALG_DETERMINISTIC_ECDSA(hash)`, `PSA_ALG_PURE_EDDSA`, `PSA_ALG_ED25519PH`, `PSA_ALG_ED448PH` |
| Key agreement | `PSA_ALG_ECDH`, `PSA_ALG_FFDH` |
| KDF | `PSA_ALG_HKDF(hash)`, `PSA_ALG_HKDF_EXPAND(hash)`, `PSA_ALG_PBKDF2_HMAC(hash)`, `PSA_ALG_PBKDF2_AES_CMAC_PRF_128`, `PSA_ALG_TLS12_PRF(hash)`, `PSA_ALG_TLS12_PSK_TO_MS(hash)` |

Use `PSA_ALG_AEAD_WITH_SHORTENED_TAG(aead_alg, tag_len)` for a truncated AEAD tag (e.g. AES-GCM with an 8-byte tag).

---

## Key Usage Flags (`PSA_KEY_USAGE_*`, from `crypto_values.h`)

| Flag | Value | Permits |
|---|---|---|
| `PSA_KEY_USAGE_EXPORT` | 0x00000001 | `psa_export_key` |
| `PSA_KEY_USAGE_COPY` | 0x00000002 | `psa_copy_key` |
| `PSA_KEY_USAGE_DECRYPT` | 0x00000200 | `psa_cipher_decrypt_*`, `psa_aead_decrypt_*` |
| `PSA_KEY_USAGE_DERIVE_PUBLIC` | 0x00000080 | Derive the public key (for asymmetric key pairs) |
| `PSA_KEY_USAGE_ENCRYPT` | 0x00000100 | `psa_cipher_encrypt_*`, `psa_aead_encrypt_*` |
| `PSA_KEY_USAGE_SIGN_MESSAGE` | 0x00000400 | `psa_sign_message` |
| `PSA_KEY_USAGE_VERIFY_MESSAGE` | 0x00000800 | `psa_verify_message` |
| `PSA_KEY_USAGE_SIGN_HASH` | 0x00001000 | `psa_sign_hash` |
| `PSA_KEY_USAGE_VERIFY_HASH` | 0x00002000 | `psa_verify_hash` |
| `PSA_KEY_USAGE_DERIVE` | 0x00004000 | `psa_key_derivation_input_key`, key agreement |
| `PSA_KEY_USAGE_VERIFY_DERIVATION` | 0x00008000 | `psa_key_derivation_verify_*` |

---

## Key Derivation Input Steps (`PSA_KEY_DERIVATION_INPUT_*`)

| Step | Value | Typical use |
|---|---|---|
| `PSA_KEY_DERIVATION_INPUT_SECRET` | 0x0101 | Secret (via `psa_key_derivation_input_key`) |
| `PSA_KEY_DERIVATION_INPUT_OTHER_SECRET` | 0x0102 | Second secret (PSK-to-MS) |
| `PSA_KEY_DERIVATION_INPUT_LABEL` | 0x0201 | Label (TLS12-PRF) |
| `PSA_KEY_DERIVATION_INPUT_SALT` | 0x0202 | Salt (HKDF/PBKDF2) |
| `PSA_KEY_DERIVATION_INPUT_INFO` | 0x0203 | Info (HKDF) |
| `PSA_KEY_DERIVATION_INPUT_SEED` | 0x0204 | Seed (TLS12-PRF) |

Capacity set via `psa_key_derivation_set_capacity(op, n)`; `PSA_KEY_DERIVATION_UNLIMITED_CAPACITY == ((size_t)-1)`.

---

## Common Error Codes (`PSA_ERROR_*`, from `crypto_values.h`)

| Code | Value | Meaning |
|---|---|---|
| `PSA_SUCCESS` | 0 | Success |
| `PSA_ERROR_NOT_SUPPORTED` | -134 | Algorithm/key type not compiled in |
| `PSA_ERROR_NOT_PERMITTED` | -133 | Missing usage flag or algorithm mismatch |
| `PSA_ERROR_BUFFER_TOO_SMALL` | -138 | Output buffer too small |
| `PSA_ERROR_INVALID_ARGUMENT` | -135 | Bad parameter (e.g. wrong key size) |
| `PSA_ERROR_INVALID_HANDLE` | -136 | Unknown key id |
| `PSA_ERROR_BAD_STATE` | -137 | Not initialized, or operation in wrong phase |
| `PSA_ERROR_INSUFFICIENT_MEMORY` | -141 | Allocation failed |
| `PSA_ERROR_INSUFFICIENT_DATA` | -143 | Derivation capacity exhausted |
| `PSA_ERROR_INSUFFICIENT_ENTROPY` | -148 | RNG could not be seeded |
| `PSA_ERROR_INVALID_SIGNATURE` | -149 | MAC/signature/hash verify mismatch |
| `PSA_ERROR_INVALID_PADDING` | -150 | Bad padding on decrypt |
| `PSA_ERROR_CORRUPTION_DETECTED` | -151 | Internal integrity check failed |
| `PSA_ERROR_STORAGE_FAILURE` / `DATA_CORRUPT` / `DATA_INVALID` | -146/-152/-153 | Persistent key store problems |

---

## Critical Pitfalls (Must Read)

The most common errors. Violating any of these produces broken or insecure code.

### 1. Forget `psa_crypto_init()` — everything else is undefined

```c
// WRONG — using PSA before init; returns PSA_ERROR_BAD_STATE or undefined
psa_key_attributes_t attr = PSA_KEY_ATTRIBUTES_INIT;
psa_set_key_type(&attr, PSA_KEY_TYPE_AES);
psa_key_id_t key;
psa_generate_key(&attr, &key);   // may fail or behave unpredictably

// CORRECT — init once at startup, check the return value
psa_status_t status = psa_crypto_init();
if (status != PSA_SUCCESS) { /* abort startup */ }

psa_key_attributes_t attr = PSA_KEY_ATTRIBUTES_INIT;
psa_set_key_type(&attr, PSA_KEY_TYPE_AES);
psa_set_key_bits(&attr, 256);
psa_set_key_usage_flags(&attr, PSA_KEY_USAGE_ENCRYPT | PSA_KEY_USAGE_DECRYPT);
psa_set_key_algorithm(&attr, PSA_ALG_CTR);
psa_key_id_t key;
status = psa_generate_key(&attr, &key);
```

### 2. Pass raw key bytes to an operation — there is no such API

```c
// WRONG — psa_cipher_encrypt takes a key id, not a key buffer
uint8_t aes_key[32] = { ... };
psa_cipher_encrypt(aes_key, PSA_ALG_CTR, iv, 16, in, in_len, out, out_size, &olen);
                          ^^^^^^^ type error: expected mbedtls_svc_key_id_t

// CORRECT — import first, then use the id
psa_key_attributes_t attr = PSA_KEY_ATTRIBUTES_INIT;
psa_set_key_type(&attr, PSA_KEY_TYPE_AES);
psa_set_key_bits(&attr, 256);
psa_set_key_usage_flags(&attr, PSA_KEY_USAGE_ENCRYPT);
psa_set_key_algorithm(&attr, PSA_ALG_CTR);
psa_key_id_t key;
psa_import_key(&attr, aes_key, sizeof(aes_key), &key);
psa_cipher_encrypt(key, PSA_ALG_CTR, iv, 16, in, in_len, out, out_size, &olen);
// ... later
psa_destroy_key(key);
```

### 3. Missing usage flag or algorithm mismatch

```c
// WRONG — key only permits ENCRYPT, but we call DECRYPT
psa_set_key_usage_flags(&attr, PSA_KEY_USAGE_ENCRYPT);
psa_set_key_algorithm(&attr, PSA_ALG_CBC_PKCS7);
psa_import_key(&attr, k, 32, &key);
psa_cipher_decrypt(key, PSA_ALG_CBC_PKCS7, iv, 16, ct, ct_len, ...);
//  -> PSA_ERROR_NOT_PERMITTED

// CORRECT — combine both flags with OR
psa_set_key_usage_flags(&attr, PSA_KEY_USAGE_ENCRYPT | PSA_KEY_USAGE_DECRYPT);
```

### 4. Reuse a multi-part operation without abort after an error

```c
// WRONG — op is in error state; further calls return PSA_ERROR_BAD_STATE
psa_mac_sign_setup(&op, key, alg);
if (psa_mac_update(&op, data, len) != PSA_SUCCESS) {
    /* op now in error state */
}
psa_mac_sign_setup(&op, key, alg);   // WRONG: must abort first

// CORRECT — always abort on the error path (harmless on success)
exit:
    psa_mac_abort(&op);
```

### 5. Operation object not zero-initialized

```c
// WRONG — stack garbage in the operation struct
psa_cipher_operation_t op;          // uninitialized
psa_cipher_encrypt_setup(&op, key, PSA_ALG_CBC_PKCS7);   // UB / BAD_STATE

// CORRECT — use the INIT macro or memset
psa_cipher_operation_t op = PSA_CIPHER_OPERATION_INIT;
psa_cipher_encrypt_setup(&op, key, PSA_ALG_CBC_PKCS7);
```

### 6. Output buffer sized by guessing instead of a PSA macro

```c
// WRONG — hash length depends on the algorithm
uint8_t hash[16];   // too small for SHA-256
psa_hash_compute(PSA_ALG_SHA_256, msg, msg_len, hash, sizeof(hash), &hash_len);
//  -> PSA_ERROR_BUFFER_TOO_SMALL

// CORRECT — size from the algorithm
uint8_t hash[PSA_HASH_LENGTH(PSA_ALG_SHA_256)];   // 32 bytes
psa_hash_compute(PSA_ALG_SHA_256, msg, msg_len, hash, sizeof(hash), &hash_len);
```

### 7. Treat `psa_status_t` like an Mbed TLS `int`

```c
// WRONG — mixing conventions
if (status == 0) { /* ok */ }            // works, but unreadable
if (status == MBEDTLS_ERR_xxx) { ... }   // WRONG: PSA returns PSA_ERROR_* not MBEDTLS_ERR_*

// CORRECT — always compare against PSA_SUCCESS / PSA_ERROR_*
if (status != PSA_SUCCESS) { handle_error(status); }
```

### 8. Forgetting to `psa_destroy_key` — leaks a key-store slot

```c
// WRONG — key occupies a slot until mbedtls_psa_crypto_free()
psa_generate_key(&attr, &key);
use_key(key);
return;   // slot leaked

// CORRECT — destroy when done; psa_destroy_key(PSA_KEY_ID_NULL)==PSA_SUCCESS (no-op)
use_key(key);
exit:
    psa_destroy_key(key);
```

### 9. Persistent key id collision / out of range

```c
// WRONG — id 0 is PSA_KEY_ID_NULL; ids above 0x3fffffff are vendor/null
psa_set_key_id(&attr, 0);                       // volatile, not persistent
psa_set_key_id(&attr, 0x80000000);              // out of user range

// CORRECT — use a stable id in [PSA_KEY_ID_USER_MIN, PSA_KEY_ID_USER_MAX]
#define MY_DEVICE_KEY_ID ((psa_key_id_t)1)
psa_set_key_id(&attr, MY_DEVICE_KEY_ID);
psa_set_key_lifetime(&attr, PSA_KEY_LIFETIME_PERSISTENT);
```

### 10. CBC no-padding with non-block-aligned input

```c
// WRONG — CBC_NO_PADDING requires input length to be a multiple of the block
psa_set_key_algorithm(&attr, PSA_ALG_CBC_NO_PADDING);
psa_cipher_encrypt(key, PSA_ALG_CBC_NO_PADDING, iv, 16,
                   in, 13 /* not 16 */, ...);   // PSA_ERROR_INVALID_ARGUMENT

// CORRECT — either pad to a block boundary or use CBC_PKCS7
psa_set_key_algorithm(&attr, PSA_ALG_CBC_PKCS7);
```

### 11. AEAD: forgetting to set nonce / additional data before update

```c
// WRONG — calling psa_aead_update before set_nonce + update_ad
psa_aead_encrypt_setup(&op, key, PSA_ALG_GCM);
psa_aead_update(&op, pt, pt_len, ...);   // PSA_ERROR_BAD_STATE

// CORRECT — order: setup -> set_nonce -> update_ad -> update -> finish
psa_aead_encrypt_setup(&op, key, PSA_ALG_GCM);
psa_aead_generate_nonce(&op, nonce, sizeof(nonce), &nonce_len);
psa_aead_update_ad(&op, ad, ad_len);
psa_aead_update(&op, pt, pt_len, ct, ct_size, &olen);
psa_aead_finish(&op, ct+olen, ct_size-olen, &olen2, tag, sizeof(tag), &tag_len);
```

### 12. Build with wrong key-type enablement — `PSA_ERROR_NOT_SUPPORTED`

```c
// Code compiles, but at runtime psa_generate_key returns PSA_ERROR_NOT_SUPPORTED
psa_set_key_type(&attr, PSA_KEY_TYPE_RSA_KEY_PAIR);
psa_generate_key(&attr, &key);   // if PSA_WANT_KEY_TYPE_RSA_KEY_PAIR_GENERATE undefined

// FIX — enable the matching PSA_WANT_* in include/psa/crypto_config.h, e.g.:
//   #define PSA_WANT_KEY_TYPE_RSA_KEY_PAIR_GENERATE 1
// then rebuild. Verify with programs/psa/psa_constant_names if unsure.
```

### 13. HMAC key length not set / truncated

```c
// WRONG — key_bits unset; some implementations pick a suboptimal length
psa_set_key_type(&attr, PSA_KEY_TYPE_HMAC);
psa_import_key(&attr, key_bytes, sizeof(key_bytes), &key);

// CORRECT — set key_bits = 8 * key_byte_length (matches hmac_demo.c)
psa_set_key_usage_flags(&attr, PSA_KEY_USAGE_SIGN_MESSAGE);
psa_set_key_algorithm(&attr, PSA_ALG_HMAC(PSA_ALG_SHA_256));
psa_set_key_type(&attr, PSA_KEY_TYPE_HMAC);
psa_set_key_bits(&attr, 8 * sizeof(key_bytes));
psa_import_key(&attr, key_bytes, sizeof(key_bytes), &key);
```

### 14. Derivation capacity smaller than requested output

```c
// WRONG — request 48 bytes but capacity defaults low / not set
psa_key_derivation_setup(&op, PSA_ALG_HKDF(PSA_ALG_SHA_256));
/* ...inputs... */
uint8_t out[48];
psa_key_derivation_output_bytes(&op, out, 48);
//  -> PSA_ERROR_INSUFFICIENT_DATA if capacity < 48

// CORRECT — set capacity to UNLIMITED or >= output length before reading
psa_key_derivation_set_capacity(&op, PSA_KEY_DERIVATION_UNLIMITED_CAPACITY);
psa_key_derivation_output_bytes(&op, out, 48);
```

---

## Execution Workflow

| Step | Name | Description |
|------|------|-------------|
| 1 | Understand | Identify the crypto operation class (hash / MAC / cipher / AEAD / sign / KDF / key mgmt) |
| 2 | Recipe | Match intent to a recipe in `recipes/`; follow its call chain verbatim |
| 3 | Query API | For any function not in the recipe, look up the exact signature in `resources/api_reference.md` (sourced from `include/psa/crypto.h`) |
| 4 | Check config | Confirm the needed `PSA_WANT_KEY_TYPE_*` / `PSA_WANT_ALG_*` / `PSA_WANT_ECC_*` are defined in `include/psa/crypto_config.h` (`resources/config_reference.md`) |
| 5 | Confirm sizes | Size every output buffer with a `PSA_*_SIZE`/`PSA_*_LENGTH` macro, never a literal |
| 6 | Present | Show the user: `#include <psa/crypto.h>`, init, attribute setup, operation, abort/destroy |
| 7 | Execute | Write the code following the closest `programs/psa/*.c` example as the starting pattern |
| 8 | Build | `cmake -B build && cmake --build build`; run with `ctest` |
| 9 | Verify | Check `status != PSA_SUCCESS` at every call; on error call the matching `*_abort` then `psa_destroy_key` |

### Step 7 Detail — Example Selection Strategy

Adapt the closest `programs/psa/` example to the user's need:

- Hash (one-shot + multi-part + clone) → `programs/psa/psa_hash.c`
- HMAC multi-part → `programs/psa/hmac_demo.c`
- AES-CBC/CTR multi-part cipher → `programs/psa/crypto_examples.c`
- AEAD (GCM / ChaCha20-Poly1305, short tags) → `programs/psa/aead_demo.c`
- HKDF key ladder + AEAD wrap/unwrap → `programs/psa/key_ladder_demo.c`
- Decoding numeric `psa_status_t`/algorithm/key-type values → `programs/psa/psa_constant_names.c`
- Migrating from legacy `mbedtls_*` → follow `recipes/migration_to_psa.md` (per-algorithm translation tables sourced from `docs/psa-transition.md`)
- Writing/integrating a PSA driver → follow `recipes/psa_driver_development.md` (JSON description + 7-step wrapper integration + driver-only builds)

Copy the example's `PSA_CHECK`/`ASSERT_STATUS` macro pattern; it guarantees errors abort the operation and destroy keys.

---

## Failure Strategies

| Situation | Action |
|---|---|
| Function returns `PSA_ERROR_NOT_SUPPORTED` | Enable the matching `PSA_WANT_*` in `crypto_config.h`, rebuild |
| `PSA_ERROR_NOT_PERMITTED` | Re-check `psa_set_key_usage_flags` and `psa_set_key_algorithm` against the operation |
| `PSA_ERROR_BAD_STATE` | Library not init'd, or multi-part op called out of order / after an error without abort |
| `PSA_ERROR_BUFFER_TOO_SMALL` | Replace literal buffer size with the correct `PSA_*_SIZE`/`PSA_*_LENGTH` macro |
| `PSA_ERROR_INSUFFICIENT_ENTROPY` | RNG seeding failed; provide an entropy source / porting layer (see `docs/architecture/`) |
| `PSA_ERROR_STORAGE_FAILURE`/`DATA_CORRUPT` | Persistent key store issue; check the ITS port (`core/psa_its_file.c`, `core/psa_crypto_storage.c`) |
| Build error about generated files | Run `git submodule update --init`, then `framework/scripts/make_generated_files.py` |
| Unsure of a numeric constant | Run `programs/psa/psa_constant_names status <n>` / `psa_constant_names algorithm <n>` |

---

## References

- Scenario recipes → `recipes/` directory
- Full `psa_*` API reference → `resources/api_reference.md`
- `crypto_config.h` configuration guide → `resources/config_reference.md`
- Consolidated gotchas → `resources/pitfalls.md`
- Example program index → `resources/example_list.md`
- Upstream spec: <https://arm-software.github.io/psa-api/crypto/1.2/>
- Migration from legacy `mbedtls_*` crypto → `docs/psa-transition.md`
- PSA driver development → `docs/psa-driver-example-and-guide.md`, `docs/proposed/psa-driver-developer-guide.md`
