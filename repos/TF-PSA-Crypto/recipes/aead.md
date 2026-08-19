# AEAD（AES-GCM / AES-CCM / ChaCha20-Poly1305）

> **适用摘要**: 使用 PSA Crypto API 进行带认证的加密/解密(AEAD)。涵盖一次性 `psa_aead_encrypt`/`psa_aead_decrypt` 与分段 `psa_aead_*_setup/set_nonce/update_ad/update/finish/verify`，附加数据(AD)处理、短标签。

> Version: selected repo/component version; align with the user project and dependency manifest.
> Evidence: `repos/TF-PSA-Crypto/resources/`, source/examples in `repos/TF-PSA-Crypto/`, and this recipe path `repos/TF-PSA-Crypto/recipes/aead.md`.
> Validation: draft metadata added from repository routing; verify APIs, Kconfig symbols, and component dependencies against the selected version.

## 触发意图

- "AES-GCM 加密"
- "psa_aead_encrypt / psa_aead_decrypt"
- "分段 AEAD psa_aead_update_ad"
- "ChaCha20-Poly1305"
- "附加数据 / additional data / AD"
- "短标签 shortened tag"

## 前置条件

| 条件 | 要求 |
|---|---|
| 库状态 | 已成功调用 `psa_crypto_init()` |
| 头文件 | `#include <psa/crypto.h>` |
| 配置项 | `PSA_WANT_ALG_GCM` 或 `PSA_WANT_ALG_CCM` 或 `PSA_WANT_ALG_CHACHA20_POLY1305`；对应密钥类型 `PSA_WANT_KEY_TYPE_AES` / `PSA_WANT_KEY_TYPE_CHACHA20` |
| 参考示例 | `programs/psa/aead_demo.c`、`programs/psa/key_ladder_demo.c` |

## 分步说明

### 1. 算法

| 算法宏 | 说明 |
|---|---|
| `PSA_ALG_GCM` | AES-GCM，默认 tag 16 字节 |
| `PSA_ALG_CCM` | AES-CCM，默认 tag 16 字节 |
| `PSA_ALG_CHACHA20_POLY1305` | ChaCha20-Poly1305，tag 固定 16 字节 |
| `PSA_ALG_CCM_STAR_NO_TAG` | CCM*（无 tag） |

要使用短标签，用 `PSA_ALG_AEAD_WITH_SHORTENED_TAG(aead_alg, tag_len)`（来自 `programs/psa/aead_demo.c`）：

```c
*alg = PSA_ALG_AEAD_WITH_SHORTENED_TAG(PSA_ALG_GCM, 8);   /* AES-GCM 8 字节 tag */
```

tag 长度可用 `PSA_AEAD_TAG_LENGTH(key_type, key_bits, alg)` 反查。

### 2. 准备密钥

来自 `programs/psa/aead_demo.c` 的 `aead_prepare`：

```c
psa_key_attributes_t attributes = PSA_KEY_ATTRIBUTES_INIT;
psa_set_key_usage_flags(&attributes, PSA_KEY_USAGE_ENCRYPT);   /* 解密端用 DECRYPT */
psa_set_key_algorithm(&attributes, alg);                       /* alg 已含短 tag 信息 */
psa_set_key_type(&attributes, PSA_KEY_TYPE_AES);               /* 或 CHACHA20 */
psa_set_key_bits(&attributes, key_bits);                       /* 128/256 */

psa_key_id_t key;
psa_import_key(&attributes, key_bytes, key_bits / 8, &key);
```

### 3. 一次性 AEAD：psa_aead_encrypt / psa_aead_decrypt

整段明文与 AD 已知时用一次性函数（来自 `programs/psa/key_ladder_demo.c` 的 `wrap_data`）：

```c
/* 输出缓冲 = 密文 + tag。用宏定尺寸 */
size_t buf_size = PSA_AEAD_ENCRYPT_OUTPUT_SIZE(key_type, alg, plaintext_size);
uint8_t ciphertext[buf_size];   /* 至少 PSA_AEAD_ENCRYPT_OUTPUT_MAX_SIZE(pt_len) */
size_t ciphertext_size;

psa_generate_random(iv, WRAPPING_IV_SIZE);   /* 13 字节 nonce（CCM 常用） */

psa_status_t status = psa_aead_encrypt(wrapping_key, alg,
                                       iv, WRAPPING_IV_SIZE,
                                       ad, ad_size,            /* 附加数据 */
                                       plaintext, plaintext_size,
                                       ciphertext, sizeof(ciphertext),
                                       &ciphertext_size);
```

解密端 `psa_aead_decrypt`：解密并验证 tag；tag 不匹配返回 `PSA_ERROR_INVALID_SIGNATURE`。

签名：

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

### 4. 分段 AEAD：encrypt

数据分批到达、或想先写 AD 再写明文时用分段 API。来自 `programs/psa/aead_demo.c` 的 `aead_encrypt`：

```c
psa_aead_operation_t op = PSA_AEAD_OPERATION_INIT;
unsigned char out[PSA_AEAD_ENCRYPT_OUTPUT_MAX_SIZE(MSG_MAX_SIZE)];
unsigned char tag[PSA_AEAD_TAG_MAX_SIZE];
unsigned char *p = out, *end = out + sizeof(out);
size_t olen, olen_tag;

psa_aead_encrypt_setup(&op, key, alg);
psa_aead_set_nonce(&op, iv, iv_len);
psa_aead_update_ad(&op, ad, ad_len);

psa_aead_update(&op, part1, part1_len, p, end - p, &olen); p += olen;
psa_aead_update(&op, part2, part2_len, p, end - p, &olen); p += olen;

psa_aead_finish(&op, p, end - p, &olen, tag, sizeof(tag), &olen_tag);
p += olen;
memcpy(p, tag, olen_tag); p += olen_tag;
/* 总输出 = p - out */
```

> 顺序必须为：`setup → set_nonce/generate_nonce → (可选 set_lengths) → update_ad → update → finish`。

分段函数签名：

```c
psa_status_t psa_aead_encrypt_setup(psa_aead_operation_t *operation,
                                    mbedtls_svc_key_id_t key, psa_algorithm_t alg);
psa_status_t psa_aead_decrypt_setup(psa_aead_operation_t *operation,
                                    mbedtls_svc_key_id_t key, psa_algorithm_t alg);
psa_status_t psa_aead_generate_nonce(psa_aead_operation_t *operation,
                                     uint8_t *nonce, size_t nonce_size,
                                     size_t *nonce_length);
psa_status_t psa_aead_set_nonce(psa_aead_operation_t *operation,
                                const uint8_t *nonce, size_t nonce_length);
psa_status_t psa_aead_set_lengths(psa_aead_operation_t *operation,
                                  size_t ad_length, size_t plaintext_length);
psa_status_t psa_aead_update_ad(psa_aead_operation_t *operation,
                                const uint8_t *input, size_t input_length);
psa_status_t psa_aead_update(psa_aead_operation_t *operation,
                             const uint8_t *input, size_t input_length,
                             uint8_t *output, size_t output_size,
                             size_t *output_length);
psa_status_t psa_aead_finish(psa_aead_operation_t *operation,
                             uint8_t *ciphertext, size_t ciphertext_size,
                             size_t *ciphertext_length,
                             uint8_t *tag, size_t tag_size,
                             size_t *tag_length);
psa_status_t psa_aead_verify(psa_aead_operation_t *operation,
                             uint8_t *plaintext, size_t plaintext_size,
                             size_t *plaintext_length,
                             const uint8_t *tag, size_t tag_length);
psa_status_t psa_aead_abort(psa_aead_operation_t *operation);
```

### 5. 分段解密：verify 而非 finish

解密端用 `psa_aead_decrypt_setup` + `psa_aead_verify`。`verify` 与 `finish` 的区别：`verify` 在 tag 不匹配时返回 `PSA_ERROR_INVALID_SIGNATURE` 且**不输出明文**（更安全）。

### 6. 缓冲区尺寸宏

| 宏 | 含义 |
|---|---|
| `PSA_AEAD_TAG_LENGTH(key_type, key_bits, alg)` | 该组合的 tag 字节数 |
| `PSA_AEAD_TAG_MAX_SIZE` | 最大 tag 长度（== 16） |
| `PSA_AEAD_ENCRYPT_OUTPUT_SIZE(key_type, alg, plaintext_len)` | 加密输出长度（密文+tag） |
| `PSA_AEAD_ENCRYPT_OUTPUT_MAX_SIZE(plaintext_len)` | 任意 AEAD 的加密输出上限 |
| `PSA_AEAD_DECRYPT_OUTPUT_SIZE(...)` / `PSA_AEAD_UPDATE_OUTPUT_SIZE(...)` | 解密/分段 update 输出长度 |

## 常见错误

| 错误 | 原因 | 解决方法 |
|---|---|---|
| `PSA_ERROR_BAD_STATE` | 调用顺序错（如 update 在 set_nonce/update_ad 之前） | 严格按 setup→nonce→ad→update→finish/verify 顺序 |
| `PSA_ERROR_INVALID_SIGNATURE` | 解密时 tag 不匹配（密文/AD/nonce/密钥被篡改或错误） | 核对所有输入；tag 失败即说明数据不可信 |
| `PSA_ERROR_NOT_PERMITTED` | 密钥缺 `ENCRYPT`(加密) 或 `DECRYPT`(解密) flag | 加密端补 `PSA_KEY_USAGE_ENCRYPT`，解密端补 `DECRYPT` |
| nonce 复用 | 同一密钥+nonce 加密多条消息会破坏 GCM/CCM 安全性 | 每次加密用 `psa_aead_generate_nonce` 或计数器 nonce，绝不复用 |
| `PSA_ERROR_BUFFER_TOO_SMALL` | 输出缓冲不足 | 加密输出至少 `PSA_AEAD_ENCRYPT_OUTPUT_SIZE(...)`；分段 update 用 `PSA_AEAD_UPDATE_OUTPUT_SIZE` |
| 短 tag 算法不生效 | 密钥的 `psa_set_key_algorithm` 用了默认 tag 而非短 tag 算法 | 密钥算法与操作算法都用 `PSA_ALG_AEAD_WITH_SHORTENED_TAG(...)` |

## 参考

- `programs/psa/aead_demo.c` — AES-128-GCM、AES-256-GCM、AES-128-GCM-8(短tag)、ChaCha20-Poly1305 的分段加密演示
- `programs/psa/key_ladder_demo.c` — `psa_aead_encrypt`/`psa_aead_decrypt` 一次性 wrap/unwrap 文件数据
- `include/psa/crypto.h` — `psa_aead_encrypt`、`psa_aead_decrypt`、`psa_aead_*_setup`、`psa_aead_generate_nonce`、`psa_aead_set_nonce`、`psa_aead_set_lengths`、`psa_aead_update_ad`、`psa_aead_update`、`psa_aead_finish`、`psa_aead_verify`、`psa_aead_abort` 文档
- `include/psa/crypto_values.h` — `PSA_ALG_GCM`、`PSA_ALG_CCM`、`PSA_ALG_CHACHA20_POLY1305`、`PSA_ALG_AEAD_WITH_SHORTENED_TAG`、`PSA_ALG_IS_AEAD`
- `include/psa/crypto_sizes.h` — `PSA_AEAD_TAG_LENGTH`、`PSA_AEAD_TAG_MAX_SIZE`、`PSA_AEAD_ENCRYPT_OUTPUT_SIZE`、`PSA_AEAD_ENCRYPT_OUTPUT_MAX_SIZE`
