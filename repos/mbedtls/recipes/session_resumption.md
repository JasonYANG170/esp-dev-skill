# 会话恢复与票据（Session Tickets）

> **适用摘要**: 启用 TLS 会话恢复以加速重连：服务端用会话缓存（`ssl_cache`）或会话票据（`ssl_ticket`），客户端用 `get_session`/`set_session` 或序列化保存会话。会话恢复可避免完整握手，显著降低重连延迟与计算开销。

> Version: selected repo/component version; align with the user project and dependency manifest.
> Evidence: `repos/mbedtls/resources/`, source/examples in `repos/mbedtls/`, and this recipe path `repos/mbedtls/recipes/session_resumption.md`.
> Validation: draft metadata added from repository routing; verify APIs, Kconfig symbols, and component dependencies against the selected version.

## 触发意图

- "TLS 会话恢复"
- "session resumption / reconnect"
- "会话票据 session ticket"
- "缓存会话加速重连"
- "DTLS 会话保存"

## 前置条件

| 条件 | 要求 |
|---|---|
| 参考示例 | `programs/ssl/ssl_server2.c`（cache/ticket 选项）、`programs/ssl/ssl_client2.c`（reconnect 选项） |
| 配置宏 | 会话缓存 `MBEDTLS_SSL_CACHE_C`；会话票据 `MBEDTLS_SSL_TICKET_C` |
| 依赖 | 需 `psa_crypto_init()`（票据加密用 RNG/密钥） |

## 分步说明

会话恢复有两条路径：(A) **服务端缓存**——服务端在内存中保存会话，客户端发 Session ID 匹配（仅 TLS 1.2）；(B) **会话票据**——服务端把加密后的会话状态发给客户端，客户端下次带回（TLS 1.2 / TLS 1.3）。客户端侧用 `mbedtls_ssl_get_session()` 取出会话，必要时用 `session_save`/`session_load` 序列化持久化。

### 1. 服务端：启用会话缓存（TLS 1.2）

```c
#include "mbedtls/ssl_cache.h"

mbedtls_ssl_cache_context cache;
mbedtls_ssl_cache_init(&cache);

/* 可选：调整缓存条目数与超时 */
mbedtls_ssl_cache_set_max_entries(&cache, 50);
mbedtls_ssl_cache_set_timeout(&cache, 3600);

/* 在 config_defaults 之后注册 get/set 回调 */
mbedtls_ssl_conf_session_cache(&conf, &cache,
                               mbedtls_ssl_cache_get,
                               mbedtls_ssl_cache_set);

/* 释放：mbedtls_ssl_cache_free(&cache); */
```

### 2. 服务端：启用会话票据（TLS 1.2 / 1.3）

```c
#include "mbedtls/ssl_ticket.h"

mbedtls_ssl_ticket_context ticket_ctx;
mbedtls_ssl_ticket_init(&ticket_ctx);

/* setup 内部生成加密密钥（依赖 PSA RNG） */
psa_crypto_init();
ret = mbedtls_ssl_ticket_setup(&ticket_ctx);   /* 失败返回负值 */

/* 注册票据加密/解密回调 */
mbedtls_ssl_conf_session_tickets_cb(&conf,
                                    mbedtls_ssl_ticket_write,   /* 见 ssl_ticket.h */
                                    mbedtls_ssl_ticket_read,
                                    &ticket_ctx);

/* TLS 1.3：控制是否发送 NewSessionTicket */
mbedtls_ssl_conf_new_session_tickets(&conf, MBEDTLS_SSL_NEW_SESSION_TICKETS_ENABLED);

/* 释放：mbedtls_ssl_ticket_free(&ticket_ctx); */
```

### 3. 客户端：保存并重用会话

```c
mbedtls_ssl_session saved_session;
mbedtls_ssl_session_init(&saved_session);

/* 第一次握手成功后取出会话 */
mbedtls_ssl_get_session(&ssl, &saved_session);

/* （可选）序列化持久化，跨进程/重启复用 */
unsigned char buf[4096]; size_t olen;
mbedtls_ssl_session_save(&saved_session, buf, sizeof(buf), &olen);
/* 下次启动后：mbedtls_ssl_session_load(&session, buf, olen); */

/* 下一次连接前，在 ssl_setup 之后注入会话 */
mbedtls_ssl_set_session(&ssl, &saved_session);
/* 然后正常 handshake —— 若命中会话则走 abbreviated/PSK 握手 */

/* 客户端还需允许使用 ticket */
mbedtls_ssl_conf_session_tickets(&conf, MBEDTLS_SSL_SESSION_TICKETS_ENABLED);
```

> 注意：TLS 1.3 的会话恢复语义与 1.2 不同（基于 PSK / NewSessionTicket）。详见 `docs/architecture/tls13-support.md`。

## 常见错误

| 错误 | 原因 | 解决方法 |
|---|---|---|
| 重连仍走完整握手 | 未注入 `set_session` 或票据回调未注册 | 确认 setup 后调用 `set_session`，服务端注册 ticket 回调 |
| `ssl_cache_*` 未定义 | 未启用 `MBEDTLS_SSL_CACHE_C` | 在配置中启用 |
| 票据 setup 失败 | 未 `psa_crypto_init()` | setup 前初始化 PSA |
| TLS 1.3 无法用 cache | session cache 仅 TLS 1.2 | TLS 1.3 用 NewSessionTicket/PSK |
| 跨重启恢复失败 | 会话仅在内存 | 用 `session_save`/`session_load` 序列化存储 |

## 参考

- 示例: `programs/ssl/ssl_server2.c`、`programs/ssl/ssl_client2.c`
- 头文件: `include/mbedtls/ssl_cache.h`、`include/mbedtls/ssl_ticket.h`、`include/mbedtls/ssl.h`
- TLS 1.3 支持: `docs/architecture/tls13-support.md`
