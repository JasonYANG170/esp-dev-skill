# Example Programs in TF-PSA-Crypto

All paths are relative to the repository root (`D:/esp-skill/espressif-repos/TF-PSA-Crypto/`). Sourced from `programs/README.md`, `programs/psa/CMakeLists.txt`, and `programs/test/CMakeLists.txt`. These are the authoritative starting points to copy when implementing a scenario.

## PSA API examples (`programs/psa/`)

| Path | Description |
|---|---|
| `programs/psa/crypto_examples.c` | AES-CBC (no padding, single block) and AES-CBC-PKCS7 / AES-CTR multi-part encrypt+decrypt. Demonstrates `psa_cipher_encrypt_setup`/`generate_iv`/`update`/`finish`/`abort`, attribute setup, `psa_generate_key`, `psa_destroy_key`. |
| `programs/psa/hmac_demo.c` | Multi-part HMAC-SHA256 sign of two messages. Demonstrates `psa_mac_sign_setup`/`update`/`sign_finish`/`abort`, `psa_import_key` of an HMAC key, `mbedtls_platform_zeroize`. |
| `programs/psa/aead_demo.c` | Multi-part AEAD for AES-128-GCM, AES-256-GCM, AES-128-GCM with 8-byte tag (`PSA_ALG_AEAD_WITH_SHORTENED_TAG`), and ChaCha20-Poly1305. Demonstrates `psa_aead_encrypt_setup`/`set_nonce`/`update_ad`/`update`/`finish`, `psa_get_key_attributes`, `PSA_AEAD_TAG_LENGTH`, `PSA_AEAD_ENCRYPT_OUTPUT_MAX_SIZE`. |
| `programs/psa/psa_hash.c` | SHA-256 via one-shot `psa_hash_compute` and multi-part `psa_hash_setup`/`update`/`finish`, plus `psa_hash_clone` + `psa_hash_verify`. Demonstrates `PSA_HASH_LENGTH`. |
| `programs/psa/key_ladder_demo.c` | HKDF-SHA256 key ladder (generate / save / wrap / unwrap). Demonstrates `psa_key_derivation_setup`/`input_bytes`/`input_key`/`output_key`/`abort`, `psa_export_key`, `psa_generate_random`, one-shot `psa_aead_encrypt`/`psa_aead_decrypt` for file wrapping. |
| `programs/psa/psa_constant_names.c` | Utility to print symbolic names from numeric `psa_status_t`/`psa_algorithm_t`/`psa_ecc_family_t`/`psa_key_type_t`/`psa_key_usage_t` values. Run as `psa_constant_names status <n>` etc. |
| `programs/psa/key_ladder_demo.sh` | Shell script driving `key_ladder_demo` end-to-end (generate master, save derived, wrap, unwrap). |
| `programs/psa/psa_hash_demo.sh` | Shell helper for the hash demo. |

## Test / utility programs (`programs/test/`)

| Path | Description |
|---|---|
| `programs/test/benchmark.c` | Benchmark for cryptographic algorithms (links `tf_psa_crypto_test` objects). |
| `programs/test/which_aes.c` | Prints which AES implementation (software / AESCE / AESNI assembly / AESNI intrinsics) is active — diagnostic for driver/acceleration selection (see `recipes/psa_driver_development.md`). Uses `MBEDTLS_DECLARE_PRIVATE_IDENTIFIERS` (sample-only; do not define in application code). |
| `programs/test/tfpsacrypto_dlopen.c` | Demonstrates `dlopen`-ing the shared `libtfpsacrypto` at runtime (built only with `-DUSE_SHARED_TF_PSA_CRYPTO_LIBRARY=On`, non-Windows). |
| `programs/test/cmake_package/` | CMake project consuming TF-PSA-Crypto via `find_package(TF-PSA-Crypto)` and `TF-PSA-Crypto_DIR`. |
| `programs/test/cmake_package_install/` | CMake project consuming an **installed** TF-PSA-Crypto via `CMAKE_PREFIX_PATH`. |
| `programs/test/cmake_subproject/` | CMake project using TF-PSA-Crypto via `add_subdirectory()`. |
| `programs/test/zeroize.c` | (from `framework/tests/programs/`) memory-wipe helper demo. |

## Fuzz harnesses (`programs/fuzz/`)

| Path | Description |
|---|---|
| `programs/fuzz/fuzz_onefile.c` | Generic one-file fuzzer entry. |
| `programs/fuzz/fuzz_pubkey.c` | Fuzz public-key parsing. |
| `programs/fuzz/fuzz_privkey.c` | Fuzz private-key parsing. |
| `programs/fuzz/fuzz_common.c` / `.h` | Shared fuzzer helpers. |

## Config presets (`configs/`)

| Path | Description |
|---|---|
| `configs/crypto-config-ccm-aes-sha256.h` | Minimal config: AES-CCM + SHA-256 only. |
| `configs/crypto-config-symmetric-only.h` | Symmetric-only config (no asymmetric crypto). |
| `configs/ext/README.md` | Notes on the extended (`ext`) config group. |

## Driver description files (`scripts/data_files/driver_jsons/`)

PSA cryptoprocessor driver JSON descriptions — the authoritative templates to copy when writing a transparent/opaque driver (see `recipes/psa_driver_development.md`).

| Path | Description |
|---|---|
| `scripts/data_files/driver_jsons/esp_aes_driver.json` | ESP AES hardware-accelerator transparent driver. Supports `cipher_encrypt`/`cipher_decrypt`/`cipher_*_setup`/`set_iv`/`update`/`finish`/`abort` for `PSA_ALG_AES_CBC` + `PSA_ALG_AES_GCM`. `fallback: false`. Enabled by `ESP_AES_DRIVER_ENABLED`. |
| `scripts/data_files/driver_jsons/esp_sha_driver.json` | ESP SHA hardware-accelerator transparent driver. Supports `hash_setup`/`update`/`finish`/`abort`/`compute` for SHA-1/256/384/512. Enabled by `ESP_SHA_DRIVER_ENABLED`. |
| `scripts/data_files/driver_jsons/mbedtls_test_transparent_driver.json` | Test transparent driver. `import_key` + `export_public_key` (the latter renamed to `mbedtls_test_transparent_export_public_key` via `names`). `fallback: true`. Enabled by `PSA_CRYPTO_DRIVER_TEST`. |
| `scripts/data_files/driver_jsons/mbedtls_test_opaque_driver.json` | Test opaque driver. Demonstrates opaque-only entry points (`export_key`/`copy_key`/`get_builtin_key`). |
| `scripts/data_files/driver_jsons/p256_transparent_driver.json` | p256-m software-accelerator transparent driver (ECDH/ECDSA on secp256r1). Enabled by `MBEDTLS_PSA_P256M_DRIVER_ENABLED`. |
| `scripts/data_files/driverlist.json` | The list of driver JSONs consumed by the build. |
| `scripts/data_files/driver_jsons/driver_transparent_schema.json` / `driver_opaque_schema.json` | JSON schemas for transparent/opaque driver descriptions. |

## Key documentation (`docs/`)

| Path | Description |
|---|---|
| `docs/psa-transition.md` | Migrating from legacy `mbedtls_*` crypto APIs to `psa_*` APIs — the primary usage/migration guide. |
| `docs/1.0-migration-guide.md` | Migrating to TF-PSA-Crypto 1.0 specifically (config changes, removed APIs). |
| `docs/psa-driver-example-and-guide.md` | How to write and integrate a PSA cryptoprocessor driver (transparent/opaque). |
| `docs/driver-only-builds.md` | Building with only specific drivers. |
| `docs/architecture/psa-crypto-implementation-structure.md` | Internal implementation structure. |
| `docs/architecture/psa-keystore-design.md` | Key store design (volatile/persistent, storage). |
| `docs/architecture/mbed-crypto-storage-specification.md` | Persistent-key storage format spec. |
| `docs/architecture/psa-storage-resilience.md` | Storage resilience / power-fail safety. |
| `docs/architecture/psa-thread-safety/psa-thread-safety.md` | Thread-safety guarantees of the PSA core. |
| `docs/proposed/psa-driver-interface.md` | The PSA driver interface spec (work in progress). |
| `docs/proposed/psa-driver-developer-guide.md` | Driver author deliverables. |
| `docs/proposed/psa-driver-integration-guide.md` | Integrating a driver into the build. |

## Key source locations (for reference, do not modify)

| Path | Description |
|---|---|
| `include/psa/crypto.h` | Public API entry header (pulls in the others). |
| `include/psa/crypto_values.h` | All `PSA_ALG_*`/`PSA_KEY_TYPE_*`/`PSA_ERROR_*` macro definitions. |
| `include/psa/crypto_sizes.h` | All `PSA_*_SIZE`/`PSA_*_LENGTH` sizing macros. |
| `include/psa/crypto_struct.h` | `psa_set_key_*`/`psa_get_key_*` inline functions + operation initializers. |
| `include/psa/crypto_types.h` | Core typedefs (`psa_status_t`, `psa_key_id_t`, `psa_algorithm_t`, …). |
| `include/psa/crypto_config.h` | The `PSA_WANT_*` mechanism switches. |
| `include/psa/crypto_extra.h` | `mbedtls_psa_crypto_free` + non-public extensions. |
| `include/tf-psa-crypto/build_info.h` | Version macros + config-load orchestration. |
| `core/psa_crypto.c` | Internal implementation (do not include). |
| `core/psa_crypto_storage.c` / `.h` | Persistent-key storage. |
| `drivers/builtin/`, `drivers/p256-m/`, `drivers/everest/` | Bundled drivers (third-party). |
