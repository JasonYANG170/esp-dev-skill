# 对称加密（AES-CBC / CTR / ChaCha20）

> **适用摘要**: 使用 PSA Crypto API 进行对称加密/解密。涵盖一次性 `psa_cipher_encrypt`/`psa_cipher_decrypt` 与分段 `psa_cipher_*_setup/generate_iv/set_iv/update/finish`，AES-CBC/PKCS7、CTR、ChaCha20 等模式。

## 触发意图

- "AES 加密 / 解密"
- "psa_cipher_encrypt / psa_cipher_decrypt"
- "分段对称加密 psa_cipher_update"
- "CTR / CBC / PKCS7 / ECB"
- "ChaCha20 stream cipher"
- "生成 IV psa_cipher_generate_iv"

## 前置条件

| 条件 | 要求 |
|---|---|
| 库状态 | 已成功调用 `psa_crypto_init()` |
| 头文件 | `#include <psa/crypto.h>` |
| 配置项 | `PSA_WANT_KEY_TYPE_AES` + 模式(如 `PSA_WANT_ALG_CBC_PKCS7`、`PSA_WANT_ALG_CTR`)；ChaCha20 需 `PSA_WANT_KEY_TYPE_CHACHA20` + `PSA_WANT_ALG_STREAM_CIPHER` |
| 参考示例 | `programs/psa/crypto_examples.c` |

## 分步说明

### 1. 算法与密钥

块密码模式（来自 `include/psa/crypto_values.h`）：

| 算法宏 | 说明 |
|---|---|
| `PSA_ALG_CBC_NO_PADDING` | CBC，输入须为块大小的整数倍 |
| `PSA_ALG_CBC_PKCS7` | CBC + PKCS7 填充 |
| `PSA_ALG_CTR` | CTR 流模式 |
| `PSA_ALG_CFB` | CFB |
| `PSA_ALG_OFB` | OFB |
| `PSA_ALG_ECB_NO_PADDING` | ECB（不推荐用于多块数据） |
| `PSA_ALG_STREAM_CIPHER` | 用于 ChaCha20 等流密码 |

AES 块大小：`PSA_BLOCK_CIPHER_BLOCK_LENGTH(PSA_KEY_TYPE_AES) == 16`。

### 2. 一次性加密：psa_cipher_encrypt

密钥已建好后，整段输入已知时用一次性函数（自动生成 IV 并前置到输出）：

```c
psa_algorithm_t alg = PSA_ALG_CTR;
uint8_t iv[16];
uint8_t output[input_size];   /* 流模式输出与输入等长 */
size_t output_len;

psa_status_t status = psa_cipher_encrypt(key, alg,
                                         input, input_size,
                                         output, sizeof(output),
                                         &output_len);
```

签名：

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

> 注意：`psa_cipher_encrypt` 的 `output` 中包含 IV（生成并前置）；`psa_cipher_decrypt` 的 `input` 中须包含前置的 IV。

### 3. 分段加密：setup / generate_iv / update / finish

需要自己掌控 IV 或分块处理时用分段 API。来自 `programs/psa/crypto_examples.c` 的 `cipher_encrypt`：

```c
psa_cipher_operation_t operation = PSA_CIPHER_OPERATION_INIT;
memset(&operation, 0, sizeof(operation));   /* 等价于 INIT 宏 */

psa_status_t status = psa_cipher_encrypt_setup(&operation, key, alg);
if (status != PSA_SUCCESS) { goto exit; }

uint8_t iv[16];
size_t iv_len;
status = psa_cipher_generate_iv(&operation, iv, sizeof(iv), &iv_len);
if (status != PSA_SUCCESS) { goto exit; }

/* 分块 update */
size_t len;
status = psa_cipher_update(&operation,
                           input_chunk, chunk_len,
                           output + out_written, out_size - out_written,
                           &len);
out_written += len;

/* finish 写入最后一块（含填充） */
status = psa_cipher_finish(&operation,
                           output + out_written, out_size - out_written,
                           &len);
out_written += len;

exit:
    psa_cipher_abort(&operation);   /* 成功/失败都调用 */
```

分段函数签名：

```c
psa_status_t psa_cipher_encrypt_setup(psa_cipher_operation_t *operation,
                                      mbedtls_svc_key_id_t key, psa_algorithm_t alg);
psa_status_t psa_cipher_decrypt_setup(psa_cipher_operation_t *operation,
                                      mbedtls_svc_key_id_t key, psa_algorithm_t alg);
psa_status_t psa_cipher_generate_iv(psa_cipher_operation_t *operation,
                                    uint8_t *iv, size_t iv_size,
                                    size_t *iv_length);
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

### 4. 解密：set_iv 而非 generate_iv

解密端使用 `psa_cipher_decrypt_setup` + `psa_cipher_set_iv`（IV 来自加密端）：

```c
psa_cipher_operation_t operation = PSA_CIPHER_OPERATION_INIT;
psa_cipher_decrypt_setup(&operation, key, alg);
psa_cipher_set_iv(&operation, iv, iv_size);
/* 后续 update / finish 与加密相同 */
```

### 5. AES-CBC + PKCS7 完整往返示例

综合 `programs/psa/crypto_examples.c` 中 `cipher_example_encrypt_decrypt_aes_cbc_pkcs7_multi`：

```c
enum { block_size = PSA_BLOCK_CIPHER_BLOCK_LENGTH(PSA_KEY_TYPE_AES),
       key_bits = 256, input_size = 100, part_size = 10 };
const psa_algorithm_t alg = PSA_ALG_CBC_PKCS7;

/* 建密钥 */
psa_key_attributes_t attributes = PSA_KEY_ATTRIBUTES_INIT;
psa_set_key_usage_flags(&attributes,
                        PSA_KEY_USAGE_ENCRYPT | PSA_KEY_USAGE_DECRYPT);
psa_set_key_algorithm(&attributes, alg);
psa_set_key_type(&attributes, PSA_KEY_TYPE_AES);
psa_set_key_bits(&attributes, key_bits);
psa_key_id_t key;
psa_generate_key(&attributes, &key);

/* 用前述分段 cipher_encrypt / cipher_decrypt 完成 100 字节明文
   按 10 字节一段 update；CBC+PKCS7 输出长度 = input_size + block_size */
uint8_t iv[block_size], encrypt[input_size + block_size], decrypt[input_size + block_size];
size_t output_len;
cipher_encrypt(key, alg, iv, sizeof(iv),
               input, sizeof(input), part_size,
               encrypt, sizeof(encrypt), &output_len);
cipher_decrypt(key, alg, iv, sizeof(iv),
               encrypt, output_len, part_size,
               decrypt, sizeof(decrypt), &output_len);
/* memcmp(input, decrypt, input_size) == 0 */

psa_destroy_key(key);
```

### 6. ChaCha20（流密码）

ChaCha20 用 `PSA_KEY_TYPE_CHACHA20` + `PSA_ALG_STREAM_CIPHER`：

```c
psa_set_key_type(&attr, PSA_KEY_TYPE_CHACHA20);
psa_set_key_bits(&attr, 256);
psa_set_key_usage_flags(&attr, PSA_KEY_USAGE_ENCRYPT | PSA_KEY_USAGE_DECRYPT);
psa_set_key_algorithm(&attr, PSA_ALG_STREAM_CIPHER);
```

（ChaCha20-Poly1305 AEAD 见 `recipes/aead.md`。）

## 常见错误

| 错误 | 原因 | 解决方法 |
|---|---|---|
| `PSA_ERROR_INVALID_ARGUMENT` | CBC_NO_PADDING 输入非块整数倍 | 改用 CBC_PKCS7，或将输入补齐到块边界 |
| `PSA_ERROR_NOT_PERMITTED` | 密钥缺 `ENCRYPT` 或 `DECRYPT` flag | 用 `\|` 同时设置两个 flag |
| `PSA_ERROR_BAD_STATE` | 分段操作未初始化或顺序错误（如先 update 再 setup） | 用 `PSA_CIPHER_OPERATION_INIT`；顺序：setup→(generate/set iv)→update→finish |
| `PSA_ERROR_BUFFER_TOO_SMALL` | 输出缓冲不足（CBC+PKCS7 输出 > 输入） | 至少 `input_size + block_size` |
| 解密结果前几个字节是 IV | 误把含前置 IV 的 `psa_cipher_encrypt` 输出直接当密文喂给分段 `set_iv` | 一次性 API 的输出含 IV；分段 API 的 IV 独立管理 |
| `PSA_ERROR_INVALID_PADDING` | 解密时填充损坏 | 检查密文完整性、密钥、IV 是否一致 |

## 参考

- `programs/psa/crypto_examples.c` — AES-CBC no-padding、CBC-PKCS7、CTR 三种模式的分段 encrypt/decrypt��含 `cipher_operation` 通用流程
- `include/psa/crypto.h` — `psa_cipher_encrypt`、`psa_cipher_decrypt`、`psa_cipher_*_setup`、`psa_cipher_generate_iv`、`psa_cipher_set_iv`、`psa_cipher_update`、`psa_cipher_finish`、`psa_cipher_abort` 文档
- `include/psa/crypto_values.h` — `PSA_ALG_CBC_*`、`PSA_ALG_CTR`、`PSA_ALG_STREAM_CIPHER`、`PSA_ALG_IS_CIPHER`（如有）、`PSA_BLOCK_CIPHER_BLOCK_LENGTH`
