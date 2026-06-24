# 生成证书签名请求（CSR）

> **适用摘要**: 加载私钥，设置主题名、密钥用途、签名算法，生成 DER 或 PEM 格式的 CSR（Certificate Signing Request）。适用于向 CA 申请证书。

## 触发意图

- "生成 CSR"
- "证书签名请求"
- "创建 PKCS#10 CSR"
- "subject_name 设置"

## 前置条件

| 条件 | 要求 |
|---|---|
| 参考示例 | `programs/x509/cert_req.c` |
| 配置宏 | `MBEDTLS_X509_CSR_WRITE_C`、`MBEDTLS_X509_CSR_PARSE_C`、`MBEDTLS_PEM_WRITE_C`、`MBEDTLS_FS_IO` |
| 依赖 | 生成 CSR 需用私钥签名，须先 `psa_crypto_init()` |

## 分步说明

### 1. 加载私钥并初始化 CSR 写入上下文

```c
#include "mbedtls/x509_csr.h"
#include "mbedtls/pk.h"
#include "mbedtls/error.h"

#define MBEDTLS_DECLARE_PRIVATE_IDENTIFIERS
#include "mbedtls/build_info.h"
#include "mbedtls/platform.h"

mbedtls_x509write_csr req;
mbedtls_pk_context key;
mbedtls_x509write_csr_init(&req);
mbedtls_pk_init(&key);

psa_crypto_init();

/* 从文件加载私钥（可带口令） */
ret = mbedtls_pk_parse_keyfile(&key, "subject.key", opt_password);
if (ret != 0) { /* 失败处理 */ }

mbedtls_x509write_csr_set_key(&req, &key);
```

### 2. 设置主题名、算法、用途

```c
/* 主题名格式：CN=...,O=...,C=.. */
ret = mbedtls_x509write_csr_set_subject_name(&req, "CN=Cert,O=mbed TLS,C=UK");

mbedtls_x509write_csr_set_md_alg(&req, MBEDTLS_MD_SHA256);   /* 签名哈希算法 */

/* 可选：密钥用途 / NS 证书类型 / SAN */
mbedtls_x509write_csr_set_key_usage(&req, MBEDTLS_X509_KU_DIGITAL_SIGNATURE);
mbedtls_x509write_csr_set_ns_cert_type(&req, MBEDTLS_X509_NS_CERT_TYPE_SSL_CLIENT);
/* SAN 需在启用 MBEDTLS_X509_SAN_SUPPORT_NOCASETONLY 等相关支持时可用 */
```

### 3. 导出 DER / PEM

```c
unsigned char output_buf[4096];

/* DER 输出：返回写入长度（>0）或负错误码 */
ret = mbedtls_x509write_csr_der(&req, output_buf, sizeof(output_buf));

/* PEM 输出 */
ret = mbedtls_x509write_csr_pem(&req, output_buf, sizeof(output_buf));
if (ret < 0) { /* 失败处理 */ }
```

### 4. 释放

```c
mbedtls_x509write_csr_free(&req);
mbedtls_pk_free(&key);
mbedtls_psa_crypto_free();
```

## 常见错误

| 错误 | 原因 | 解决方法 |
|---|---|---|
| 签名失败 / `PSA_ERROR_BAD_STATE` | 未 `psa_crypto_init()` | 在 `parse_keyfile` 前调用 |
| subject_name 返回负值 | 格式非法（缺逗号、非法字符） | 用 `CN=...,O=...,C=..` 严格格式 |
| `MD_CAN_SHA256` 未定义 | 4.x 改用 PSA 想要的算法宏 | 确认 `PSA_WANT_ALG_SHA_256` 启用 |
| PEM 输出乱码 | buffer 太小被截断 | output_buf 至少 4096 字节 |
| 私钥口令错误 | keyfile 加密但口令不符 | 传入正确口令字符串 |

## 参考

- 示例: `programs/x509/cert_req.c`
- 头文件: `include/mbedtls/x509_csr.h`
- 4.x 起 `set_md_alg` 依赖 PSA 算法宏（`PSA_WANT_ALG_*`）
