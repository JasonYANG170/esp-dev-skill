# PSA Crypto API Quick Reference (TF-PSA-Crypto 1.0.0)

Grounded in `include/psa/crypto.h`, `crypto_struct.h`, `crypto_values.h`, `crypto_sizes.h`.
API version: PSA Crypto 1.2 (`PSA_CRYPTO_API_VERSION_MAJOR == 1`, `_MINOR == 2`).

All functions return `psa_status_t` (`PSA_SUCCESS == 0`; errors negative). Include with `#include <psa/crypto.h>`.

---

## 1. Library lifecycle

```c
/* include/psa/crypto.h */
psa_status_t psa_crypto_init(void);

/* include/psa/crypto_extra.h */
void mbedtls_psa_crypto_free(void);   /* free all PSA resources, destroy volatile keys */
```

## 2. Key attributes

Object type `psa_key_attributes_t`; initializer `PSA_KEY_ATTRIBUTES_INIT`. Setters (in `crypto_struct.h`):

```c
void psa_set_key_id(psa_key_attributes_t *attributes, mbedtls_svc_key_id_t key);
void psa_set_key_lifetime(psa_key_attributes_t *attributes, psa_key_lifetime_t lifetime);
void psa_set_key_usage_flags(psa_key_attributes_t *attributes, psa_key_usage_t usage_flags);
void psa_set_key_algorithm(psa_key_attributes_t *attributes, psa_algorithm_t alg);
void psa_set_key_type(psa_key_attributes_t *attributes, psa_key_type_t type);
void psa_set_key_bits(psa_key_attributes_t *attributes, size_t bits);
```

Getters (read a key's attributes via a handle):

```c
mbedtls_svc_key_id_t psa_get_key_id(const psa_key_attributes_t *attributes);
psa_key_lifetime_t   psa_get_key_lifetime(const psa_key_attributes_t *attributes);
psa_key_usage_t      psa_get_key_usage_flags(const psa_key_attributes_t *attributes);
psa_algorithm_t      psa_get_key_algorithm(const psa_key_attributes_t *attributes);
psa_key_type_t       psa_get_key_type(const psa_key_attributes_t *attributes);
size_t               psa_get_key_bits(const psa_key_attributes_t *attributes);
```

```c
psa_status_t psa_get_key_attributes(mbedtls_svc_key_id_t key,
                                    psa_key_attributes_t *attributes);   /* fill from key */
void psa_reset_key_attributes(psa_key_attributes_t *attributes);
static psa_key_attributes_t psa_key_attributes_init(void);
```

## 3. Key management

```c
psa_status_t psa_import_key(const psa_key_attributes_t *attributes,
                            const uint8_t *data, size_t data_length,
                            psa_key_id_t *key);
psa_status_t psa_generate_key(const psa_key_attributes_t *attributes,
                              psa_key_id_t *key);
psa_status_t psa_generate_key_custom(const psa_key_attributes_t *attributes,
                                     const psa_custom_key_parameters_t *custom,
                                     const uint8_t *custom_data,
                                     size_t custom_data_length,
                                     mbedtls_svc_key_id_t *key);
/* Note: psa_generate_key_custom takes a psa_custom_key_parameters_t +
   optional custom_data; see the docstring in include/psa/crypto.h. */

psa_status_t psa_destroy_key(mbedtls_svc_key_id_t key);
psa_status_t psa_purge_key(mbedtls_svc_key_id_t key);   /* drop cached persistent key material */
psa_status_t psa_copy_key(mbedtls_svc_key_id_t source_key,
                          const psa_key_attributes_t *attributes,
                          psa_key_id_t *target_key);

psa_status_t psa_export_key(mbedtls_svc_key_id_t key,
                            uint8_t *data, size_t data_size, size_t *data_length);
psa_status_t psa_export_public_key(mbedtls_svc_key_id_t key,
                                   uint8_t *data, size_t data_size, size_t *data_length);
```

## 4. Random

```c
psa_status_t psa_generate_random(uint8_t *output, size_t output_size,
                                 size_t *output_length);
```

## 5. Hash

One-shot:

```c
psa_status_t psa_hash_compute(psa_algorithm_t alg,
                              const uint8_t *input, size_t input_length,
                              uint8_t *hash, size_t hash_size, size_t *hash_length);
psa_status_t psa_hash_compare(psa_algorithm_t alg,
                              const uint8_t *input, size_t input_length,
                              const uint8_t *hash, size_t hash_length);
```

Multi-part (type `psa_hash_operation_t`, init `PSA_HASH_OPERATION_INIT`):

```c
psa_status_t psa_hash_setup(psa_hash_operation_t *operation, psa_algorithm_t alg);
psa_status_t psa_hash_update(psa_hash_operation_t *operation,
                             const uint8_t *input, size_t input_length);
psa_status_t psa_hash_finish(psa_hash_operation_t *operation,
                             uint8_t *hash, size_t hash_size, size_t *hash_length);
psa_status_t psa_hash_verify(psa_hash_operation_t *operation,
                             const uint8_t *hash, size_t hash_length);
psa_status_t psa_hash_abort(psa_hash_operation_t *operation);
psa_status_t psa_hash_clone(const psa_hash_operation_t *source_operation,
                            psa_hash_operation_t *target_operation);
```

## 6. MAC

One-shot:

```c
psa_status_t psa_mac_compute(mbedtls_svc_key_id_t key, psa_algorithm_t alg,
                             const uint8_t *input, size_t input_length,
                             uint8_t *mac, size_t mac_size, size_t *mac_length);
psa_status_t psa_mac_verify(mbedtls_svc_key_id_t key, psa_algorithm_t alg,
                            const uint8_t *input, size_t input_length,
                            const uint8_t *mac, size_t mac_length);
```

Multi-part (type `psa_mac_operation_t`, init `PSA_MAC_OPERATION_INIT`):

```c
psa_status_t psa_mac_sign_setup(psa_mac_operation_t *operation,
                                mbedtls_svc_key_id_t key, psa_algorithm_t alg);
psa_status_t psa_mac_verify_setup(psa_mac_operation_t *operation,
                                  mbedtls_svc_key_id_t key, psa_algorithm_t alg);
psa_status_t psa_mac_update(psa_mac_operation_t *operation,
                            const uint8_t *input, size_t input_length);
psa_status_t psa_mac_sign_finish(psa_mac_operation_t *operation,
                                 uint8_t *mac, size_t mac_size, size_t *mac_length);
psa_status_t psa_mac_verify_finish(psa_mac_operation_t *operation,
                                   const uint8_t *mac, size_t mac_length);
psa_status_t psa_mac_abort(psa_mac_operation_t *operation);
```

## 7. Symmetric cipher

One-shot:

```c
psa_status_t psa_cipher_encrypt(mbedtls_svc_key_id_t key, psa_algorithm_t alg,
                                const uint8_t *input, size_t input_length,
                                uint8_t *output, size_t output_size,
                                size_t *output_length);
psa_status_t psa_cipher_decrypt(mbedtls_svc_key_id_t key, psa_algorithm_t alg,
                                const uint8_t *input, size_t input_length,
                                uint8_t *output, size_t output_size,
                                size_t *output_length);
```

Multi-part (type `psa_cipher_operation_t`, init `PSA_CIPHER_OPERATION_INIT`):

```c
psa_status_t psa_cipher_encrypt_setup(psa_cipher_operation_t *operation,
                                      mbedtls_svc_key_id_t key, psa_algorithm_t alg);
psa_status_t psa_cipher_decrypt_setup(psa_cipher_operation_t *operation,
                                      mbedtls_svc_key_id_t key, psa_algorithm_t alg);
psa_status_t psa_cipher_generate_iv(psa_cipher_operation_t *operation,
                                    uint8_t *iv, size_t iv_size, size_t *iv_length);
psa_status_t psa_cipher_set_iv(psa_cipher_operation_t *operation,
                               const uint8_t *iv, size_t iv_length);
psa_status_t psa_cipher_update(psa_cipher_operation_t *operation,
                               const uint8_t *input, size_t input_length,
                               uint8_t *output, size_t output_size,
                               size_t *output_length);
psa_status_t psa_cipher_finish(psa_cipher_operation_t *operation,
                               uint8_t *output, size_t output_size,
                               size_t *output_length);
psa_status_t psa_cipher_abort(psa_cipher_operation_t *operation);
```

## 8. AEAD

One-shot:

```c
psa_status_t psa_aead_encrypt(mbedtls_svc_key_id_t key, psa_algorithm_t alg,
                              const uint8_t *nonce, size_t nonce_length,
                              const uint8_t *additional_data, size_t additional_data_length,
                              const uint8_t *plaintext, size_t plaintext_length,
                              uint8_t *ciphertext, size_t ciphertext_size,
                              size_t *ciphertext_length);
psa_status_t psa_aead_decrypt(mbedtls_svc_key_id_t key, psa_algorithm_t alg,
                              const uint8_t *nonce, size_t nonce_length,
                              const uint8_t *additional_data, size_t additional_data_length,
                              const uint8_t *ciphertext, size_t ciphertext_length,
                              uint8_t *plaintext, size_t plaintext_size,
                              size_t *plaintext_length);
```

Multi-part (type `psa_aead_operation_t`, init `PSA_AEAD_OPERATION_INIT`):

```c
psa_status_t psa_aead_encrypt_setup(psa_aead_operation_t *operation,
                                    mbedtls_svc_key_id_t key, psa_algorithm_t alg);
psa_status_t psa_aead_decrypt_setup(psa_aead_operation_t *operation,
                                    mbedtls_svc_key_id_t key, psa_algorithm_t alg);
psa_status_t psa_aead_generate_nonce(psa_aead_operation_t *operation,
                                     uint8_t *nonce, size_t nonce_size, size_t *nonce_length);
psa_status_t psa_aead_set_nonce(psa_aead_operation_t *operation,
                                const uint8_t *nonce, size_t nonce_length);
psa_status_t psa_aead_set_lengths(psa_aead_operation_t *operation,
                                  size_t ad_length, size_t plaintext_length);
psa_status_t psa_aead_update_ad(psa_aead_operation_t *operation,
                                const uint8_t *input, size_t input_length);
psa_status_t psa_aead_update(psa_aead_operation_t *operation,
                             const uint8_t *input, size_t input_length,
                             uint8_t *output, size_t output_size, size_t *output_length);
psa_status_t psa_aead_finish(psa_aead_operation_t *operation,
                             uint8_t *ciphertext, size_t ciphertext_size,
                             size_t *ciphertext_length,
                             uint8_t *tag, size_t tag_size, size_t *tag_length);
psa_status_t psa_aead_verify(psa_aead_operation_t *operation,
                             uint8_t *plaintext, size_t plaintext_size,
                             size_t *plaintext_length,
                             const uint8_t *tag, size_t tag_length);
psa_status_t psa_aead_abort(psa_aead_operation_t *operation);
```

## 9. Asymmetric signatures

Message-level (hash-and-sign internally):

```c
psa_status_t psa_sign_message(mbedtls_svc_key_id_t key, psa_algorithm_t alg,
                              const uint8_t *input, size_t input_length,
                              uint8_t *signature, size_t signature_size,
                              size_t *signature_length);
psa_status_t psa_verify_message(mbedtls_svc_key_id_t key, psa_algorithm_t alg,
                                const uint8_t *input, size_t input_length,
                                const uint8_t *signature, size_t signature_length);
```

Hash-level:

```c
psa_status_t psa_sign_hash(mbedtls_svc_key_id_t key, psa_algorithm_t alg,
                           const uint8_t *hash, size_t hash_length,
                           uint8_t *signature, size_t signature_size,
                           size_t *signature_length);
psa_status_t psa_verify_hash(mbedtls_svc_key_id_t key, psa_algorithm_t alg,
                             const uint8_t *hash, size_t hash_length,
                             const uint8_t *signature, size_t signature_length);
```

## 10. Key agreement & derivation

Raw key agreement:

```c
psa_status_t psa_raw_key_agreement(psa_algorithm_t alg,
                                   mbedtls_svc_key_id_t private_key,
                                   const uint8_t *peer_key, size_t peer_key_length,
                                   uint8_t *output, size_t output_size,
                                   size_t *output_length);
```

Key derivation (type `psa_key_derivation_operation_t`, init `PSA_KEY_DERIVATION_OPERATION_INIT`):

```c
psa_status_t psa_key_derivation_setup(psa_key_derivation_operation_t *operation,
                                      psa_algorithm_t alg);
psa_status_t psa_key_derivation_get_capacity(const psa_key_derivation_operation_t *operation,
                                             size_t *capacity);
psa_status_t psa_key_derivation_set_capacity(psa_key_derivation_operation_t *operation,
                                             size_t capacity);
psa_status_t psa_key_derivation_input_bytes(psa_key_derivation_operation_t *operation,
                                            psa_key_derivation_step_t step,
                                            const uint8_t *data, size_t data_length);
psa_status_t psa_key_derivation_input_integer(psa_key_derivation_operation_t *operation,
                                              psa_key_derivation_step_t step,
                                              uint64_t value);
psa_status_t psa_key_derivation_input_key(psa_key_derivation_operation_t *operation,
                                          psa_key_derivation_step_t step,
                                          mbedtls_svc_key_id_t key);
psa_status_t psa_key_derivation_key_agreement(psa_key_derivation_operation_t *operation,
                                              psa_key_derivation_step_t step,
                                              mbedtls_svc_key_id_t private_key,
                                              const uint8_t *peer_key, size_t peer_key_length);
psa_status_t psa_key_derivation_output_bytes(psa_key_derivation_operation_t *operation,
                                             uint8_t *output, size_t output_length);
psa_status_t psa_key_derivation_output_key(const psa_key_attributes_t *attributes,
                                           psa_key_derivation_operation_t *operation,
                                           psa_key_id_t *key);
psa_status_t psa_key_derivation_output_key_custom(const psa_key_attributes_t *attributes,
                                                  psa_key_derivation_operation_t *operation,
                                                  const psa_custom_key_parameters_t *custom,
                                                  const uint8_t *custom_data,
                                                  size_t custom_data_length,
                                                  mbedtls_svc_key_id_t *key);
psa_status_t psa_key_derivation_verify_bytes(psa_key_derivation_operation_t *operation,
                                             const uint8_t *expected_output, size_t output_length);
psa_status_t psa_key_derivation_verify_key(psa_key_derivation_operation_t *operation,
                                           mbedtls_svc_key_id_t *key);
psa_status_t psa_key_derivation_abort(psa_key_derivation_operation_t *operation);
```

## 11. Commonly used macros

From `include/psa/crypto_values.h`:

| Category | Macros |
|---|---|
| Status | `PSA_SUCCESS` (0), `PSA_ERROR_NOT_SUPPORTED`, `_NOT_PERMITTED`, `_INVALID_ARGUMENT`, `_INVALID_HANDLE`, `_BUFFER_TOO_SMALL`, `_BAD_STATE`, `_INSUFFICIENT_MEMORY`, `_INSUFFICIENT_DATA`, `_INSUFFICIENT_ENTROPY`, `_INVALID_SIGNATURE`, `_INVALID_PADDING`, `_COMMUNICATION_FAILURE`, `_HARDWARE_FAILURE`, `_CORRUPTION_DETECTED`, `_STORAGE_FAILURE`, `_DATA_CORRUPT`, `_DATA_INVALID` |
| Key types | `PSA_KEY_TYPE_AES/ARIA/CAMELLIA/CHACHA20`, `PSA_KEY_TYPE_HMAC`, `PSA_KEY_TYPE_DERIVE`, `PSA_KEY_TYPE_PASSWORD/PASSWORD_HASH/PEPPER`, `PSA_KEY_TYPE_RSA_KEY_PAIR/_PUBLIC_KEY`, `PSA_KEY_TYPE_ECC_KEY_PAIR(family)` / `_ECC_PUBLIC_KEY(family)`, `PSA_KEY_TYPE_DH_KEY_PAIR(group)` / `_DH_PUBLIC_KEY(group)`, `PSA_KEY_TYPE_RAW_DATA` |
| ECC family | `PSA_ECC_FAMILY_SECP_R1/SECP_K1/SECP_R2/SECT_K1/SECT_R1/SECT_R2/BRAINPOOL_P_R1/MONTGOMERY/TWISTED_EDWARDS` |
| Lifetime / id | `PSA_KEY_LIFETIME_VOLATILE` (0), `PSA_KEY_LIFETIME_PERSISTENT` (0x1), `PSA_KEY_ID_NULL` (0), `PSA_KEY_ID_USER_MIN` (0x1), `PSA_KEY_ID_USER_MAX` (0x3fffffff) |
| Usage | `PSA_KEY_USAGE_EXPORT/COPY/ENCRYPT/DECRYPT/SIGN_MESSAGE/VERIFY_MESSAGE/SIGN_HASH/VERIFY_HASH/DERIVE/VERIFY_DERIVATION` |
| Hash alg | `PSA_ALG_MD5/SHA_1/SHA_224/SHA_256/SHA_384/SHA_512/SHA3_256/SHA3_512/RIPEMD160` |
| MAC alg | `PSA_ALG_HMAC(hash)`, `PSA_ALG_CMAC` |
| Cipher alg | `PSA_ALG_STREAM_CIPHER/CTR/CFB/OFB/ECB_NO_PADDING/CBC_NO_PADDING/CBC_PKCS7` |
| AEAD alg | `PSA_ALG_GCM/CCM/CHACHA20_POLY1305/CCM_STAR_NO_TAG`, `PSA_ALG_AEAD_WITH_SHORTENED_TAG(aead,tag_len)`, `PSA_ALG_AEAD_WITH_DEFAULT_LENGTH_TAG(aead)` |
| Sign alg | `PSA_ALG_RSA_PKCS1V15_SIGN(hash)`, `PSA_ALG_RSA_PSS(hash)`, `PSA_ALG_ECDSA(hash)`, `PSA_ALG_DETERMINISTIC_ECDSA(hash)`, `PSA_ALG_PURE_EDDSA`, `PSA_ALG_ED25519PH`, `PSA_ALG_ED448PH`, `PSA_ALG_RSA_PKCS1V15_CRYPT`, `PSA_ALG_RSA_OAEP(hash)` |
| KA alg | `PSA_ALG_ECDH`, `PSA_ALG_FFDH` |
| KDF alg | `PSA_ALG_HKDF(hash)`, `PSA_ALG_HKDF_EXPAND(hash)`, `PSA_ALG_PBKDF2_HMAC(hash)`, `PSA_ALG_PBKDF2_AES_CMAC_PRF_128`, `PSA_ALG_TLS12_PRF(hash)`, `PSA_ALG_TLS12_PSK_TO_MS(hash)` |
| Deriv step | `PSA_KEY_DERIVATION_INPUT_SECRET/OTHER_SECRET/LABEL/SALT/INFO/SEED` |
| Predicates | `PSA_ALG_IS_HASH/MAC/AEAD/AEAD_ON_BLOCK_CIPHER/KEY_AGREEMENT/KEY_DERIVATION/RAW_KEY_AGREEMENT/SIGN_HASH/SIGN_MESSAGE` |

From `include/psa/crypto_sizes.h`:

| Macro | Meaning |
|---|---|
| `PSA_HASH_LENGTH(alg)` | Hash output length |
| `PSA_HASH_MAX_SIZE` | Max hash output |
| `PSA_MAC_LENGTH(type, bits, alg)` | MAC length |
| `PSA_MAC_MAX_SIZE` | `== PSA_HASH_MAX_SIZE` |
| `PSA_AEAD_TAG_LENGTH(type, bits, alg)` | AEAD tag length |
| `PSA_AEAD_TAG_MAX_SIZE` | 16 |
| `PSA_AEAD_ENCRYPT_OUTPUT_SIZE(type, alg, pt_len)` | Encrypt output size |
| `PSA_AEAD_ENCRYPT_OUTPUT_MAX_SIZE(pt_len)` | Encrypt output upper bound |
| `PSA_SIGN_OUTPUT_SIZE(type, bits, alg)` | Signature upper bound |
| `PSA_EXPORT_KEY_OUTPUT_SIZE(type, bits)` | Export buffer size |
| `PSA_EXPORT_KEY_PAIR_MAX_SIZE` | Any key pair export upper bound |
| `PSA_EXPORT_PUBLIC_KEY_MAX_SIZE` | Any public key export upper bound |
| `PSA_RAW_KEY_AGREEMENT_OUTPUT_MAX_SIZE` | Raw KA output upper bound |
| `PSA_BLOCK_CIPHER_BLOCK_LENGTH(type)` | Block size (AES => 16) |
| `PSA_BITS_TO_BYTES(bits)` / `PSA_BYTES_TO_BITS(bytes)` | Unit conversion |

Operation initializers (from `crypto_struct.h`): `PSA_KEY_ATTRIBUTES_INIT`, `PSA_HASH_OPERATION_INIT`, `PSA_MAC_OPERATION_INIT`, `PSA_CIPHER_OPERATION_INIT`, `PSA_AEAD_OPERATION_INIT`, `PSA_KEY_DERIVATION_OPERATION_INIT`. Derivation capacity constant: `PSA_KEY_DERIVATION_UNLIMITED_CAPACITY == ((size_t)-1)`.

## 12. Version (from `include/tf-psa-crypto/build_info.h`)

```
TF_PSA_CRYPTO_VERSION_MAJOR 1, MINOR 0, PATCH 0
TF_PSA_CRYPTO_VERSION_STRING "1.0.0"
PSA_CRYPTO_API_VERSION_MAJOR 1, MINOR 2
```

## 13. Migration bridge functions (legacy mbedtls_* → PSA)

Helpers in `<mbedtls/psa_util.h>` and `<mbedtls/pk.h>` for code transitioning from legacy Mbed TLS APIs. See `recipes/migration_to_psa.md`.

### RNG callback bridging (`<mbedtls/psa_util.h>`)

For third-party code that still requires an `f_rng, p_rng` pair:

```c
int mbedtls_psa_get_random(void *p_rng, unsigned char *output, size_t output_size);
#define MBEDTLS_PSA_RANDOM_STATE    NULL   /* pass as p_rng */
```

### Hash type conversion (`<mbedtls/psa_util.h>`)

```c
static inline psa_algorithm_t mbedtls_md_psa_alg_from_type(mbedtls_md_type_t md_type);  /* legacy md_type -> PSA_ALG_* */
static inline mbedtls_md_type_t mbedtls_md_type_from_psa_alg(psa_algorithm_t psa_alg);  /* reverse */
```

### ECDSA signature format conversion (`<mbedtls/psa_util.h>`)

PSA uses raw fixed-length ECDSA signatures; legacy used ASN.1 DER.

```c
int mbedtls_ecdsa_raw_to_der(size_t bits, const unsigned char *raw, size_t raw_len,
                             unsigned char *der, size_t der_size, size_t *der_len);
int mbedtls_ecdsa_der_to_raw(size_t bits, const unsigned char *der, size_t der_len,
                             unsigned char *raw, size_t raw_size, size_t *raw_len);
```

### PK ↔ PSA key bridge (`<mbedtls/pk.h>`)

```c
/* Wrap a PSA key into a PK context (renamed from mbedtls_pk_setup_opaque in 3.x) */
int mbedtls_pk_wrap_psa(mbedtls_pk_context *ctx, const mbedtls_svc_key_id_t key);

/* Check if a PK key can perform alg+usage (replaces mbedtls_pk_can_do_ext) */
int mbedtls_pk_can_do_psa(const mbedtls_pk_context *pk, psa_algorithm_t alg, psa_key_usage_t usage);

/* Derive PSA attributes from a parsed PK key (usage: SIGN_HASH/SIGN_MESSAGE/DECRYPT/DERIVE/VERIFY_*/ENCRYPT) */
int mbedtls_pk_get_psa_attributes(const mbedtls_pk_context *pk,
                                  psa_key_usage_t usage,
                                  psa_key_attributes_t *attributes);

/* Import a PK key into the PSA key store (equivalent to psa_import_key with PK material) */
int mbedtls_pk_import_into_psa(const mbedtls_pk_context *pk,
                               const psa_key_attributes_t *attributes,
                               mbedtls_svc_key_id_t *key_id);

/* Copy a PSA key (exportable) into a PK context */
int mbedtls_pk_copy_from_psa(mbedtls_svc_key_id_t key, mbedtls_pk_context *pk);
/* Copy only the public part of a PSA key into a PK context */
int mbedtls_pk_copy_public_from_psa(mbedtls_svc_key_id_t key, mbedtls_pk_context *pk);
```

## 14. PSA cryptoprocessor driver interface (for driver authors)

Driver dispatch is invoked internally for each PSA operation when a matching driver is compiled in. Application code never calls these directly. See `recipes/psa_driver_development.md`, `docs/psa-driver-example-and-guide.md`, and `docs/proposed/psa-driver-developer-guide.md`.

### Driver types and dispatch

- **Transparent** driver: operates on cleartext keys; dispatched by algorithm+key-type+size match; used for **accelerators**. Entry-point naming: `<prefix>_transparent_<entry_point>`.
- **Opaque** driver: keys live in a protected location; dispatched by key **location** (`psa_key_lifetime_t` location field); used for **secure elements/HSMs**. Entry-point naming: `<prefix>_opaque_<entry_point>`.

### Entry points with auto-generated dispatch wrappers

Only these entry points support JSON-driven auto-generation of the dispatch wrapper (from `docs/psa-driver-example-and-guide.md`):

| Transparent | Opaque |
|---|---|
| `import_key`, `export_public_key` | `import_key`, `export_public_key`, `export_key`, `copy_key`, `get_builtin_key` |

All other entry points (`cipher_*`, `hash_*`, `aead_*`, `sign_hash`, `verify_hash`, `key_agreement`, `generate_key`, etc.) require **manual** editing of `psa_crypto_driver_wrappers.h.jinja` / `psa_crypto_driver_wrappers_no_static.c.jinja`.

### Driver entry-point return codes

```c
/* Expected return values from a driver entry point (psa_status_t): */
PSA_SUCCESS;               /* operation completed by this driver */
PSA_ERROR_NOT_SUPPORTED;   /* input valid but this driver doesn't handle it → library tries next driver / built-in (transparent only) */
PSA_ERROR_*;               /* any other error → propagated immediately, no fallback */
```

### Driver-only build configuration triple

For a mechanism provided only by a driver (no built-in), all three must be set together in `crypto_config.h` / build flags:

```c
#define PSA_WANT_<MECHANISM>              1   /* mechanism available in PSA API */
#define MBEDTLS_PSA_ACCEL_<MECHANISM>     1   /* declares an accelerator exists */
/* #undef MBEDTLS_<XXX>_C */                  /* built-in implementation removed */
```

Where `<MECHANISM>` is one of `ALG_SHA_256`, `KEY_TYPE_AES`, `ALG_GCM`, `ECC_SECP_R1_256`, `KEY_TYPE_ECC_KEY_PAIR_BASIC`, `ALG_FFDH`, etc. (see `docs/driver-only-builds.md` for the full supported family list and per-family constraints).
