# 签发 X.509 证书

> **适用摘要**: 加载签发者（CA）私钥与证书、主体公钥或 CSR，设置版本/序列号/有效期/主题/扩展，签发 DER 或 PEM 证书。适用于自建 CA 签发端实体证书。

> Evidence: `repos/mbedtls/resources/`, source/examples in `repos/mbedtls/`, and this recipe path `repos/mbedtls/recipes/cert_signing.md`.
> Validation: draft metadata added from repository routing; verify APIs, Kconfig symbols, and component dependencies against the selected version.

## 触发意图

- "签发证书"
- "自签名证书"
- "用 CA 签发证书"
- "设置 basic constraints / key usage"

## 前置条件

| 条件 | 要求 |
|---|---|
| 参考示例 | `programs/x509/cert_write.c` |
| 配置宏 | `MBEDTLS_X509_CRT_WRITE_C`、`MBEDTLS_X509_CRT_PARSE_C`、`MBEDTLS_PEM_WRITE_C`、`MBEDTLS_FS_IO`、`MBEDTLS_MD_C`、`PSA_WANT_ALG_SHA_256` |
| 依赖 | 签名需私钥，须先 `psa_crypto_init()` |

## 分步说明

### 1. 加载签发者与主体密钥并初始化写入上下文

```c
#define MBEDTLS_DECLARE_PRIVATE_IDENTIFIERS
#include "mbedtls/build_info.h"
#include "mbedtls/platform.h"
#include "mbedtls/x509_crt.h"
#include "mbedtls/pk.h"

mbedtls_x509write_crt crt;
mbedtls_pk_context issuer_key, subject_key;
mbedtls_x509write_crt_init(&crt);
mbedtls_pk_init(&issuer_key);
mbedtls_pk_init(&subject_key);

psa_crypto_init();

mbedtls_pk_parse_keyfile(&issuer_key, "issuer.key", NULL);
mbedtls_pk_parse_keyfile(&subject_key, "subject.key", NULL);

mbedtls_x509write_crt_set_issuer_key(&crt, &issuer_key);
mbedtls_x509write_crt_set_subject_key(&crt, &subject_key);
```

### 2. 设置证书字段

```c
mbedtls_x509write_crt_set_version(&crt, MBEDTLS_X509_CRT_VERSION_3);
mbedtls_x509write_crt_set_md_alg(&crt, MBEDTLS_MD_SHA256);

/* 序列号（4.x 用 _raw，旧的 set_serial 已删除） */
mbedtls_x509write_crt_set_serial_raw(&crt, serial_buf, serial_len);

/* 有效期：格式 "200101000000Z" - "20301231235959Z" */
mbedtls_x509write_crt_set_validity(&crt, "20230101000000Z", "20301231235959Z");

/* 签发者 / 主体 DN */
mbedtls_x509write_crt_set_issuer_name(&crt, "CN=My CA,O=Org,C=CN");
mbedtls_x509write_crt_set_subject_name(&crt, "CN=device-001,O=Org,C=CN");
```

### 3. 设置扩展

```c
/* 基本约束：是否为 CA 及路径深度 */
mbedtls_x509write_crt_set_basic_constraints(&crt, opt_is_ca, opt_pathlen);

/* SKI / AKI（强烈建议） */
mbedtls_x509write_crt_set_subject_key_identifier(&crt);
mbedtls_x509write_crt_set_authority_key_identifier(&crt);

/* 密钥用途 / 扩展密钥用途 / NS 类型 */
mbedtls_x509write_crt_set_key_usage(&crt, MBEDTLS_X509_KU_DIGITAL_SIGNATURE);
mbedtls_x509write_crt_set_ext_key_usage(&crt, opt_ext_key_usage);
mbedtls_x509write_crt_set_ns_cert_type(&crt, MBEDTLS_X509_NS_CERT_TYPE_SSL_SERVER);
```

### 4. 导出 DER / PEM

```c
unsigned char output_buf[4096];
ret = mbedtls_x509write_crt_der(&crt, output_buf, sizeof(output_buf));  /* 返回长度 */
ret = mbedtls_x509write_crt_pem(&crt, output_buf, sizeof(output_buf));  /* 返回 0 成功 */
```

### 5. 释放

```c
mbedtls_x509write_crt_free(&crt);
mbedtls_pk_free(&issuer_key);
mbedtls_pk_free(&subject_key);
mbedtls_psa_crypto_free();
```

## 常见错误

| 错误 | 原因 | 解决方法 |
|---|---|---|
| `mbedtls_x509write_crt_set_serial` 未定义 | 4.x 已删除该旧函数 | 改用 `mbedtls_x509write_crt_set_serial_raw()` |
| 签名失败 `BAD_STATE` | 未 `psa_crypto_init()` | 初始化 PSA 后再调用 |
| 自签证书主体=签发者 | 自签时 issuer/subject 密钥应一致 | 自签用同一密钥对 |
| `ext_key_usage` 报错 | OID 字符串格式错 | 用 `MBEDTLS_OID_*` 或合法点分 OID 字符串 |
| DER 返回值理解错误 | DER 返回写入长度（>0），非 0 才成功 | `ret > 0` 为成功 |

## 参考

- 示例: `programs/x509/cert_write.c`
- 头文件: `include/mbedtls/x509_crt.h`
- 4.x 迁移：`set_serial` → `set_serial_raw`（见 `docs/4.0-migration-guide.md`）
