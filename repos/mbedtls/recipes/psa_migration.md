# 从 3.x 迁移到 4.x（PSA Crypto）

> **适用摘要**: 将基于 Mbed TLS 3.x 的代码迁移到 4.x：移除手动 RNG（entropy/ctr_drbg）、删除 `mbedtls_ssl_conf_rng` 调用、改用 `psa_crypto_init()`、用 PSA 配置宏替换旧加密宏。这是 4.x 最大的破坏性变更。

## 触发意图

- "迁移到 mbedTLS 4.x"
- "psa_crypto_init 用法"
- "mbedtls_ssl_conf_rng 编译报错"
- "entropy / ctr_drbg 不再需要"
- "MBEDTLS_RSA_C 改成什么"

## 前置条件

| 条件 | 要求 |
|---|---|
| 参考文档 | `docs/4.0-migration-guide.md` |
| 4.x 关键点 | RNG 由 PSA 子系统统一提供；旧加密 API 多已移除 |

## 分步说明

### 1. 删除 RNG 相关代码

```c
// ❌ 3.x 旧写法 —— 4.x 中这些已删除/不再需要
mbedtls_entropy_context entropy;
mbedtls_ctr_drbg_context ctr_drbg;
mbedtls_entropy_init(&entropy);
mbedtls_ctr_drbg_init(&ctr_drbg);
mbedtls_ctr_drbg_seed(&ctr_drbg, mbedtls_entropy_func, &entropy, ...);
mbedtls_ssl_conf_rng(&conf, mbedtls_ctr_drbg_random, &ctr_drbg);   // 4.x 已删除！
```

```c
// ✅ 4.x 正确写法
psa_status_t status = psa_crypto_init();   // 全局唯一一次
if (status != PSA_SUCCESS) { /* 失败处理 */ }
// 不再调用 mbedtls_ssl_conf_rng；TLS/X.509 自动使用 PSA 全局 RNG
```

> 关键规则：`psa_crypto_init()` 必须在任何加密操作之前调用——包括解析密钥、解析证书、启动 TLS 握手。

### 2. 函数原型变化（去掉 f_rng / p_rng）

```c
// ❌ 3.x：X.509 / SSL 写入函数带 f_rng, p_rng 参数
mbedtls_pk_parse_key(&key, data, len, f_rng, p_rng);
mbedtls_x509write_crt_pem(&crt, buf, size, f_rng, p_rng);

// ✅ 4.x：这些函数不再接收 RNG 参数
mbedtls_pk_parse_key(&key, data, len, NULL, 0);          // 后两参变成口令 + 口令长度
mbedtls_x509write_crt_pem(&crt, buf, size);              // 无 RNG 参数
mbedtls_x509write_csr_pem(&req, buf, size);
```

### 3. 配置宏迁移（加密能力改用 PSA_WANT_*）

```c
// ❌ 3.x：加密能力用 MBEDTLS_*_C 宏
#define MBEDTLS_RSA_C
#define MBEDTLS_AES_C
#define MBEDTLS_SHA256_C
#define MBEDTLS_PKCS1_V15

// ✅ 4.x：加密能力由 TF-PSA-Crypto 的 PSA_WANT_* 控制
//   在 tf-psa-crypto/include/psa/crypto_config.h 中启用：
#define PSA_WANT_ALG_RSA_PKCS1V15_SIGN
#define PSA_WANT_ALG_SHA_256
#define PSA_WANT_KEY_TYPE_RSA_KEY_PAIR
//   X.509/TLS 自身的配置仍在 include/mbedtls/mbedtls_config.h
```

> TLS/X.509 的配置（`MBEDTLS_SSL_*`、`MBEDTLS_X509_*`）不受影响，仍在 `mbedtls_config.h`。

### 4. 版本号 API 变化

```c
// ❌ 3.x：已删除
mbedtls_ssl_conf_min_version(&conf, MBEDTLS_SSL_MAJOR_VERSION_3, MBEDTLS_SSL_MINOR_VERSION_3);

// ✅ 4.x
mbedtls_ssl_conf_min_tls_version(&conf, MBEDTLS_SSL_VERSION_TLS1_2);
mbedtls_ssl_conf_max_tls_version(&conf, MBEDTLS_SSL_VERSION_TLS1_3);
```

### 5. 其它已删除/改名的项

- `mbedtls_ssl_conf_curves()` → 用 `mbedtls_ssl_conf_groups()`
- `mbedtls_ssl_conf_sig_hashes()` → 用 `mbedtls_ssl_conf_sig_algs()`
- `mbedtls_x509write_crt_set_serial()` → 用 `mbedtls_x509write_crt_set_serial_raw()`
- `compat-2.x.h` 头文件已删除
- (D)TLS 1.2 中的 DHE 密钥交换已移除（仅保留 ECDHE）

## 常见错误

| 错误 | 原因 | 解决方法 |
|---|---|---|
| `mbedtls_ssl_conf_rng` 未定义 | 4.x 已删除 | 删除该调用 |
| `mbedtls_entropy_*` / `mbedtls_ctr_drbg_*` 未定义 | 4.x 已移除手动 RNG | 用 `psa_crypto_init()` |
| 编译报 `MBEDTLS_RSA_C` 无效 | 加密宏移至 PSA | 改用 `PSA_WANT_*` |
| `PSA_ERROR_BAD_STATE` | 未调用 `psa_crypto_init()` | 在程序入口先调用 |
| `conf_min_version` 未定义 | 4.x 删除了旧版本 API | 改用 `conf_min_tls_version` |

## 参考

- 文档: `docs/4.0-migration-guide.md`（CMake、仓库拆分、PSA、RNG、函数原型、已删除特性）
- 头文件: `include/mbedtls/ssl.h`（`mbedtls_ssl_conf_min/max_tls_version`、`conf_groups`、`conf_sig_algs`）
- TF-PSA-Crypto: `tf-psa-crypto/include/psa/crypto.h`（`psa_crypto_init`）
