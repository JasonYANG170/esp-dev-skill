# MAC（HMAC / AES-CMAC）

> **适用摘要**: 使用 PSA Crypto API 计算消息认证码。涵盖 HMAC（任意哈希）与 AES-CMAC，一次性 `psa_mac_compute`/`psa_mac_verify` 与分段 `psa_mac_sign_setup/update/finish`，以及签名(sign)与验签(verify)的对称用法。

## 触发意图

- "计算 HMAC-SHA256"
- "psa_mac_compute / psa_mac_verify"
- "分段 HMAC psa_mac_update"
- "AES-CMAC"
- "PSA MAC 签名 / 验签"

## 前置条件

| 条件 | 要求 |
|---|---|
| 库状态 | 已成功调用 `psa_crypto_init()` |
| 头文件 | `#include <psa/crypto.h>` |
| 配置项 | `PSA_WANT_ALG_HMAC` + 对应哈希(如 `PSA_WANT_ALG_SHA_256`)；CMAC 还需 `PSA_WANT_ALG_CMAC` 与 `PSA_WANT_KEY_TYPE_AES` |
| 参考示例 | `programs/psa/hmac_demo.c` |

## 分步说明

### 1. 准备 MAC 密钥

HMAC 用 `PSA_KEY_TYPE_HMAC` + 算法 `PSA_ALG_HMAC(hash)`；CMAC 用 `PSA_KEY_TYPE_AES` + `PSA_ALG_CMAC`。

来自 `programs/psa/hmac_demo.c` 的 HMAC 密钥准备：

```c
const psa_algorithm_t alg = PSA_ALG_HMAC(PSA_ALG_SHA_256);
const unsigned char key_bytes[32] = { /* 32 字节密钥 */ };

psa_key_attributes_t attributes = PSA_KEY_ATTRIBUTES_INIT;
psa_set_key_usage_flags(&attributes, PSA_KEY_USAGE_SIGN_MESSAGE);
psa_set_key_algorithm(&attributes, alg);
psa_set_key_type(&attributes, PSA_KEY_TYPE_HMAC);
psa_set_key_bits(&attributes, 8 * sizeof(key_bytes));   /* 可选但推荐 */

psa_key_id_t key = 0;
psa_status_t status = psa_import_key(&attributes,
                                     key_bytes, sizeof(key_bytes), &key);
```

> 注意：HMAC 算法通过宏 `PSA_ALG_HMAC(hash_alg)` 构造，`hash_alg` 可以是任意已启用的哈希。

### 2. 一次性 MAC：psa_mac_compute / psa_mac_verify

整段消息已知时：

```c
uint8_t mac[PSA_MAC_MAX_SIZE];   /* 安全但非最优；可用 PSA_MAC_LENGTH(...) */
size_t mac_len;

psa_mac_compute(key, alg,
                msg, msg_len,
                mac, sizeof(mac), &mac_len);

/* 验签端 */
psa_status_t v = psa_mac_verify(key, alg,
                                msg, msg_len,
                                expected_mac, expected_mac_len);
/* PSA_SUCCESS = 匹配; PSA_ERROR_INVALID_SIGNATURE = 不匹配 */
```

签名：

```c
psa_status_t psa_mac_compute(mbedtls_svc_key_id_t key, psa_algorithm_t alg,
                             const uint8_t *input, size_t input_length,
                             uint8_t *mac, size_t mac_size,
                             size_t *mac_length);
psa_status_t psa_mac_verify(mbedtls_svc_key_id_t key, psa_algorithm_t alg,
                            const uint8_t *input, size_t input_length,
                            const uint8_t *mac, size_t mac_length);
```

### 3. 分段 MAC：sign setup / update / sign finish

数据分批到达时用分段 API（来自 `programs/psa/hmac_demo.c`）：

```c
psa_mac_operation_t op = PSA_MAC_OPERATION_INIT;
size_t out_len = 0;
uint8_t out[PSA_MAC_MAX_SIZE];

/* 计算 HMAC(key, msg1_part1 | msg1_part2) */
psa_mac_sign_setup(&op, key, alg);
psa_mac_update(&op, msg1_part1, sizeof(msg1_part1));
psa_mac_update(&op, msg1_part2, sizeof(msg1_part2));
psa_mac_sign_finish(&op, out, sizeof(out), &out_len);

/* 同一个 op 对象可继续 setup 计算下一条消息（已 finish 后回到空闲态） */
psa_mac_sign_setup(&op, key, alg);
psa_mac_update(&op, msg2_part1, sizeof(msg2_part1));
psa_mac_update(&op, msg2_part2, sizeof(msg2_part2));
psa_mac_sign_finish(&op, out, sizeof(out), &out_len);
```

分段函数签名：

```c
psa_status_t psa_mac_sign_setup(psa_mac_operation_t *operation,
                                mbedtls_svc_key_id_t key, psa_algorithm_t alg);
psa_status_t psa_mac_verify_setup(psa_mac_operation_t *operation,
                                  mbedtls_svc_key_id_t key, psa_algorithm_t alg);
psa_status_t psa_mac_update(psa_mac_operation_t *operation,
                            const uint8_t *input, size_t input_length);
psa_status_t psa_mac_sign_finish(psa_mac_operation_t *operation,
                                 uint8_t *mac, size_t mac_size,
                                 size_t *mac_length);
psa_status_t psa_mac_verify_finish(psa_mac_operation_t *operation,
                                   const uint8_t *mac, size_t mac_length);
psa_status_t psa_mac_abort(psa_mac_operation_t *operation);
```

### 4. 验签端：verify_setup / update / verify_finish

```c
psa_mac_operation_t op = PSA_MAC_OPERATION_INIT;
psa_mac_verify_setup(&op, key, alg);
psa_mac_update(&op, msg, msg_len);
psa_status_t status = psa_mac_verify_finish(&op, expected_mac, expected_mac_len);
/* PSA_SUCCESS = 验证通过 */
```

`psa_mac_verify_finish` 与 `psa_mac_sign_finish` 的区别：前者不输出 MAC，而是与传入的 `mac` 比对，返回 `PSA_SUCCESS` 或 `PSA_ERROR_INVALID_SIGNATURE`。

### 5. 出错清理

任何 `psa_mac_*` 函数返回非 `PSA_SUCCESS`（除少数情况）都会让操作对象进入错误态，**必须**调用 `psa_mac_abort`。`psa_mac_abort` 在成功路径上调用也无害：

```c
exit:
    psa_mac_abort(&op);   /* 出错时必需，成功时无害 */
    psa_destroy_key(key);
    mbedtls_platform_zeroize(out, sizeof(out));   /* 抹除 MAC（来自 hmac_demo.c） */
```

### 6. 缓冲区尺寸

| 宏 | 含义 |
|---|---|
| `PSA_MAC_LENGTH(key_type, key_bits, alg)` | 该组合的 MAC 输出长度 |
| `PSA_MAC_MAX_SIZE` | `== PSA_HASH_MAX_SIZE`，足够任意已启用 HMAC |

`programs/psa/hmac_demo.c` 直接用 `PSA_MAC_MAX_SIZE` 作为缓冲（"安全但非最优"），生产代码可用 `PSA_MAC_LENGTH(...)` 精确定尺寸。

## 常见错误

| 错误 | 原因 | 解决方法 |
|---|---|---|
| `PSA_ERROR_NOT_PERMITTED` | 密钥缺 `PSA_KEY_USAGE_SIGN_MESSAGE`(sign) 或 `PSA_KEY_USAGE_VERIFY_MESSAGE`(verify) | 在属性里补上对应 flag |
| `PSA_ERROR_NOT_SUPPORTED` | 算法未启用 | 启用 `PSA_WANT_ALG_HMAC` + 哈希，或 `PSA_WANT_ALG_CMAC` |
| `PSA_ERROR_INVALID_SIGNATURE` | `psa_mac_verify_finish` / `psa_mac_verify` 比对不匹配 | 检查密钥、消息、算法、字节序 |
| `PSA_ERROR_BAD_STATE` | 操作对象未用 `PSA_MAC_OPERATION_INIT` 初始化，或出错后未 abort | 零初始化；出错后 `psa_mac_abort` |
| 算法与密钥类型不兼容 | 用 AES 密钥跑 HMAC，或用 HMAC 密钥跑 CMAC | HMAC↔`PSA_KEY_TYPE_HMAC`；CMAC↔`PSA_KEY_TYPE_AES` |

## 参考

- `programs/psa/hmac_demo.c` — HMAC-SHA256 分段 sign，含 `PSA_CHECK` 与 `psa_mac_abort` 清理
- `include/psa/crypto.h` — `psa_mac_compute`、`psa_mac_verify`、`psa_mac_sign_setup`、`psa_mac_verify_setup`、`psa_mac_update`、`psa_mac_sign_finish`、`psa_mac_verify_finish`、`psa_mac_abort` 文档
- `include/psa/crypto_values.h` — `PSA_ALG_HMAC`、`PSA_ALG_CMAC`、`PSA_ALG_IS_MAC`、`PSA_KEY_USAGE_SIGN_MESSAGE`/`VERIFY_MESSAGE`
- `include/psa/crypto_sizes.h` — `PSA_MAC_LENGTH`、`PSA_MAC_MAX_SIZE`
