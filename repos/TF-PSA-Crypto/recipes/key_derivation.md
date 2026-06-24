# 密钥派生（HKDF / PBKDF2 / TLS12-PRF）

> **适用摘要**: 使用 PSA Crypto API 的密钥派生框架从主密钥派生新密钥或密钥材料。涵盖 HKDF、PBKDF2、TLS12-PRF 的输入步骤、`psa_key_derivation_output_key` 直接派生密钥对象、以及密钥阶梯(key ladder)模式。

## 触发意图

- "HKDF 派生密钥"
- "psa_key_derivation_setup / input_bytes / input_key / output_bytes"
- "PBKDF2 从口令派生密钥"
- "TLS12-PRF"
- "psa_key_derivation_output_key 直接派生新密钥"
- "密钥阶梯 key ladder"

## 前置条件

| 条件 | 要求 |
|---|---|
| 库状态 | 已成功调用 `psa_crypto_init()` |
| 头文件 | `#include <psa/crypto.h>` |
| 配置项 | `PSA_WANT_ALG_HKDF`（+ 哈希如 `PSA_WANT_ALG_SHA_256`）；PBKDF2 需 `PSA_WANT_ALG_PBKDF2_HMAC`；TLS12-PRF 需 `PSA_WANT_ALG_TLS12_PRF`；派生源密钥类型 `PSA_WANT_KEY_TYPE_DERIVE` |
| 参考示例 | `programs/psa/key_ladder_demo.c` |

## 分步说明

### 1. KDF 算法

| 算法宏 | 输入步骤 |
|---|---|
| `PSA_ALG_HKDF(hash)` | SALT, SECRET, INFO |
| `PSA_ALG_HKDF_EXTRACT(hash)` | SALT, SECRET |
| `PSA_ALG_HKDF_EXPAND(hash)` | SECRET, INFO |
| `PSA_ALG_PBKDF2_HMAC(hash)` | SALT, PASSWORD(=SECRET), cost 整数 |
| `PSA_ALG_TLS12_PRF(hash)` | SECRET, LABEL, SEED |
| `PSA_ALG_TLS12_PSK_TO_MS(hash)` | SECRET(PSK), OTHER_SECRET, LABEL, SEED |

输入步骤常量（`include/psa/crypto_values.h`）：`PSA_KEY_DERIVATION_INPUT_SECRET`(0x0101)、`PSA_KEY_DERIVATION_INPUT_OTHER_SECRET`(0x0102)、`PSA_KEY_DERIVATION_INPUT_LABEL`(0x0201)、`PSA_KEY_DERIVATION_INPUT_SALT`(0x0202)、`PSA_KEY_DERIVATION_INPUT_INFO`(0x0203)、`PSA_KEY_DERIVATION_INPUT_SEED`(0x0204)。

容量用 `psa_key_derivation_set_capacity(op, n)`；`PSA_KEY_DERIVATION_UNLIMITED_CAPACITY == ((size_t)-1)`。

### 2. 通用四步流程

1. `psa_key_derivation_setup(&op, alg)` — 选择 KDF
2. 提供输入：`psa_key_derivation_input_bytes(&op, step, data, len)`（非秘密）或 `psa_key_derivation_input_key(&op, step, key)`（秘密，推荐）
3. 取输出：`psa_key_derivation_output_bytes(&op, out, n)` 或 `psa_key_derivation_output_key(&attributes, &op, &new_key)`（直接得到密钥对象）
4. `psa_key_derivation_abort(&op)` — 清理（成功/失败都调用）

### 3. HKDF 密钥阶梯（来自 key_ladder_demo.c）

`programs/psa/key_ladder_demo.c` 的 `derive_key_ladder` 演示了逐级派生：

```c
#define KDF_ALG PSA_ALG_HKDF(PSA_ALG_SHA_256)
#define DERIVE_KEY_SALT ((uint8_t *) "key_ladder_demo.derive")
#define KEY_SIZE_BYTES 40

psa_key_attributes_t attributes = PSA_KEY_ATTRIBUTES_INIT;
psa_set_key_usage_flags(&attributes, PSA_KEY_USAGE_DERIVE | PSA_KEY_USAGE_EXPORT);
psa_set_key_algorithm(&attributes, KDF_ALG);
psa_set_key_type(&attributes, PSA_KEY_TYPE_DERIVE);
psa_set_key_bits(&attributes, PSA_BYTES_TO_BITS(KEY_SIZE_BYTES));

for (size_t i = 0; i < ladder_depth; i++) {
    psa_key_derivation_operation_t operation =
        PSA_KEY_DERIVATION_OPERATION_INIT;

    psa_key_derivation_setup(&operation, KDF_ALG);
    psa_key_derivation_input_bytes(&operation,
                                   PSA_KEY_DERIVATION_INPUT_SALT,
                                   DERIVE_KEY_SALT, DERIVE_KEY_SALT_LENGTH);
    psa_key_derivation_input_key(&operation,
                                 PSA_KEY_DERIVATION_INPUT_SECRET,
                                 *key);                          /* 父密钥 */
    psa_key_derivation_input_bytes(&operation,
                                   PSA_KEY_DERIVATION_INPUT_INFO,
                                   (uint8_t *) ladder[i], strlen(ladder[i]));

    psa_destroy_key(*key);   /* 销毁父密钥（已不再需要） */
    *key = 0;

    /* 直接派生出新密钥对象 */
    psa_key_derivation_output_key(&attributes, &operation, key);
    psa_key_derivation_abort(&operation);
}
```

要点：

- 派生源密钥用 `PSA_KEY_TYPE_DERIVE`，并设 `PSA_KEY_USAGE_DERIVE` flag。
- `psa_key_derivation_output_key(&attributes, &operation, &key)` 用 `attributes` 决定输出密钥的类型/长度/用途；秘密不离开隔离边界（比 output_bytes + import_key 更安全）。

### 4. 关键函数签名

```c
psa_status_t psa_key_derivation_setup(psa_key_derivation_operation_t *operation,
                                      psa_algorithm_t alg);
psa_status_t psa_key_derivation_input_bytes(psa_key_derivation_operation_t *operation,
                                            psa_key_derivation_step_t step,
                                            const uint8_t *data, size_t data_length);
psa_status_t psa_key_derivation_input_integer(psa_key_derivation_operation_t *operation,
                                              psa_key_derivation_step_t step,
                                              uint64_t value);   /* PBKDF2 迭代次数 */
psa_status_t psa_key_derivation_input_key(psa_key_derivation_operation_t *operation,
                                          psa_key_derivation_step_t step,
                                          mbedtls_svc_key_id_t key);
psa_status_t psa_key_derivation_set_capacity(psa_key_derivation_operation_t *operation,
                                             size_t capacity);
psa_status_t psa_key_derivation_get_capacity(const psa_key_derivation_operation_t *operation,
                                             size_t *capacity);
psa_status_t psa_key_derivation_output_bytes(psa_key_derivation_operation_t *operation,
                                             uint8_t *output, size_t output_length);
psa_status_t psa_key_derivation_output_key(const psa_key_attributes_t *attributes,
                                           psa_key_derivation_operation_t *operation,
                                           psa_key_id_t *key);
psa_status_t psa_key_derivation_abort(psa_key_derivation_operation_t *operation);
```

### 5. 派生对称密钥（AES）并用 AEAD 包装

`programs/psa/key_ladder_demo.c` 的 `derive_wrapping_key` 演示从派生密钥再派生一个 AES-128 包装密钥：

```c
psa_key_attributes_t attributes = PSA_KEY_ATTRIBUTES_INIT;
psa_set_key_usage_flags(&attributes, PSA_KEY_USAGE_ENCRYPT);   /* 解包端用 DECRYPT */
psa_set_key_algorithm(&attributes, PSA_ALG_CCM);
psa_set_key_type(&attributes, PSA_KEY_TYPE_AES);
psa_set_key_bits(&attributes, 128);

psa_key_derivation_operation_t operation = PSA_KEY_DERIVATION_OPERATION_INIT;
psa_key_derivation_setup(&operation, KDF_ALG);
psa_key_derivation_input_bytes(&operation, PSA_KEY_DERIVATION_INPUT_SALT,
                               WRAPPING_KEY_SALT, WRAPPING_KEY_SALT_LENGTH);
psa_key_derivation_input_key(&operation, PSA_KEY_DERIVATION_INPUT_SECRET, derived_key);
psa_key_derivation_input_bytes(&operation, PSA_KEY_DERIVATION_INPUT_INFO, NULL, 0);

psa_key_id_t wrapping_key;
psa_key_derivation_output_key(&attributes, &operation, &wrapping_key);
psa_key_derivation_abort(&operation);

/* 之后用 wrapping_key 跑 psa_aead_encrypt（见 recipes/aead.md） */
```

### 6. 密钥协商 + KDF：psa_key_derivation_key_agreement

把 ECDH 结果直接喂入 KDF（比 `psa_raw_key_agreement` 更安全，共享秘密不暴露）：

```c
psa_key_derivation_setup(&op, PSA_ALG_HKDF(PSA_ALG_SHA_256));
psa_key_derivation_input_bytes(&op, PSA_KEY_DERIVATION_INPUT_SALT, salt, salt_len);
psa_key_derivation_key_agreement(&op, PSA_KEY_DERIVATION_INPUT_SECRET,
                                 my_priv, peer_pub, peer_pub_len);
psa_key_derivation_input_bytes(&op, PSA_KEY_DERIVATION_INPUT_INFO, info, info_len);

uint8_t session_key[32];
psa_key_derivation_output_bytes(&op, session_key, sizeof(session_key));
```

### 7. PBKDF2 注意点

- `PSA_ALG_PBKDF2_HMAC(hash)`：口令用 `PSA_KEY_TYPE_PASSWORD`（或 `PASSWORD_HASH`），通过 `psa_key_derivation_input_key(&op, PSA_KEY_DERIVATION_INPUT_SECRET, password_key)`。
- 迭代次数用 `psa_key_derivation_input_integer(&op, PSA_KEY_DERIVATION_INPUT_COST, iterations)`（提供 `input_integer` 的 step 由 PBKDF2 定义）。
- 盐用 `PSA_KEY_DERIVATION_INPUT_SALT`。

## 常见错误

| 错误 | 原因 | 解决方法 |
|---|---|---|
| `PSA_ERROR_INSUFFICIENT_DATA` | 请求输出 > 容量 | 用 `psa_key_derivation_set_capacity(op, PSA_KEY_DERIVATION_UNLIMITED_CAPACITY)`，或 ≥ 输出长度 |
| `PSA_ERROR_INVALID_ARGUMENT` | step 与算法不兼容，或顺序错 | 按算法表提供对应 step；SECRET 用 `input_key` 而非 `input_bytes` |
| `PSA_ERROR_NOT_PERMITTED` | 派生源密钥缺 `PSA_KEY_USAGE_DERIVE` | 在源密钥属性里加 `PSA_KEY_USAGE_DERIVE` |
| `PSA_ERROR_BAD_STATE` | 操作对象未初始化或已输出后又改输入 | 用 `PSA_KEY_DERIVATION_OPERATION_INIT`；所有 input 必须在首次 output 之前完成 |
| 派生出的密钥类型错误 | `psa_key_derivation_output_key` 的 attributes 类型与算法不兼容 | output_key 的 `attributes` 独立设置（类型/位长/用途/算法） |
| 容量耗尽后再读 | 一旦容量为 0，再小输出也失败 | 每次派生用新的 operation 对象 |

## 参考

- `programs/psa/key_ladder_demo.c` — HKDF 密钥阶梯、`output_key` 派生 AES 包装密钥、AEAD wrap/unwrap
- `programs/psa/key_ladder_demo.sh` — 命令行运行示例（generate / save / wrap / unwrap）
- `include/psa/crypto.h` — `psa_key_derivation_setup`、`input_bytes`、`input_integer`、`input_key`、`key_agreement`、`output_bytes`、`output_key`、`set_capacity`、`get_capacity`、`abort` 文档
- `include/psa/crypto_values.h` — `PSA_ALG_HKDF`、`PSA_ALG_HKDF_EXPAND`、`PSA_ALG_PBKDF2_HMAC`、`PSA_ALG_TLS12_PRF`、`PSA_KEY_DERIVATION_INPUT_*`、`PSA_KEY_DERIVATION_UNLIMITED_CAPACITY`
- `include/psa/crypto_sizes.h` — `PSA_BITS_TO_BYTES`、`PSA_BYTES_TO_BITS`
