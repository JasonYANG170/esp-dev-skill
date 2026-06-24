# 非对称签名与密钥协商（ECDSA / RSA / ECDH）

> **适用摘要**: 使用 PSA Crypto API 进行非对称签名/验签（ECDSA、确定性 ECDSA、RSA-PSS/PKCS1v15）和密钥协商（ECDH/FFDH 原始协商），以及用 `psa_export_public_key` 导出公钥。

## 触发意图

- "ECDSA 签名 / 验签"
- "psa_sign_message / psa_verify_message"
- "psa_sign_hash / psa_verify_hash"
- "RSA-PSS / RSA-PKCS1v15"
- "ECDH 密钥协商 / psa_raw_key_agreement"
- "导出公钥 psa_export_public_key"
- "Ed25519 / Ed25519ph"

## 前置条件

| 条件 | 要求 |
|---|---|
| 库状态 | 已成功调用 `psa_crypto_init()` |
| 头文件 | `#include <psa/crypto.h>` |
| 配置项 | `PSA_WANT_ALG_ECDSA`/`PSA_WANT_ALG_DETERMINISTIC_ECDSA` + `PSA_WANT_KEY_TYPE_ECC_KEY_PAIR_*` + `PSA_WANT_ECC_SECP_R1_256`（等曲线）；RSA 需 `PSA_WANT_ALG_RSA_PSS`/`RSA_PKCS1V15_SIGN` + `PSA_WANT_KEY_TYPE_RSA_KEY_PAIR_*`；ECDH 需 `PSA_WANT_ALG_ECDH` |
| 参考示例 | `programs/psa/key_ladder_demo.c`（含 `psa_export_key`/`psa_get_key_attributes`） |

## 分步说明

### 1. 算法

签名算法（`include/psa/crypto_values.h`）：

| 算法宏 | 说明 |
|---|---|
| `PSA_ALG_ECDSA(hash)` | ECDSA，随机 k，输出为原始 r‖s |
| `PSA_ALG_DETERMINISTIC_ECDSA(hash)` | RFC 6979 确定性 ECDSA |
| `PSA_ALG_RSA_PKCS1V15_SIGN(hash)` | RSA PKCS#1 v1.5 签名 |
| `PSA_ALG_RSA_PSS(hash)` | RSASSA-PSS |
| `PSA_ALG_PURE_EDDSA` | 纯 EdDSA（不预哈希） |
| `PSA_ALG_ED25519PH` / `PSA_ALG_ED448PH` | 预哈希 EdDSA |

`hash` 是一个哈希算法（如 `PSA_ALG_SHA_256`）。

### 2. 两类签名 API

| 函数 | 输入 | 说明 |
|---|---|---|
| `psa_sign_message` / `psa_verify_message` | 原始消息 | 内部先做 hash-and-sign；需要 `PSA_KEY_USAGE_SIGN_MESSAGE`/`VERIFY_MESSAGE` |
| `psa_sign_hash` / `psa_verify_hash` | 已计算好的 hash | 需要先用 `psa_hash_compute` 算摘要；需要 `PSA_KEY_USAGE_SIGN_HASH`/`VERIFY_HASH` |

签名：

```c
psa_status_t psa_sign_message(mbedtls_svc_key_id_t key, psa_algorithm_t alg,
                              const uint8_t *input, size_t input_length,
                              uint8_t *signature, size_t signature_size,
                              size_t *signature_length);
psa_status_t psa_sign_hash(mbedtls_svc_key_id_t key, psa_algorithm_t alg,
                           const uint8_t *hash, size_t hash_length,
                           uint8_t *signature, size_t signature_size,
                           size_t *signature_length);
```

`psa_verify_message` / `psa_verify_hash` 参数相同，但把 `signature` 换成输入且无 `signature_length` 输出。验签失败返回 `PSA_ERROR_INVALID_SIGNATURE`。

### 3. ECDSA 签名/验签完整流程

```c
/* 1. 生成 secp256r1 密钥对 */
psa_key_attributes_t attr = PSA_KEY_ATTRIBUTES_INIT;
psa_set_key_type(&attr,
                 PSA_KEY_TYPE_ECC_KEY_PAIR(PSA_ECC_FAMILY_SECP_R1));
psa_set_key_bits(&attr, 256);
psa_set_key_usage_flags(&attr,
                        PSA_KEY_USAGE_SIGN_HASH | PSA_KEY_USAGE_VERIFY_HASH);
psa_set_key_algorithm(&attr, PSA_ALG_ECDSA(PSA_ALG_SHA_256));
psa_key_id_t key;
psa_generate_key(&attr, &key);

/* 2. 导出公钥分发给验签端 */
uint8_t pub[PSA_EXPORT_PUBLIC_KEY_MAX_SIZE];
size_t pub_len;
psa_export_public_key(key, pub, sizeof(pub), &pub_len);

/* 验签端把 pub 导入为公钥对象 */
psa_key_attributes_t vattr = PSA_KEY_ATTRIBUTES_INIT;
psa_set_key_type(&vattr,
                 PSA_KEY_TYPE_ECC_PUBLIC_KEY(PSA_ECC_FAMILY_SECP_R1));
psa_set_key_bits(&vattr, 256);
psa_set_key_usage_flags(&vattr, PSA_KEY_USAGE_VERIFY_HASH);
psa_set_key_algorithm(&vattr, PSA_ALG_ECDSA(PSA_ALG_SHA_256));
psa_key_id_t pub_key;
psa_import_key(&vattr, pub, pub_len, &pub_key);

/* 3. 签名 (hash-and-sign 模式，传原始消息) */
uint8_t sig[PSA_SIGNATURE_MAX_SIZE];   /* 或 PSA_SIGN_OUTPUT_SIZE(type,bits,alg) */
size_t sig_len;
psa_sign_message(key, PSA_ALG_ECDSA(PSA_ALG_SHA_256),
                 msg, msg_len, sig, sizeof(sig), &sig_len);

/* 4. 验签 */
psa_status_t ok = psa_verify_message(pub_key, PSA_ALG_ECDSA(PSA_ALG_SHA_256),
                                     msg, msg_len, sig, sig_len);
/* PSA_SUCCESS = 验证通过; PSA_ERROR_INVALID_SIGNATURE = 不通过 */

psa_destroy_key(key);
psa_destroy_key(pub_key);
```

> ECC 曲线通过 `PSA_KEY_TYPE_ECC_KEY_PAIR(family)` / `PSA_KEY_TYPE_ECC_PUBLIC_KEY(family)` 构造，`family` 是 `PSA_ECC_FAMILY_*`（如 `PSA_ECC_FAMILY_SECP_R1`、`PSA_ECC_FAMILY_MONTGOMERY`、`PSA_ECC_FAMILY_TWISTED_EDWARDS`）。曲线位长由 `psa_set_key_bits` 指定。

### 4. RSA 签名

RSA 把密钥类型换成 `PSA_KEY_TYPE_RSA_KEY_PAIR`/`PSA_KEY_TYPE_RSA_PUBLIC_KEY`，算法换成 `PSA_ALG_RSA_PSS(PSA_ALG_SHA_256)` 或 `PSA_ALG_RSA_PKCS1V15_SIGN(PSA_ALG_SHA_256)`。其余流程与 ECDSA 相同。RSA 密钥位长常用 2048/3072/4096。

### 5. 原始密钥协商：psa_raw_key_agreement

ECDH/FFDH 原始协商直接输出共享秘密（来自 `include/psa/crypto.h`）：

```c
psa_algorithm_t alg = PSA_ALG_ECDH;   /* 或 PSA_ALG_FFDH */

/* 本地私钥 */
psa_key_attributes_t attr = PSA_KEY_ATTRIBUTES_INIT;
psa_set_key_type(&attr,
                 PSA_KEY_TYPE_ECC_KEY_PAIR(PSA_ECC_FAMILY_SECP_R1));
psa_set_key_bits(&attr, 256);
psa_set_key_usage_flags(&attr, PSA_KEY_USAGE_DERIVE);
psa_set_key_algorithm(&attr, alg);
psa_key_id_t my_priv;
psa_generate_key(&attr, &my_priv);

/* 我方公钥（发给对端） */
uint8_t my_pub[PSA_EXPORT_PUBLIC_KEY_MAX_SIZE];
size_t my_pub_len;
psa_export_public_key(my_priv, my_pub, sizeof(my_pub), &my_pub_len);

/* peer_pub 是对端公钥字节 */
uint8_t shared[PSA_RAW_KEY_AGREEMENT_OUTPUT_MAX_SIZE];
size_t shared_len;
psa_raw_key_agreement(alg, my_priv,
                      peer_pub, peer_pub_len,
                      shared, sizeof(shared), &shared_len);

psa_destroy_key(my_priv);
```

签名：

```c
psa_status_t psa_raw_key_agreement(psa_algorithm_t alg,
                                   mbedtls_svc_key_id_t private_key,
                                   const uint8_t *peer_key, size_t peer_key_length,
                                   uint8_t *output, size_t output_size,
                                   size_t *output_length);
```

> 原始 ECDH 输出**不应**直接用作对称密钥；应再过一层 KDF（见 `recipes/key_derivation.md` 的 `psa_key_derivation_key_agreement`）。

### 6. 签名/输出尺寸

| 宏 | 含义 |
|---|---|
| `PSA_SIGN_OUTPUT_SIZE(key_type, key_bits, alg)` | 签名输出上限 |
| `PSA_SIGNATURE_MAX_SIZE` | 任意已启用算法的签名上限 |
| `PSA_EXPORT_PUBLIC_KEY_MAX_SIZE` | 公钥导出上限 |
| `PSA_RAW_KEY_AGREEMENT_OUTPUT_MAX_SIZE` | 原始密钥协商输出上限（== `PSA_BITS_TO_BYTES(PSA_VENDOR_ECC_MAX_CURVE_BITS)` 等） |

## 常见错误

| 错误 | 原因 | 解决方法 |
|---|---|---|
| `PSA_ERROR_NOT_PERMITTED` | sign 用了只有 VERIFY flag 的密钥（或反之） | sign 需 `SIGN_HASH`/`SIGN_MESSAGE`；verify 需 `VERIFY_*` |
| `PSA_ERROR_NOT_SUPPORTED` | 算法/曲线/密钥类型未启用 | 启用对应 `PSA_WANT_ALG_*`、`PSA_WANT_ECC_*`、`PSA_WANT_KEY_TYPE_*_KEY_PAIR_*` |
| `PSA_ERROR_INVALID_SIGNATURE` | 验签失败 | 核对消息、hash、签名、公钥/私钥配对、算法 |
| `PSA_ERROR_INVALID_ARGUMENT` | `psa_sign_hash` 的 hash 长度与算法期望不符 | 用 `PSA_ALG_GET_HASH`/`PSA_ALG_SIGN_GET_HASH` 确定哈希，再算对应长度摘要 |
| 公钥导入失败 | curve family 或 bits 与私钥不一致 | 验签端的 `key_type`/`key_bits` 必须与签名端匹配 |
| 把原始 ECDH 输出当对称密钥 | 共享秘密需经 KDF 派生 | 用 `psa_key_derivation_key_agreement` + HKDF |
| ECDSA 签名不确定 | 用了 `PSA_ALG_ECDSA`（随机 k） | 确定性输出请用 `PSA_ALG_DETERMINISTIC_ECDSA(hash)` |

## 参考

- `include/psa/crypto.h` — `psa_sign_message`、`psa_verify_message`、`psa_sign_hash`、`psa_verify_hash`、`psa_export_public_key`、`psa_raw_key_agreement` 文档
- `include/psa/crypto.h` — `psa_key_derivation_key_agreement`（密钥协商 + KDF 结合）
- `programs/psa/key_ladder_demo.c` — `psa_export_key` 与 `psa_get_key_attributes` 用法
- `include/psa/crypto_values.h` — `PSA_ALG_ECDSA`、`PSA_ALG_DETERMINISTIC_ECDSA`、`PSA_ALG_RSA_PSS`、`PSA_ALG_RSA_PKCS1V15_SIGN`、`PSA_ALG_ECDH`、`PSA_ALG_FFDH`、`PSA_ECC_FAMILY_*`、`PSA_KEY_TYPE_ECC_*`/`PSA_KEY_TYPE_RSA_*`
- `include/psa/crypto_sizes.h` — `PSA_SIGN_OUTPUT_SIZE`、`PSA_SIGNATURE_MAX_SIZE`、`PSA_EXPORT_PUBLIC_KEY_MAX_SIZE`、`PSA_RAW_KEY_AGREEMENT_OUTPUT_MAX_SIZE`
