# DTLS 服务器（含 HelloVerify Cookie）

> **适用摘要**: 使用 Mbed TLS 实现 DTLS 服务器，绑定 UDP 端口，启用 HelloVerifyRequest cookie 防 DoS、接受客户端、握手并回显数据。DTLS 服务器在 UDP 上易受放大攻击，必须启用 cookie 校验。

## 触发意图

- "DTLS 服务器"
- "UDP TLS 服务端"
- "HelloVerify cookie"
- "DTLS 防 DoS"

## 前置条件

| 条件 | 要求 |
|---|---|
| 参考示例 | `programs/ssl/dtls_server.c` |
| 配置宏 | `MBEDTLS_SSL_PROTO_DTLS`、`MBEDTLS_SSL_SRV_C`、`MBEDTLS_SSL_COOKIE_C`、`MBEDTLS_TIMING_C`、`MBEDTLS_NET_C` |
| 依赖 | 需 `psa_crypto_init()`；需 cookie 上下文；需定时器上下文 |

## 分步说明

DTLS 服务器相比 TLS 服务器多两个必需环节：(1) 通过 `mbedtls_ssl_cookie_setup()` 创建 cookie 上下文并用 `mbedtls_ssl_conf_dtls_cookies()` 注册 cookie 写/校验回调；(2) 每次 accept 后调用 `mbedtls_ssl_set_client_transport_id()` 设置客户端地址（用作 cookie 绑定的标识）。握手可能返回 `MBEDTLS_ERR_SSL_HELLO_VERIFY_REQUIRED`，这表示需要让客户端重发带 cookie 的 ClientHello，应复位并重新 accept。

### 1. 初始化 cookie 上下文

```c
#include "mbedtls/ssl.h"
#include "mbedtls/ssl_cookie.h"
#include "mbedtls/net_sockets.h"
#include "mbedtls/timing.h"

mbedtls_ssl_cookie_ctx cookie_ctx;
mbedtls_ssl_cookie_init(&cookie_ctx);

mbedtls_net_context listen_fd, client_fd;
mbedtls_ssl_context ssl;
mbedtls_ssl_config conf;
mbedtls_x509_crt srvcert;
mbedtls_pk_context pkey;
mbedtls_timing_delay_context timer;

/* init 各上下文 ... */
psa_crypto_init();

/* 装载证书与私钥 ... */
```

### 2. 绑定 UDP 端口并配置 cookie

```c
mbedtls_net_bind(&listen_fd, "::", "4433", MBEDTLS_NET_PROTO_UDP);

mbedtls_ssl_config_defaults(&conf,
                            MBEDTLS_SSL_IS_SERVER,
                            MBEDTLS_SSL_TRANSPORT_DATAGRAM,
                            MBEDTLS_SSL_PRESET_DEFAULT);

mbedtls_ssl_conf_read_timeout(&conf, 10000);
mbedtls_ssl_conf_ca_chain(&conf, srvcert.next, NULL);
mbedtls_ssl_conf_own_cert(&conf, &srvcert, &pkey);

/* 启用 HelloVerify cookie */
mbedtls_ssl_cookie_setup(&cookie_ctx);
mbedtls_ssl_conf_dtls_cookies(&conf,
                              mbedtls_ssl_cookie_write,
                              mbedtls_ssl_cookie_check,
                              &cookie_ctx);

mbedtls_ssl_setup(&ssl, &conf);
mbedtls_ssl_set_timer_cb(&ssl, &timer,
                         mbedtls_timing_set_delay,
                         mbedtls_timing_get_delay);
```

### 3. 服务循环（处理 HelloVerify）

```c
unsigned char client_ip[16];
size_t cliip_len;

reset:
mbedtls_ssl_session_reset(&ssl);

/* accept 并取得客户端地址 */
mbedtls_net_accept(&listen_fd, &client_fd,
                   client_ip, sizeof(client_ip), &cliip_len);

/* 用客户端地址作为 cookie 标识 —— DTLS 服务器必需 */
mbedtls_ssl_set_client_transport_id(&ssl, client_ip, cliip_len);

mbedtls_ssl_set_bio(&ssl, &client_fd,
                    mbedtls_net_send, mbedtls_net_recv,
                    mbedtls_net_recv_timeout);

/* 握手 */
do {
    ret = mbedtls_ssl_handshake(&ssl);
} while (ret == MBEDTLS_ERR_SSL_WANT_READ ||
         ret == MBEDTLS_ERR_SSL_WANT_WRITE);

if (ret == MBEDTLS_ERR_SSL_HELLO_VERIFY_REQUIRED) {
    /* 正常流程：要求客户端重带 cookie 的 ClientHello */
    goto reset;        /* 注意：先 mbedtls_net_free(&client_fd) */
}
/* ... 回显数据：read 后 write 回去 ... */
goto reset;
```

### 4. 释放资源

```c
mbedtls_net_free(&client_fd);
mbedtls_net_free(&listen_fd);
mbedtls_ssl_free(&ssl);
mbedtls_ssl_config_free(&conf);
mbedtls_ssl_cookie_free(&cookie_ctx);
mbedtls_x509_crt_free(&srvcert);
mbedtls_pk_free(&pkey);
mbedtls_psa_crypto_free();
```

## 常见错误

| 错误 | 原因 | 解决方法 |
|---|---|---|
| 未启用 cookie，受放大攻击 | 未调用 `mbedtls_ssl_conf_dtls_cookies` | 生产 DTLS 服务器必须注册 cookie 回调 |
| `HELLO_VERIFY_REQUIRED` 当错误处理 | 这是正常握手中间状态 | reset 后重新 accept |
| 多客户端 cookie 串扰 | 未调用 `set_client_transport_id` | 每次 accept 后设置客户端地址标识 |
| 重置后 BIO 指向已释放 fd | reset 前未释放旧 client_fd | reset 前先 `mbedtls_net_free(&client_fd)` |
| cookie 校验总失败 | cookie 上下文未 `setup` 或跨进程 | 单进程内 `mbedtls_ssl_cookie_setup()` 初始化一次 |

## 参考

- 示例: `programs/ssl/dtls_server.c`
- 头文件: `include/mbedtls/ssl_cookie.h`、`include/mbedtls/ssl.h`
- DTLS cookie 相关 API：`mbedtls_ssl_cookie_setup`、`mbedtls_ssl_conf_dtls_cookies`、`mbedtls_ssl_set_client_transport_id`
