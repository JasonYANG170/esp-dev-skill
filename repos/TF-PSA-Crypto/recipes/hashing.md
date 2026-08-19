# 哈希（SHA-256 / SHA-3 等）

> **适用摘要**: 使用 PSA Crypto API 计算消息摘要。涵盖一次性(one-shot) `psa_hash_compute` 和分段(multi-part) `psa_hash_setup/update/finish`，以及 `psa_hash_clone` 克隆与 `psa_hash_verify` 比对。

> Version: selected repo/component version; align with the user project and dependency manifest.
> Evidence: `repos/TF-PSA-Crypto/resources/`, source/examples in `repos/TF-PSA-Crypto/`, and this recipe path `repos/TF-PSA-Crypto/recipes/hashing.md`.
> Validation: draft metadata added from repository routing; verify APIs, Kconfig symbols, and component dependencies against the selected version.

## 触发意图

- "计算 SHA-256"
- "psa_hash_compute 怎么用"
- "流式哈希 / 分段哈希"
- "psa_hash_update / finish"
- "克隆哈希操作 psa_hash_clone"

## 前置条件

| 条件 | 要求 |
|---|---|
| 库状态 | 已成功调用 `psa_crypto_init()` |
| 头文件 | `#include <psa/crypto.h>` |
| 配置项 | 对应哈希算法已启用，如 `PSA_WANT_ALG_SHA_256` |
| 参考示例 | `programs/psa/psa_hash.c` |

## 分步说明

### 1. 选择算法

哈希算法（来自 `include/psa/crypto_values.h`）：`PSA_ALG_SHA_1`、`PSA_ALG_SHA_224`、`PSA_ALG_SHA_256`、`PSA_ALG_SHA_384`、`PSA_ALG_SHA_512`、`PSA_ALG_SHA3_256`、`PSA_ALG_SHA3_512`、`PSA_ALG_MD5`、`PSA_ALG_RIPEMD160`。

### 2. 一次性哈希：psa_hash_compute

整段消息已知时用一次性函数（来自 `programs/psa/psa_hash.c`）：

```c
#define HASH_ALG PSA_ALG_SHA_256

const uint8_t sample_message[] = "Hello World!";
const size_t sample_message_length = sizeof(sample_message) - 1;

uint8_t hash[PSA_HASH_LENGTH(HASH_ALG)];   /* 由算法算出长度，SHA-256 => 32 */
size_t hash_length;

psa_status_t status = psa_hash_compute(HASH_ALG,
                                       sample_message, sample_message_length,
                                       hash, sizeof(hash),
                                       &hash_length);
if (status != PSA_SUCCESS) { /* 处理 */ }
```

签名：

```c
psa_status_t psa_hash_compute(psa_algorithm_t alg,
                              const uint8_t *input, size_t input_length,
                              uint8_t *hash, size_t hash_size,
                              size_t *hash_length);
```

### 3. 一次性比对：psa_hash_compare

计算哈希并与期望值比对（不匹配返回 `PSA_ERROR_INVALID_SIGNATURE`）：

```c
psa_status_t psa_hash_compare(HASH_ALG,
                              input, input_length,
                              expected_hash, expected_hash_length);
/* PSA_SUCCESS = 匹配; PSA_ERROR_INVALID_SIGNATURE = 不匹配 */
```

### 4. 分段哈希：psa_hash_setup / update / finish

数据分批到达或需要边读边哈希时用分段 API（来自 `programs/psa/psa_hash.c`）：

```c
psa_hash_operation_t hash_operation = PSA_HASH_OPERATION_INIT;

psa_status_t status = psa_hash_setup(&hash_operation, HASH_ALG);
if (status == PSA_ERROR_NOT_SUPPORTED) { /* 算法未启用 */ }
else if (status != PSA_SUCCESS) { /* 其它错误 */ }

status = psa_hash_update(&hash_operation,
                         sample_message, sample_message_length);
if (status != PSA_SUCCESS) { goto cleanup; }

uint8_t hash[PSA_HASH_LENGTH(HASH_ALG)];
size_t hash_length;
status = psa_hash_finish(&hash_operation, hash, sizeof(hash), &hash_length);
if (status != PSA_SUCCESS) { goto cleanup; }

/* ...使用 hash... */
return;

cleanup:
    psa_hash_abort(&hash_operation);   /* 出错后必须 abort */
```

分段函数签名：

```c
psa_status_t psa_hash_setup(psa_hash_operation_t *operation, psa_algorithm_t alg);
psa_status_t psa_hash_update(psa_hash_operation_t *operation,
                             const uint8_t *input, size_t input_length);
psa_status_t psa_hash_finish(psa_hash_operation_t *operation,
                             uint8_t *hash, size_t hash_size,
                             size_t *hash_length);
psa_status_t psa_hash_verify(psa_hash_operation_t *operation,
                             const uint8_t *hash, size_t hash_length);
psa_status_t psa_hash_abort(psa_hash_operation_t *operation);
```

### 5. 克隆哈希：psa_hash_clone

把一个进行中的哈希操作复制成另一个独立操作（例如对同一前缀计算多个不同后缀的哈希）。来自 `programs/psa/psa_hash.c`：

```c
psa_hash_operation_t hash_operation          = PSA_HASH_OPERATION_INIT;
psa_hash_operation_t cloned_hash_operation   = PSA_HASH_OPERATION_INIT;

psa_hash_setup(&hash_operation, HASH_ALG);
psa_hash_update(&hash_operation, msg, msg_len);

/* 克隆当前进度 */
psa_hash_clone(&hash_operation, &cloned_hash_operation);

/* 原操作继续 finish 得到一个哈希 */
psa_hash_finish(&hash_operation, hash, sizeof(hash), &hash_len);

/* 克隆出的操作可用于 verify 同一前缀的哈希 */
psa_hash_verify(&cloned_hash_operation, expected, expected_len);
```

`psa_hash_clone` 签名：

```c
psa_status_t psa_hash_clone(const psa_hash_operation_t *source_operation,
                            psa_hash_operation_t *target_operation);
```

### 6. 缓冲区尺寸宏

| 宏 | 含义 |
|---|---|
| `PSA_HASH_LENGTH(alg)` | 指定算法的输出字节数 |
| `PSA_HASH_MAX_SIZE` | 任意已启用哈希算法的最大输出字节数 |

```c
uint8_t hash[PSA_HASH_LENGTH(PSA_ALG_SHA_256)];   /* 编译期确定，32 */
uint8_t any_hash[PSA_HASH_MAX_SIZE];              /* 通用上限 */
```

## 常见错误

| 错误 | 原因 | 解决方法 |
|---|---|---|
| `PSA_ERROR_NOT_SUPPORTED` | 算法未启用 | 在 `crypto_config.h` 启用对应 `PSA_WANT_ALG_SHA_*` |
| `PSA_ERROR_BAD_STATE` | 未 `psa_crypto_init`，或操作对象未用 `PSA_HASH_OPERATION_INIT` 初始化，或出错后未 abort 又继续 | 先 init 库；操作对象零初始化；出错后 `psa_hash_abort` |
| `PSA_ERROR_BUFFER_TOO_SMALL` | `hash` 缓冲区小于算法输出 | 用 `PSA_HASH_LENGTH(alg)` 定尺寸 |
| `PSA_ERROR_INVALID_ARGUMENT` | `psa_hash_compare` 的 `hash_length` 与算法输出长度不符 | 比对前确认长度匹配 |
| `psa_hash_clone` 后原操作失效 | 误解 clone 语义 | clone 只是复制进度；原操作仍可继续 update/finish |

## 参考

- `programs/psa/psa_hash.c` — 一次性 + 分段 + clone 的完整示例
- `include/psa/crypto.h` — `psa_hash_compute`、`psa_hash_compare`、`psa_hash_setup`、`psa_hash_update`、`psa_hash_finish`、`psa_hash_verify`、`psa_hash_abort`、`psa_hash_clone` 文档
- `include/psa/crypto_values.h` — `PSA_ALG_SHA_*`、`PSA_ALG_IS_HASH` 定义
- `include/psa/crypto_sizes.h` — `PSA_HASH_LENGTH`、`PSA_HASH_MAX_SIZE`
