# DTLS 客户端

> **适用摘要**: 使用 Mbed TLS 建立 DTLS（基于 UDP 的 TLS）客户端，完成握手并收发数据报。适用于物联网设备、低功耗 UDP 加密通信等场景。

## 触发意图

- "DTLS 客户端"
- "UDP 加密通信"
- "datagram TLS"
- "DTLS 握手"

## 前置条件

| 条件 | 要求 |
|---|---|
| 参考示例 | `programs/ssl/dtls_client.c` |
| 配置宏 | `MBEDTLS_SSL_PROTO_DTLS`、`MBEDTLS_SSL_CLI_C`、`MBEDTLS_NET_C`、`MBEDTLS_TIMING_C`、`MBEDTLS_X509_CRT_PARSE_C` |
| 依赖 | 需绑定定时器回调（`mbedtls_timing_set_delay` / `mbedtls_timing_get_delay`），需提供 `recv_timeout` BIO |

## 分步说明

DTLS 与 TLS 的关键区别：transport 设为 `MBEDTLS_SSL_TRANSPORT_DATAGRAM`；BIO 必须同时提供发送、接收和**带超时的接收**回调（`mbedtls_net_recv_timeout`）；必须通过 `mbedtls_ssl_set_timer_cb()` 绑定定时器回调以驱动 DTLS 重传状态机；应通过 `mbedtls_ssl_conf_read_timeout()` 设置读超时。

### 1. 初始化（含定时器上下文）

```c
#include "mbedtls/ssl.h"
#include "mbedtls/net_sockets.h"
#include "mbedtls/timing.h"
#include "mbedtls/error.h"

mbedtls_ssl_context ssl;
mbedtls_ssl_config conf;
mbedtls_x509_crt cacert;
mbedtls_net_context server_fd;
mbedtls_timing_delay_context timer;   /* DTLS 必需的定时器上下文 */

mbedtls_ssl_init(&ssl);
mbedtls_ssl_config_init(&conf);
mbedtls_x509_crt_init(&cacert);
mbedtls_net_init(&server_fd);

psa_crypto_init();

mbedtls_x509_crt_parse(&cacert, ca_pem, ca_pem_len);
```

### 2. 建立 UDP 连接并配置

```c
#define READ_TIMEOUT_MS 1000

mbedtls_net_connect(&server_fd, SERVER_ADDR, "4433",
                    MBEDTLS_NET_PROTO_UDP);

mbedtls_ssl_config_defaults(&conf,
                            MBEDTLS_SSL_IS_CLIENT,
                            MBEDTLS_SSL_TRANSPORT_DATAGRAM,
                            MBEDTLS_SSL_PRESET_DEFAULT);

mbedtls_ssl_conf_authmode(&conf, MBEDTLS_SSL_VERIFY_REQUIRED);
mbedtls_ssl_conf_ca_chain(&conf, &cacert, NULL);
mbedtls_ssl_conf_read_timeout(&conf, READ_TIMEOUT_MS);

mbedtls_ssl_setup(&ssl, &conf);
mbedtls_ssl_set_hostname(&ssl, SERVER_NAME);

/* DTLS 三参数 BIO：send / recv / recv_timeout */
mbedtls_ssl_set_bio(&ssl, &server_fd,
                    mbedtls_net_send, mbedtls_net_recv,
                    mbedtls_net_recv_timeout);

/* 绑定定时器回调 —— DTLS 重传依赖它 */
mbedtls_ssl_set_timer_cb(&ssl, &timer,
                         mbedtls_timing_set_delay,
                         mbedtls_timing_get_delay);
```

### 3. 握手（DTLS 风格的重试循环）

```c
do {
    ret = mbedtls_ssl_handshake(&ssl);
} while (ret == MBEDTLS_ERR_SSL_WANT_READ ||
         ret == MBEDTLS_ERR_SSL_WANT_WRITE);

if (ret != 0) { /* 握手失败处理 */ }
```

### 4. 收发数据报（处理超时与重试）

```c
int retry_left = 5;

send_request:
do {
    ret = mbedtls_ssl_write(&ssl, (unsigned char *)MESSAGE, sizeof(MESSAGE) - 1);
} while (ret == MBEDTLS_ERR_SSL_WANT_READ ||
         ret == MBEDTLS_ERR_SSL_WANT_WRITE);

do {
    ret = mbedtls_ssl_read(&ssl, buf, sizeof(buf) - 1);
} while (ret == MBEDTLS_ERR_SSL_WANT_READ ||
         ret == MBEDTLS_ERR_SSL_WANT_WRITE);

if (ret == MBEDTLS_ERR_SSL_TIMEOUT) {
    /* 读超时：DTLS 允许重传请求 */
    if (retry_left-- > 0) { goto send_request; }
}
```

### 5. 关闭与释放

```c
do {
    ret = mbedtls_ssl_close_notify(&ssl);
} while (ret == MBEDTLS_ERR_SSL_WANT_WRITE);

mbedtls_net_free(&server_fd);
mbedtls_x509_crt_free(&cacert);
mbedtls_ssl_free(&ssl);
mbedtls_ssl_config_free(&conf);
mbedtls_psa_crypto_free();
```

## 常见错误

| 错误 | 原因 | 解决方法 |
|---|---|---|
| 握手卡死 / 无重传 | 未绑定定时器回调 | 必须调用 `mbedtls_ssl_set_timer_cb()` |
| `MBEDTLS_ERR_SSL_TIMEOUT` | 数据报丢包或服务器无响应 | 调大 `mbedtls_ssl_conf_read_timeout()` 或实现重试 |
| BIO 缺少 `recv_timeout` | 只传了 send/recv 两参数 | DTLS 需提供第三参数 `mbedtls_net_recv_timeout` |
| 握手失败 `-0x7780` 之外异常 | 未启用 `MBEDTLS_TIMING_C` | 在配置中启用 timing 模块 |
| 链接报未定义 `mbedtls_timing_*` | 未链接 timing 模块 | 确认编译含 `mbedtls/timing.c` |

## 参考

- 示例: `programs/ssl/dtls_client.c`
- 头文件: `include/mbedtls/ssl.h`、`include/mbedtls/timing.h`、`include/mbedtls/net_sockets.h`
- DTLS 超时/重传机制见 `mbedtls_ssl_conf_read_timeout`、`mbedtls_ssl_conf_handshake_timeout` 文档
