# 加密套件/协议裁剪与错误诊断

> **适用摘要**: 收紧协议版本、加密套件、椭圆曲线、签名算法与证书 profile 以满足安全要求；用 `mbedtls_strerror` / `ssl_get_verify_result` 诊断错误。适用于生产环境加固与排障。

## 触发意图

- "限制 TLS 版本"
- "禁用弱加密套件"
- "配置 ALPN（HTTP2 等）"
- "解读 mbedTLS 错误码"
- "证书校验失败排查"

## 前置条件

| 条件 | 要求 |
|---|---|
| 参考示例 | `programs/ssl/ssl_client2.c`、`programs/ssl/ssl_server2.c`、`programs/util/strerror.c` |
| 配置宏 | 各能力宏需在 `mbedtls_config.h` / `crypto_config.h` 启用 |

## 分步说明

### 1. 限制协议版本

```c
/* 仅允许 TLS 1.2 */
mbedtls_ssl_conf_min_tls_version(&conf, MBEDTLS_SSL_VERSION_TLS1_2);
mbedtls_ssl_conf_max_tls_version(&conf, MBEDTLS_SSL_VERSION_TLS1_2);

/* 仅允许 TLS 1.3 */
mbedtls_ssl_conf_min_tls_version(&conf, MBEDTLS_SSL_VERSION_TLS1_3);
mbedtls_ssl_conf_max_tls_version(&conf, MBEDTLS_SSL_VERSION_TLS1_3);
/* 注意：4.x 已删除旧的 mbedtls_ssl_conf_min_version / max_version */
```

### 2. 指定加密套件列表

```c
/* 仅允许这些套件（按优先级）；NULL 结尾 */
const int ciphersuites[] = {
    MBEDTLS_TLS_ECDHE_ECDSA_WITH_AES_256_GCM_SHA384,
    MBEDTLS_TLS_ECDHE_RSA_WITH_AES_256_GCM_SHA384,
    0
};
mbedtls_ssl_conf_ciphersuites(&conf, ciphersuites);

/* 握手后查询实际协商的套件名 */
const char *name = mbedtls_ssl_get_ciphersuite(&ssl);  /* 例如 "TLS-ECDHE-RSA-WITH-AES-256-GCM-SHA384" */
```

### 3. 限制椭圆曲线 / 签名算法

```c
/* 椭圆曲线列表（4.x 用 conf_groups，旧的 conf_curves 已删除） */
const uint16_t groups[] = { MBEDTLS_SSL_IANA_TLS_GROUP_SECP256R1,
                            MBEDTLS_SSL_IANA_TLS_GROUP_X25519, 0 };
mbedtls_ssl_conf_groups(&conf, groups);

/* 签名算法列表（4.x 用 conf_sig_algs，旧的 conf_sig_hashes 已删除） */
const uint16_t sig_algs[] = { MBEDTLS_SSL_SIG_ECDSA_SECP256R1_SHA256,
                              MBEDTLS_SSL_SIG_RSA_PSS_RSAE_SHA256, 0 };
mbedtls_ssl_conf_sig_algs(&conf, sig_algs);
```

### 4. 证书 profile 与 ALPN

```c
/* 用预设 profile 收紧允许的哈希/密钥/曲线（4.x: default / next / suiteb） */
mbedtls_ssl_conf_cert_profile(&conf, &mbedtls_x509_crt_profile_default);

/* ALPN：例如协商 HTTP/2 */
const char *alpn[] = { "h2", "http/1.1", NULL };
mbedtls_ssl_conf_alpn_protocols(&conf, alpn);
const char *neg = mbedtls_ssl_get_alpn_protocol(&ssl);  /* 协商结果，未达成返回 NULL */
```

### 5. 错误诊断

```c
#include "mbedtls/error.h"

char errbuf[128];
/* mbedTLS 错误码为负（如 -0x7780）；strerror 转为可读字符串 */
mbedtls_strerror(ret, errbuf, sizeof(errbuf));
mbedtls_printf("err %d (-0x%04x): %s\n", ret, (unsigned)-ret, errbuf);

/* 证书校验错误标志（位掩码，0 表示完全通过） */
uint32_t flags = mbedtls_ssl_get_verify_result(&ssl);
if (flags != 0) {
    char vrfy[512];
    mbedtls_x509_crt_verify_info(vrfy, sizeof(vrfy), "  ! ", flags);
}
```

### 关键返回码速查

| 返回码 | 含义 | 处理 |
|---|---|---|
| `MBEDTLS_ERR_SSL_WANT_READ` / `WANT_WRITE` | 非阻塞 I/O 暂不可用 | 重试同一操作 |
| `MBEDTLS_ERR_SSL_PEER_CLOSE_NOTIFY` | 对端正常关闭 | 跳出读循环 |
| `MBEDTLS_ERR_SSL_TIMEOUT` | 读超时（DTLS 常见） | 重传或重连 |
| `MBEDTLS_ERR_SSL_HELLO_VERIFY_REQUIRED` | DTLS 需 cookie 重试 | reset 后重新 accept |
| `MBEDTLS_ERR_SSL_FATAL_ALERT_MESSAGE` | 致命告警 | 检查配置/证书 |
| `MBEDTLS_ERR_NET_CONN_RESET` | TCP/UDP 连接被重置 | 重新连接 |

## 常见错误

| 错误 | 原因 | 解决方法 |
|---|---|---|
| `conf_curves` / `conf_sig_hashes` 未定义 | 4.x 已改名 | 改用 `conf_groups` / `conf_sig_algs` |
| `NO_CIPHERSUITE` | 双方无共同套件 | 检查双方套件/曲线/算法列表有交集 |
| ALPN 协商失败 | 双方 ALPN 列表无交集 | 确认双方都声明了目标协议 |
| 错误码看不出原因 | 未调用 strerror | 用 `mbedtls_strerror` 转可读串 |
| verify flags 非零 | 证书过期/CN 不符/链断 | 用 `verify_info` 输出 flags 详情 |

## 参考

- 示例: `programs/ssl/ssl_client2.c`、`programs/ssl/ssl_server2.c`
- 示例: `programs/util/strerror.c` — 错误码转描述工具
- 头文件: `include/mbedtls/ssl.h`、`include/mbedtls/ssl_ciphersuites.h`、`include/mbedtls/error.h`、`include/mbedtls/x509_crt.h`
- 迁移: `docs/4.0-migration-guide.md`（`conf_curves`→`conf_groups` 等）
