# TLS 服务器

> **适用摘要**: 使用 Mbed TLS 实现 TLS 服务器（监听端口、装载服务器证书与私钥、接受客户端、握手并收发数据）。适用于 HTTPS 服务端、TLS 设备网关等场景。

## 触发意图

- "写一个 TLS 服务器"
- "HTTPS server"
- "装载服务器证书和私钥"
- "启用会话缓存"
- "TLS 多客户端服务"

## 前置条件

| 条件 | 要求 |
|---|---|
| 参考示例 | `programs/ssl/ssl_server.c`、`programs/ssl/ssl_pthread_server.c` |
| 配置宏 | `MBEDTLS_NET_C`、`MBEDTLS_SSL_SRV_C`、`MBEDTLS_X509_CRT_PARSE_C`、`MBEDTLS_PEM_PARSE_C`；会话缓存还需 `MBEDTLS_SSL_CACHE_C` |
| 依赖 | 需先调用 `psa_crypto_init()` |

## 分步说明

服务器与客户端的主要区别：endpoint 设为 `MBEDTLS_SSL_IS_SERVER`；需通过 `mbedtls_ssl_conf_own_cert()` 装载服务器证书链与私钥；可选用会话缓存（`mbedtls_ssl_cache_*`）加速会话恢复。每服务完一个客户端后，调用 `mbedtls_ssl_session_reset()` 复位上下文以复用同一 `ssl` 结构。

### 1. 装载证书与私钥

```c
#include "mbedtls/ssl.h"
#include "mbedtls/ssl_cache.h"
#include "mbedtls/net_sockets.h"
#include "mbedtls/error.h"

mbedtls_ssl_context ssl;
mbedtls_ssl_config conf;
mbedtls_x509_crt srvcert;
mbedtls_pk_context pkey;
mbedtls_ssl_cache_context cache;

mbedtls_ssl_init(&ssl);
mbedtls_ssl_config_init(&conf);
mbedtls_ssl_cache_init(&cache);
mbedtls_x509_crt_init(&srvcert);
mbedtls_pk_init(&pkey);

psa_crypto_init();   /* 4.x 必需 */

/* 服务器证书（可连续 parse 多张以构成证书链） */
mbedtls_x509_crt_parse(&srvcert, srv_crt_pem, srv_crt_len);
mbedtls_x509_crt_parse(&srvcert, ca_pem, ca_pem_len);

/* 私钥（PEM 或 DER）；带口令的密钥第 4/5 参数提供口令 */
mbedtls_pk_parse_key(&pkey, srv_key_pem, srv_key_len, NULL, 0);
```

### 2. 绑定监听端口并配置

```c
mbedtls_net_context listen_fd, client_fd;
mbedtls_net_init(&listen_fd);
mbedtls_net_init(&client_fd);

mbedtls_net_bind(&listen_fd, NULL, "4433", MBEDTLS_NET_PROTO_TCP);

mbedtls_ssl_config_defaults(&conf,
                            MBEDTLS_SSL_IS_SERVER,
                            MBEDTLS_SSL_TRANSPORT_STREAM,
                            MBEDTLS_SSL_PRESET_DEFAULT);

/* 会话缓存（可选，提升会话恢复性能） */
mbedtls_ssl_conf_session_cache(&conf, &cache,
                               mbedtls_ssl_cache_get,
                               mbedtls_ssl_cache_set);

mbedtls_ssl_conf_ca_chain(&conf, srvcert.next, NULL);   /* 中间证书链 */
mbedtls_ssl_conf_own_cert(&conf, &srvcert, &pkey);
mbedtls_ssl_conf_dbg(&conf, my_debug, stdout);

mbedtls_ssl_setup(&ssl, &conf);
```

### 3. 服务循环（每客户端一次）

```c
while (1) {
    mbedtls_ssl_session_reset(&ssl);          /* 复位以服务新客户端 */

    /* 阻塞等待并接受一个客户端 */
    mbedtls_net_accept(&listen_fd, &client_fd, NULL, 0, NULL);

    mbedtls_ssl_set_bio(&ssl, &client_fd,
                        mbedtls_net_send, mbedtls_net_recv, NULL);

    /* 握手 */
    while ((ret = mbedtls_ssl_handshake(&ssl)) != 0) {
        if (ret != MBEDTLS_ERR_SSL_WANT_READ &&
            ret != MBEDTLS_ERR_SSL_WANT_WRITE) { break; }
    }

    if (ret == 0) {
        /* 读写应用数据：mbedtls_ssl_read / mbedtls_ssl_write */
        do {
            ret = mbedtls_ssl_read(&ssl, buf, sizeof(buf));
        } while (ret == MBEDTLS_ERR_SSL_WANT_READ ||
                 ret == MBEDTLS_ERR_SSL_WANT_WRITE);

        /* ... 处理请求并响应 ... */

        mbedtls_ssl_close_notify(&ssl);
    }

    mbedtls_net_free(&client_fd);   /* 关闭该客户端连接 */
}
```

### 4. 释放资源

```c
mbedtls_net_free(&listen_fd);
mbedtls_x509_crt_free(&srvcert);
mbedtls_pk_free(&pkey);
mbedtls_ssl_free(&ssl);
mbedtls_ssl_config_free(&conf);
mbedtls_ssl_cache_free(&cache);
mbedtls_psa_crypto_free();
```

## 常见错误

| 错误 | 原因 | 解决方法 |
|---|---|---|
| 第二个客户端握手失败 | 未在每次循环 `mbedtls_ssl_session_reset()` | accept 前先 reset ssl 上下文 |
| 证书与私钥不配对 | 装载了错误的密钥 | 用 `mbedtls_pk_check_pair()` 校验公私钥配对 |
| `MBEDTLS_ERR_SSL_CONN_EOF` / 对端关闭 | 客户端提前断开 | read 返回 `<=0` 时跳出并 reset |
| 会话恢复无效 | 未启用 cache 或 cache 容量不足 | 启用 `MBEDTLS_SSL_CACHE_C`，按需 `mbedtls_ssl_cache_set_max_entries()` |
| `mbedtls_pk_parse_key` 失败 | 密钥带口令未提供 | 第 4/5 参数传入口令字符串与长度 |

## 参考

- 示例: `programs/ssl/ssl_server.c` — 单连接 HTTPS 服务器
- 示例: `programs/ssl/ssl_pthread_server.c` — 每客户端一线程
- 示例: `programs/ssl/ssl_server2.c` — 带完整选项的服务器特性演示
- 头文件: `include/mbedtls/ssl.h`、`include/mbedtls/ssl_cache.h`
