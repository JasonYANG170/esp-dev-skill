# 解析与校验 X.509 证书

> **适用摘要**: 从内存或文件加载 PEM/DER 证书，输出证书信息，并用受信任 CA 链校验端实体证书（含吊销列表 CRL）。适用于自建 PKI 校验、证书检查工具等。

## 触发意图

- "解析 X.509 证书"
- "校验证书链"
- "打印证书信息"
- "证书是否过期"
- "加载 CRL 吊销列表"

## 前置条件

| 条件 | 要求 |
|---|---|
| 参考示例 | `programs/x509/cert_app.c`、`programs/x509/crl_app.c` |
| 配置宏 | `MBEDTLS_X509_CRT_PARSE_C`、`MBEDTLS_PEM_PARSE_C`；CRL 还需 `MBEDTLS_X509_CRL_PARSE_C`；文件 IO 需 `MBEDTLS_FS_IO` |
| 依赖 | 解析私钥前需 `psa_crypto_init()` |

## 分步说明

### 1. 从内存/文件加载证书链

```c
#include "mbedtls/x509_crt.h"
#include "mbedtls/x509_crl.h"

mbedtls_x509_crt cacert, clicert;
mbedtls_x509_crl crl;
mbedtls_x509_crt_init(&cacert);
mbedtls_x509_crt_init(&clicert);
mbedtls_x509_crl_init(&crl);

/* 从内存 PEM 解析（buf 须以 '\0' 结尾） */
/* 返回 0 成功；>0 表示成功但跳过的证书数；<0 失败 */
ret = mbedtls_x509_crt_parse(&cacert, ca_pem, ca_pem_len);

/* 从文件解析（需 MBEDTLS_FS_IO） */
ret = mbedtls_x509_crt_parse_file(&clicert, "client.crt");

/* 多张证书可对同一 chain 连续 parse，构成链 */
```

### 2. 打印证书信息

```c
char info[4096];
/* 返回写入字节数（不含结尾 '\0'），失败返回负值 */
mbedtls_x509_crt_info(info, sizeof(info), "  ", &clicert);
mbedtls_printf("%s\n", info);

/* 单独取 DN / 序列号 */
mbedtls_x509_dn_gets(info, sizeof(info), &clicert.subject);
mbedtls_x509_serial_gets(info, sizeof(info), &clicert.serial);

/* 过期判断 */
if (mbedtls_x509_time_is_past(&clicert.valid_to))   { /* 已过期 */ }
if (mbedtls_x509_time_is_future(&clicert.valid_from)) { /* 尚未生效 */ }
```

### 3. 用 CA 链校验端实体证书

```c
uint32_t flags = 0;
/* 参数：待校验证书、可信 CA 链、CRL（可空）、CN、flags 输出、回调、入参 */
ret = mbedtls_x509_crt_verify(&clicert, &cacert, &crl, "client.example.com",
                              &flags, NULL, NULL);
if (ret != 0 || flags != 0) {
    char vrfy[512];
    mbedtls_x509_crt_verify_info(vrfy, sizeof(vrfy), "  ! ", flags);
    /* flags 各位指示具体问题（过期、CN 不匹配、用途不符等） */
}

/* 检查密钥用途 */
mbedtls_x509_crt_check_key_usage(&clicert, MBEDTLS_X509_KU_DIGITAL_SIGNATURE);
mbedtls_x509_crt_check_extended_key_usage(&clicert, MBEDTLS_OID_SERVER_AUTH);
```

### 4. 释放

```c
mbedtls_x509_crt_free(&cacert);
mbedtls_x509_crt_free(&clicert);
mbedtls_x509_crl_free(&crl);
```

## 常见错误

| 错误 | 原因 | 解决方法 |
|---|---|---|
| `mbedtls_x509_crt_parse` 返回负值 | PEM 无结尾 `\0` 或格式错 | 传入长度 `strlen(pem)+1`，确认是合法 PEM/DER |
| flags 非零但 ret=0 | 校验通过但有告警（如自签名） | 检查 flags 各位含义 |
| `parse_file` 失败 | 未启用 `MBEDTLS_FS_IO` | 启用文件 IO 或改用内存 parse |
| CN 校验不通过 | 域名与证书 CN/SAN 不符 | 确认传入的 CN 与证书匹配 |
| CRL 未生效 | 未把 crl 传入 verify 第 3 参数 | 把 `&crl` 传入；NULL 表示不检查吊销 |

## 参考

- 示例: `programs/x509/cert_app.c` — 连接服务器并校验证书链
- 示例: `programs/x509/crl_app.c` — 加载并 dump CRL
- 头文件: `include/mbedtls/x509_crt.h`、`include/mbedtls/x509_crl.h`、`include/mbedtls/x509.h`
