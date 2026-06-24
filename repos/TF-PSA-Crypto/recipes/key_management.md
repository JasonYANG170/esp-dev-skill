# 密钥管理（生成 / 导入 / 导出 / 销毁）

> **适用摘要**: 使用 `psa_key_attributes_t` 描述密钥，通过 `psa_generate_key` / `psa_import_key` 创建密钥，`psa_export_key` / `psa_export_public_key` 取出密钥，`psa_destroy_key` 释放。涵盖易失(volatile)与持久(persistent)两种生命周期。

## 触发意图

- "如何创建一个 AES 密钥"
- "psa_generate_key 怎么用"
- "怎么导入已有的密钥字节"
- "PSA 持久化密钥 / persistent key"
- "导出公钥 psa_export_public_key"
- "psa_set_key_usage_flags / algorithm / bits / type"

## 前置条件

| 条件 | 要求 |
|---|---|
| 库状态 | 已成功调用 `psa_crypto_init()` |
| 头文件 | `#include <psa/crypto.h>` |
| 配置项 | 用到的密钥类型与算法在 `crypto_config.h` 中以 `PSA_WANT_*` 启用 |
| 参考示例 | `programs/psa/crypto_examples.c`、`programs/psa/key_ladder_demo.c` |

## 分步说明

### 1. 设置属性对象

密钥通过 `psa_key_attributes_t` 描述。初始化用 `PSA_KEY_ATTRIBUTES_INIT`，再用 `psa_set_key_*` 填充：

```c
psa_key_attributes_t attributes = PSA_KEY_ATTRIBUTES_INIT;

psa_set_key_type(&attributes, PSA_KEY_TYPE_AES);          /* 密钥类型 */
psa_set_key_bits(&attributes, 256);                       /* 密钥长度(位) */
psa_set_key_usage_flags(&attributes,
                        PSA_KEY_USAGE_ENCRYPT | PSA_KEY_USAGE_DECRYPT);
psa_set_key_algorithm(&attributes, PSA_ALG_CBC_PKCS7);    /* 允许的算法 */
```

四个 setter 均为 `static inline`（定义在 `include/psa/crypto_struct.h`），签名：

```c
void psa_set_key_id(psa_key_attributes_t *attributes, mbedtls_svc_key_id_t key);
void psa_set_key_lifetime(psa_key_attributes_t *attributes, psa_key_lifetime_t lifetime);
void psa_set_key_usage_flags(psa_key_attributes_t *attributes, psa_key_usage_t usage_flags);
void psa_set_key_algorithm(psa_key_attributes_t *attributes, psa_algorithm_t alg);
void psa_set_key_type(psa_key_attributes_t *attributes, psa_key_type_t type);
void psa_set_key_bits(psa_key_attributes_t *attributes, size_t bits);
```

属性对象用完后可调用 `psa_reset_key_attributes(&attributes)` 清空（可选）。

### 2. 生成随机密钥：psa_generate_key

库内部 RNG 生成（无需传 RNG）。来自 `programs/psa/crypto_examples.c` 的写法：

```c
psa_key_attributes_t attributes = PSA_KEY_ATTRIBUTES_INIT;
psa_key_id_t key = 0;

psa_set_key_usage_flags(&attributes,
                        PSA_KEY_USAGE_ENCRYPT | PSA_KEY_USAGE_DECRYPT);
psa_set_key_algorithm(&attributes, PSA_ALG_CBC_NO_PADDING);
psa_set_key_type(&attributes, PSA_KEY_TYPE_AES);
psa_set_key_bits(&attributes, 256);

psa_status_t status = psa_generate_key(&attributes, &key);
if (status != PSA_SUCCESS) { /* 处理错误 */ }
```

签名：

```c
psa_status_t psa_generate_key(const psa_key_attributes_t *attributes,
                              psa_key_id_t *key);
```

### 3. 导入已有密钥字节：psa_import_key

把现成的密钥字节注册为一个 PSA 密钥对象（来自 `programs/psa/hmac_demo.c`）：

```c
const unsigned char key_bytes[32] = { /* 32 字节 HMAC 密钥 */ };
psa_key_id_t key = 0;

psa_key_attributes_t attributes = PSA_KEY_ATTRIBUTES_INIT;
psa_set_key_usage_flags(&attributes, PSA_KEY_USAGE_SIGN_MESSAGE);
psa_set_key_algorithm(&attributes, PSA_ALG_HMAC(PSA_ALG_SHA_256));
psa_set_key_type(&attributes, PSA_KEY_TYPE_HMAC);
psa_set_key_bits(&attributes, 8 * sizeof(key_bytes));     /* 可选但推荐 */

psa_status_t status = psa_import_key(&attributes,
                                     key_bytes, sizeof(key_bytes), &key);
```

签名：

```c
psa_status_t psa_import_key(const psa_key_attributes_t *attributes,
                            const uint8_t *data,
                            size_t data_length,
                            psa_key_id_t *key);
```

### 4. 导出密钥 / 公钥：psa_export_key / psa_export_public_key

导出需要 `PSA_KEY_USAGE_EXPORT` 标志。输出缓冲区大小用宏计算：

```c
psa_key_attributes_t attr = PSA_KEY_ATTRIBUTES_INIT;
psa_get_key_attributes(key, &attr);
psa_key_type_t type = psa_get_key_type(&attr);
size_t bits = psa_get_key_bits(&attr);

uint8_t buf[PSA_EXPORT_KEY_OUTPUT_SIZE(type, bits)];
size_t out_len = 0;
psa_export_key(key, buf, sizeof(buf), &out_len);
```

对于非对称密钥对，通常只需要导出**公钥**（不需要 `EXPORT` 标志即可导出公钥）：

```c
uint8_t pub[PSA_EXPORT_PUBLIC_KEY_MAX_SIZE];   /* 通用上限宏 */
size_t pub_len = 0;
psa_export_public_key(key, pub, sizeof(pub), &pub_len);
```

签名：

```c
psa_status_t psa_export_key(mbedtls_svc_key_id_t key,
                            uint8_t *data, size_t data_size,
                            size_t *data_length);
psa_status_t psa_export_public_key(mbedtls_svc_key_id_t key,
                                   uint8_t *data, size_t data_size,
                                   size_t *data_length);
```

### 5. 持久(persistent)密钥

默认创建的是易失密钥（`PSA_KEY_LIFETIME_VOLATILE == 0`），随 `mbedtls_psa_crypto_free()` 或进程结束而消失。要创建持久密钥，设置一个稳定的 id 并把 lifetime 设为 `PSA_KEY_LIFETIME_PERSISTENT`：

```c
#define MY_DEVICE_KEY_ID  ((psa_key_id_t)1)   /* 范围 [PSA_KEY_ID_USER_MIN, PSA_KEY_ID_USER_MAX] */

psa_key_attributes_t attributes = PSA_KEY_ATTRIBUTES_INIT;
psa_set_key_id(&attributes, MY_DEVICE_KEY_ID);                    /* 声明持久 */
psa_set_key_lifetime(&attributes, PSA_KEY_LIFETIME_PERSISTENT);   /* 写入存储 */
psa_set_key_type(&attributes, PSA_KEY_TYPE_AES);
psa_set_key_bits(&attributes, 128);
psa_set_key_usage_flags(&attributes, PSA_KEY_USAGE_ENCRYPT | PSA_KEY_USAGE_DECRYPT);
psa_set_key_algorithm(&attributes, PSA_ALG_GCM);

psa_key_id_t key;
psa_generate_key(&attributes, &key);   /* 重启后用同一个 id 即可重新引用 */
```

`PSA_KEY_ID_USER_MIN == 0x00000001`，`PSA_KEY_ID_USER_MAX == 0x3fffffff`，`PSA_KEY_ID_NULL == 0`。

### 6. 销毁密钥：psa_destroy_key

```c
psa_status_t psa_destroy_key(mbedtls_svc_key_id_t key);
```

无论易失还是持久，`psa_destroy_key` 都会释放该密钥（持久的会从存储中删除）。`psa_destroy_key(PSA_KEY_ID_NULL)` 是合法的 no-op，保证返回 `PSA_SUCCESS`，因此清理代码里可直接调用：

```c
exit:
    psa_destroy_key(key);   /* key==0 时无副作用 */
```

### 7. 读取已建密钥的属性：psa_get_key_attributes

```c
psa_key_attributes_t attr = PSA_KEY_ATTRIBUTES_INIT;
psa_get_key_attributes(key, &attr);
psa_key_type_t t  = psa_get_key_type(&attr);
size_t          b = psa_get_key_bits(&attr);
psa_algorithm_t a = psa_get_key_algorithm(&attr);
psa_key_usage_t u = psa_get_key_usage_flags(&attr);
psa_reset_key_attributes(&attr);
```

`programs/psa/aead_demo.c` 中的 `aead_info()` 函数演示了这种用法（用 `PSA_AEAD_TAG_LENGTH` 反推 tag 长度）。

## 常见错误

| 错误 | 原因 | 解决方法 |
|---|---|---|
| `PSA_ERROR_NOT_SUPPORTED` | 该密钥类型/算法未在 `crypto_config.h` 用 `PSA_WANT_*` 启用 | 启用对应 `PSA_WANT_KEY_TYPE_*` / `PSA_WANT_ALG_*`，重新编译 |
| `PSA_ERROR_NOT_PERMITTED` | 后续操作需要的 usage flag 没设 | 在 `psa_set_key_usage_flags` 中补上（如要解密需 `PSA_KEY_USAGE_DECRYPT`） |
| `PSA_ERROR_INVALID_ARGUMENT` | `psa_import_key` 的字节长度与 `psa_set_key_bits` 不匹配，或算法与类型不兼容 | 保证 `8 * data_length == bits`（结构化密钥除外）；核对类型↔算法兼容表 |
| `PSA_ERROR_BUFFER_TOO_SMALL` | 导出缓冲区太小 | 用 `PSA_EXPORT_KEY_OUTPUT_SIZE(type, bits)` 或 `PSA_EXPORT_PUBLIC_KEY_MAX_SIZE` 定尺寸 |
| `PSA_ERROR_ALREADY_EXISTS` | 持久密钥 id 已被占用且属性不同 | 先 `psa_destroy_key(id)` 再创建，或换一个 id |
| 持久密钥重启后取不到 | 没在创建时设置 id + `PSA_KEY_LIFETIME_PERSISTENT` | 用 `psa_set_key_id` + `psa_set_key_lifetime` |
| 内存/槽位泄漏 | 用完没调用 `psa_destroy_key` | 在 `exit:` 标签里统一销毁 |

## 参考

- `programs/psa/crypto_examples.c` — `psa_generate_key` 生成 AES 密钥
- `programs/psa/hmac_demo.c` — `psa_import_key` 导入 HMAC 密钥
- `programs/psa/key_ladder_demo.c` — `psa_export_key` 导出派生密钥到文件
- `programs/psa/aead_demo.c` — `psa_get_key_attributes` / `psa_get_key_type` / `psa_get_key_bits` 读取属性
- `include/psa/crypto.h` — `psa_import_key`、`psa_generate_key`、`psa_export_key`、`psa_export_public_key`、`psa_destroy_key`、`psa_copy_key`、`psa_purge_key` 文档
- `include/psa/crypto_values.h` — `PSA_KEY_TYPE_*`、`PSA_KEY_USAGE_*`、`PSA_KEY_LIFETIME_*`、`PSA_KEY_ID_*` 定义
- `include/psa/crypto_struct.h` — `psa_set_key_*` / `psa_get_key_*` 内联函数
