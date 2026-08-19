# TLS 客户端

> **适用摘要**: 使用 Mbed TLS 建立 TLS 1.2/1.3 客户端连接，完成握手、读写应用数据并校验服务器证书。适用于 HTTPS、MQTT over TLS 等需要加密通道的客户端场景。

> Version: selected repo/component version; align with the user project and dependency manifest.
> Evidence: `repos/mbedtls/resources/`, source/examples in `repos/mbedtls/`, and this recipe path `repos/mbedtls/recipes/tls_client.md`.
> Validation: draft metadata added from repository routing; verify APIs, Kconfig symbols, and component dependencies against the selected version.

## 触发意图

- "写一个 TLS 客户端"
- "HTTPS 请求"
- "连接 TLS 服务器"
- "SSL/TLS handshake"
- "校验服务器证书"

## 前置条件

| 条件 | 要求 |
|---|---|
| 参考示例 | `programs/ssl/ssl_client1.c` |
| 配置宏 | `MBEDTLS_NET_C`、`MBEDTLS_SSL_CLI_C`、`MBEDTLS_X509_CRT_PARSE_C`、`MBEDTLS_PEM_PARSE_C` |
| 依赖 | 需在首次加密操作前调用 `psa_crypto_init()`（4.x 起 RNG 由 PSA 子系统提供） |

## 分步说明

Mbed TLS 4.x 起的客户端流程与旧版的关键差异：**不再需要**手动创建 `mbedtls_entropy_context` / `mbedtls_ctr_drbg_context`，也**不再调用** `mbedtls_ssl_conf_rng()`。所有需要随机数的操作统一使用 PSA Crypto 提供的全局 RNG，应用只需在入口调用一次 `psa_crypto_init()`。

### 1. 声明并初始化上下文

```c
#include "mbedtls/build_info.h"
#include "mbedtls/platform.h"
#include "mbedtls/net_sockets.h"
#include "mbedtls/ssl.h"
#include "mbedtls/error.h"
#include "mbedtls/debug.h"
#include <string.h>

mbedtls_net_context server_fd;
mbedtls_ssl_context ssl;
mbedtls_ssl_config conf;
mbedtls_x509_crt cacert;

mbedtls_net_init(&server_fd);
mbedtls_ssl_init(&ssl);
mbedtls_ssl_config_init(&conf);
mbedtls_x509_crt_init(&cacert);

/* 4.x 必需：在任何加密操作（含密钥/证书解析、TLS 握手）之前调用 */
psa_status_t status = psa_crypto_init();
if (status != PSA_SUCCESS) {
    /* 出错处理 */
}
```

### 2. 加载 CA 证书并建立 TCP 连接

```c
/* 从内存中的 PEM 数据解析 CA 证书链（buf 须以 '\0' 结尾） */
ret = mbedtls_x509_crt_parse(&cacert,
                             (const unsigned char *) ca_pem,
                             ca_pem_len);
if (ret < 0) { /* ret < 0 表示失败；ret > 0 表示跳过的证书数 */ }

/* 建立底层 TCP 连接 */
ret = mbedtls_net_connect(&server_fd, "example.com", "443",
                          MBEDTLS_NET_PROTO_TCP);
```

### 3. 配置并装载 SSL 上下文

```c
ret = mbedtls_ssl_config_defaults(&conf,
                                  MBEDTLS_SSL_IS_CLIENT,
                                  MBEDTLS_SSL_TRANSPORT_STREAM,
                                  MBEDTLS_SSL_PRESET_DEFAULT);

/* 生产环境应使用 MBEDTLS_SSL_VERIFY_REQUIRED，而非 OPTIONAL */
mbedtls_ssl_conf_authmode(&conf, MBEDTLS_SSL_VERIFY_REQUIRED);
mbedtls_ssl_conf_ca_chain(&conf, &cacert, NULL);
mbedtls_ssl_conf_dbg(&conf, my_debug, stdout);

ret = mbedtls_ssl_setup(&ssl, &conf);

/* 设置 SNI / 主机名校验（客户端强烈建议调用） */
ret = mbedtls_ssl_set_hostname(&ssl, "example.com");

/* 绑定网络 BIO（发送/接收回调） */
mbedtls_ssl_set_bio(&ssl, &server_fd,
                    mbedtls_net_send, mbedtls_net_recv, NULL);
```

### 4. 执行握手（处理非阻塞返回码）

```c
while ((ret = mbedtls_ssl_handshake(&ssl)) != 0) {
    if (ret != MBEDTLS_ERR_SSL_WANT_READ &&
        ret != MBEDTLS_ERR_SSL_WANT_WRITE) {
        /* 真正的握手错误 */
        break;
    }
    /* WANT_READ / WANT_WRITE：底层 I/O 暂时不可用，重试 */
}
```

### 5. 校验证书并收发数据

```c
uint32_t flags = mbedtls_ssl_get_verify_result(&ssl);
if (flags != 0) {
    char vrfy[512];
    mbedtls_x509_crt_verify_info(vrfy, sizeof(vrfy), "  ! ", flags);
    /* flags 非零表示证书校验存在问题（过期、CN 不匹配、链不完整等） */
}

/* 发送 */
while ((ret = mbedtls_ssl_write(&ssl, buf, len)) <= 0) {
    if (ret != MBEDTLS_ERR_SSL_WANT_READ &&
        ret != MBEDTLS_ERR_SSL_WANT_WRITE) { break; }
}

/* 接收：循环读取直到对端关闭 */
do {
    ret = mbedtls_ssl_read(&ssl, buf, sizeof(buf));
    if (ret == MBEDTLS_ERR_SSL_WANT_READ ||
        ret == MBEDTLS_ERR_SSL_WANT_WRITE) { continue; }
    if (ret == MBEDTLS_ERR_SSL_PEER_CLOSE_NOTIFY) { break; }  /* 正常关闭 */
    if (ret <= 0) { break; }
    /* ret > 0：处理 ret 字节应用数据 */
} while (1);

mbedtls_ssl_close_notify(&ssl);
```

### 6. 释放资源

```c
mbedtls_net_free(&server_fd);
mbedtls_x509_crt_free(&cacert);
mbedtls_ssl_free(&ssl);
mbedtls_ssl_config_free(&conf);
mbedtls_psa_crypto_free();
```

## 常见错误

| 错误 | 原因 | 解决方法 |
|---|---|---|
| 握手前崩溃 / `PSA_ERROR_BAD_STATE` | 未调用 `psa_crypto_init()` | 在初始化阶段先调用 `psa_crypto_init()` 并检查返回值 |
| `mbedtls_ssl_conf_rng` 未定义 | 4.x 已移除 RNG 回调 | 删除该调用，改用 PSA 全局 RNG |
| `MBEDTLS_ERR_SSL_FATAL_ALERT_MESSAGE` | SNI/主机名不匹配或证书校验失败 | 确认 `mbedtls_ssl_set_hostname()` 与服务器证书匹配，CA 链完整 |
| `MBEDTLS_ERR_NET_CONN_RESET` | 对端重置连接 | 重新建立连接，服务器侧可能超时 |
| `mbedtls_x509_crt_parse` 返回负值 | PEM 数据未以 `\0` 结尾或格式错误 | 确保传入长度含结尾 `\0`，使用 `strlen(pem)+1` |
| 编译报 `MBEDTLS_NET_C` not defined | 未启用网络模块 | 在 `mbedtls_config.h` 启用 `MBEDTLS_NET_C` |

## 参考

- 示例: `programs/ssl/ssl_client1.c` — 简单 HTTPS 客户端
- 示例: `programs/ssl/ssl_client2.c` — 带完整选项的客户端特性演示
- 头文件: `include/mbedtls/ssl.h`、`include/mbedtls/net_sockets.h`
- 迁移说明: `docs/4.0-migration-guide.md`（RNG / PSA 章节）
