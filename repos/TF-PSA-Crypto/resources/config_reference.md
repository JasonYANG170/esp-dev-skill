# TF-PSA-Crypto Configuration Reference

Grounded in `include/psa/crypto_config.h` (default `PSA_WANT_*` set), `configs/README.txt`, `README.md`, and `include/tf-psa-crypto/build_info.h`. Library version 1.0.0; config format version `TF_PSA_CRYPTO_CONFIG_VERSION == 0x01000000`.

## How configuration works

Mechanism availability is **compile-time**, controlled by C preprocessor macros in a single file:

- Default: `include/psa/crypto_config.h`
- Override path: set `TF_PSA_CRYPTO_CONFIG_FILE` (CMake option or preprocessor define) to point at another header
- Optional user additions: `TF_PSA_CRYPTO_USER_CONFIG_FILE`

To **enable** a mechanism, ensure its `PSA_WANT_*` is defined to `1`. To **disable**, comment it out or `#undef` it. The naming convention: a `PSA_ALG_*`/`PSA_KEY_TYPE_*`/`PSA_ECC_*`/`PSA_DH_*` value in `crypto_values.h` corresponds to a `PSA_WANT_*` switch here (replace the `PSA_` prefix with `PSA_WANT_`).

Three ways to edit (from `README.md`):

1. Edit `include/psa/crypto_config.h` directly.
2. `python3 scripts/config.py set/get/unset PSA_WANT_*` (programmatic, `--help` for usage).
3. Use a `configs/` preset via `-DTF_PSA_CRYPTO_CONFIG_FILE=...` (no source-tree edit).

> If you see `PSA_ERROR_NOT_SUPPORTED` at runtime, a needed `PSA_WANT_*` is almost certainly missing.

## Algorithm switches (`PSA_WANT_ALG_*`)

All of the following are defined to `1` in the default `crypto_config.h`:

| Switch | Mechanism |
|---|---|
| `PSA_WANT_ALG_MD5` | MD5 hash |
| `PSA_WANT_ALG_RIPEMD160` | RIPEMD-160 hash |
| `PSA_WANT_ALG_SHA_1` | SHA-1 |
| `PSA_WANT_ALG_SHA_224` / `_SHA_256` / `_SHA_384` / `_SHA_512` | SHA-2 family |
| `PSA_WANT_ALG_SHA3_224` / `_SHA3_256` / `_SHA3_384` / `_SHA3_512` | SHA-3 family |
| `PSA_WANT_ALG_HMAC` | HMAC (over any enabled hash) |
| `PSA_WANT_ALG_CMAC` | AES-CMAC |
| `PSA_WANT_ALG_STREAM_CIPHER` | ChaCha20 stream mode |
| `PSA_WANT_ALG_CTR` | CTR |
| `PSA_WANT_ALG_CFB` | CFB |
| `PSA_WANT_ALG_OFB` | OFB |
| `PSA_WANT_ALG_ECB_NO_PADDING` | ECB |
| `PSA_WANT_ALG_CBC_NO_PADDING` | CBC, no padding |
| `PSA_WANT_ALG_CBC_PKCS7` | CBC + PKCS7 |
| `PSA_WANT_ALG_CCM` / `_CCM_STAR_NO_TAG` | AES-CCM / CCM* |
| `PSA_WANT_ALG_GCM` | AES-GCM |
| `PSA_WANT_ALG_CHACHA20_POLY1305` | ChaCha20-Poly1305 |
| `PSA_WANT_ALG_RSA_PKCS1V15_SIGN` | RSASSA-PKCS1-v1_5 sign |
| `PSA_WANT_ALG_RSA_PKCS1V15_CRYPT` | RSAES-PKCS1-v1_5 encrypt |
| `PSA_WANT_ALG_RSA_PSS` | RSASSA-PSS |
| `PSA_WANT_ALG_RSA_OAEP` | RSAES-OAEP |
| `PSA_WANT_ALG_ECDSA` | ECDSA (random k) |
| `PSA_WANT_ALG_DETERMINISTIC_ECDSA` | Deterministic ECDSA (RFC 6979) |
| `PSA_WANT_ALG_JPAKE` | EC-JPAKE (PAKE) |
| `PSA_WANT_ALG_ECDH` | ECDH key agreement |
| `PSA_WANT_ALG_FFDH` | FFDH (finite-field) key agreement |
| `PSA_WANT_ALG_HKDF` / `_HKDF_EXTRACT` / `_HKDF_EXPAND` | HKDF |
| `PSA_WANT_ALG_PBKDF2_HMAC` | PBKDF2-HMAC |
| `PSA_WANT_ALG_PBKDF2_AES_CMAC_PRF_128` | PBKDF2 with AES-CMAC PRF |
| `PSA_WANT_ALG_TLS12_PRF` | TLS 1.2 PRF |
| `PSA_WANT_ALG_TLS12_PSK_TO_MS` | TLS 1.2 PSK-to-master-secret |
| `PSA_WANT_ALG_TLS12_ECJPAKE_TO_PMS` | TLS 1.2 EC-JPAKE-to-premaster |

> EdDSA (`PSA_ALG_PURE_EDDSA`, `ED25519PH`, `ED448PH`) is exposed via the API but is gated by the underlying EdDSA/key-type availability — verify in your build if needed.

## Key-type switches (`PSA_WANT_KEY_TYPE_*`)

Default-enabled:

| Switch | Key type |
|---|---|
| `PSA_WANT_KEY_TYPE_RAW_DATA` | `PSA_KEY_TYPE_RAW_DATA` |
| `PSA_WANT_KEY_TYPE_HMAC` | HMAC key |
| `PSA_WANT_KEY_TYPE_DERIVE` | Derivation secret |
| `PSA_WANT_KEY_TYPE_PASSWORD` | Password (PBKDF2) |
| `PSA_WANT_KEY_TYPE_PASSWORD_HASH` | Password hash (PBKDF2) |
| `PSA_WANT_KEY_TYPE_AES` / `_ARIA` / `_CAMELLIA` / `_CHACHA20` | Symmetric block/stream keys |
| `PSA_WANT_KEY_TYPE_RSA_PUBLIC_KEY` | RSA public key |
| `PSA_WANT_KEY_TYPE_ECC_PUBLIC_KEY` | ECC public key (all enabled curves) |
| `PSA_WANT_KEY_TYPE_DH_PUBLIC_KEY` | DH/FFDH public key (all enabled groups) |

Key-pair types are split into fine-grained capabilities (a key pair may support some but not all of import/export/generate/derive):

| Switch | Capability |
|---|---|
| `PSA_WANT_KEY_TYPE_RSA_KEY_PAIR_BASIC` | RSA key pair exists |
| `PSA_WANT_KEY_TYPE_RSA_KEY_PAIR_IMPORT` | importable |
| `PSA_WANT_KEY_TYPE_RSA_KEY_PAIR_EXPORT` | exportable |
| `PSA_WANT_KEY_TYPE_RSA_KEY_PAIR_GENERATE` | generatable |
| `PSA_WANT_KEY_TYPE_ECC_KEY_PAIR_BASIC` | ECC key pair exists |
| `PSA_WANT_KEY_TYPE_ECC_KEY_PAIR_IMPORT` | importable |
| `PSA_WANT_KEY_TYPE_ECC_KEY_PAIR_EXPORT` | exportable |
| `PSA_WANT_KEY_TYPE_ECC_KEY_PAIR_GENERATE` | generatable |
| `PSA_WANT_KEY_TYPE_ECC_KEY_PAIR_DERIVE` | derivable (via KDF) |
| `PSA_WANT_KEY_TYPE_DH_KEY_PAIR_BASIC` | DH key pair exists |
| `PSA_WANT_KEY_TYPE_DH_KEY_PAIR_IMPORT` | importable |
| `PSA_WANT_KEY_TYPE_DH_KEY_PAIR_EXPORT` | exportable |
| `PSA_WANT_KEY_TYPE_DH_KEY_PAIR_GENERATE` | generatable |

Example: to generate an RSA key pair you need both `PSA_WANT_KEY_TYPE_RSA_KEY_PAIR_BASIC` and `PSA_WANT_KEY_TYPE_RSA_KEY_PAIR_GENERATE`.

## Curve switches (`PSA_WANT_ECC_*`)

Default-enabled:

| Switch | Curve → family / bits |
|---|---|
| `PSA_WANT_ECC_SECP_R1_256` | secp256r1 (`PSA_ECC_FAMILY_SECP_R1`, 256) |
| `PSA_WANT_ECC_SECP_R1_384` | secp384r1 |
| `PSA_WANT_ECC_SECP_R1_521` | secp521r1 |
| `PSA_WANT_ECC_SECP_K1_256` | secp256k1 (`PSA_ECC_FAMILY_SECP_K1`) |
| `PSA_WANT_ECC_BRAINPOOL_P_R1_256` / `_384` / `_512` | Brainpool (`PSA_ECC_FAMILY_BRAINPOOL_P_R1`) |
| `PSA_WANT_ECC_MONTGOMERY_255` / `_448` | Curve25519 / Curve448 (`PSA_ECC_FAMILY_MONTGOMERY`) |

Internal-only (commented out by default, not public API): `PSA_WANT_ECC_SECP_K1_192`, `PSA_WANT_ECC_SECP_R1_192`.

> For secp256r1, consider also enabling `MBEDTLS_PSA_P256M_DRIVER_ENABLED` (see the in-file comment in `crypto_config.h`) for the p256-m built-in driver.

## DH group switches (`PSA_WANT_DH_*`)

Default-enabled (RFC 7919 FFDHE groups, family `PSA_DH_FAMILY_RFC7919`):

| Switch | Group |
|---|---|
| `PSA_WANT_DH_RFC7919_2048` | ffdhe2048 |
| `PSA_WANT_DH_RFC7919_3072` | ffdhe3072 |
| `PSA_WANT_DH_RFC7919_4096` | ffdhe4096 |
| `PSA_WANT_DH_RFC7919_6144` | ffdhe6144 |
| `PSA_WANT_DH_RFC7919_8192` | ffdhe8192 |

## Built-in driver option

| Option | Purpose |
|---|---|
| `MBEDTLS_PSA_P256M_DRIVER_ENABLED` | Use the p256-m driver for secp256r1 (see `drivers/p256-m/`, comment in `crypto_config.h`) |

## Provided presets (`configs/`)

From `configs/README.txt`:

| File | Focus |
|---|---|
| `configs/crypto-config-ccm-aes-sha256.h` | Minimal: AES-CCM + SHA-256 |
| `configs/crypto-config-symmetric-only.h` | Symmetric-only (no asymmetric) |

Usage:

```bash
# Method 1: replace the default file
cp configs/crypto-config-ccm-aes-sha256.h include/psa/crypto_config.h

# Method 2: external config (keeps source tree clean)
cmake -DTF_PSA_CRYPTO_CONFIG_FILE="configs/crypto-config-ccm-aes-sha256.h" \
      -B build -S .
cmake --build build
```

The second method also works for a custom file kept outside the TF-PSA-Crypto tree.

## Config version symbol

```c
#define TF_PSA_CRYPTO_CONFIG_VERSION 0x01000000   /* in crypto_config.h */
```

This enables compatibility handling. It must satisfy `(TF_PSA_CRYPTO_CONFIG_VERSION >= 0x01000000)` and `<= TF_PSA_CRYPTO_VERSION_NUMBER` (checked in `include/tf-psa-crypto/build_info.h`).

## Build-time CMake options that affect config

| CMake option | Effect |
|---|---|
| `-DTF_PSA_CRYPTO_CONFIG_FILE=<path>` | Use this file as the config (replaces default) |
| `-DTF_PSA_CRYPTO_USER_CONFIG_FILE=<path>` | Append extra user defines |
| `-DTF_PSA_CRYPTO_INCLUDE_AFTER_RAW_CONFIG=<path>` | Include a file after the raw config is read |
| `-DENABLE_TESTING=Off` | Skip test build (no Python needed) |

After all config files are read, `build_info.h` defines `TF_PSA_CRYPTO_CONFIG_FILES_READ` then `TF_PSA_CRYPTO_CONFIG_IS_FINALIZED` — at that point dependencies are resolved and `PSA_*_SIZE` macros are safe to query.

## Programmatic editing (`scripts/config.py`)

```bash
python3 scripts/config.py --help
python3 scripts/config.py set   PSA_WANT_ALG_SHA3_256
python3 scripts/config.py get   PSA_WANT_KEY_TYPE_AES
python3 scripts/config.py unset PSA_WANT_ALG_MD5
```

## Common patterns

| You want to … | Enable |
|---|---|
| Compute SHA-256 hashes | `PSA_WANT_ALG_SHA_256` |
| Do AES-GCM AEAD | `PSA_WANT_KEY_TYPE_AES` + `PSA_WANT_ALG_GCM` |
| HMAC-SHA256 | `PSA_WANT_ALG_HMAC` + `PSA_WANT_ALG_SHA_256` |
| Generate an RSA-2048 key + PSS sign | `PSA_WANT_KEY_TYPE_RSA_KEY_PAIR_BASIC` + `_GENERATE` + `PSA_WANT_ALG_RSA_PSS` + a hash |
| ECDSA on secp256r1 | `PSA_WANT_KEY_TYPE_ECC_KEY_PAIR_BASIC` + `_GENERATE` + `PSA_WANT_ALG_ECDSA` + `PSA_WANT_ECC_SECP_R1_256` + hash |
| ECDH on Curve25519 | `PSA_WANT_KEY_TYPE_ECC_KEY_PAIR_BASIC` + `_GENERATE` + `PSA_WANT_ALG_ECDH` + `PSA_WANT_ECC_MONTGOMERY_255` |
| HKDF key ladder | `PSA_WANT_ALG_HKDF` + `PSA_WANT_KEY_TYPE_DERIVE` + hash |
| PBKDF2 from password | `PSA_WANT_ALG_PBKDF2_HMAC` + `PSA_WANT_KEY_TYPE_PASSWORD` + hash |
