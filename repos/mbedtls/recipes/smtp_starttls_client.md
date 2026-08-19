# SMTP over TLS / STARTTLS 升级（ssl_mail_client）

> **适用摘要**: 用同一底层 TCP 连接先跑明文 SMTP（读 banner、发 `EHLO`、发 `STARTTLS`），收到 `220` 后再把该 socket 就地升级为 TLS——即 STARTTLS 机会式加密模式。升级后在 TLS 通道内继续 SMTP 会话（`AUTH LOGIN` base64 鉴权、`MAIL FROM`/`RCPT TO`/`DATA`）。也可选用 mode=0 直接在连接时即上 TLS（SMTPS，端口 465）。该 STARTTLS 模式可迁移到 IMAP/POP3/LDAP/Postgres 等同类协议。

> Version: selected repo/component version; align with the user project and dependency manifest.
> Evidence: `repos/mbedtls/resources/`, source/examples in `repos/mbedtls/`, and this recipe path `repos/mbedtls/recipes/smtp_starttls_client.md`.
> Validation: draft metadata added from repository routing; verify APIs, Kconfig symbols, and component dependencies against the selected version.

## 触发意图

- "SMTP over TLS"
- "STARTTLS 升级"
- "明文连接升级到 TLS"
- "邮件客户端 TLS"
- "ssl_mail_client"

## 前置条件

| 条件 | 要求 |
|---|---|
| 参考示例 | `programs/ssl/ssl_mail_client.c` |
| 配置宏 | `MBEDTLS_NET_C`、`MBEDTLS_SSL_CLI_C`、`MBEDTLS_SSL_TLS_C`、`MBEDTLS_X509_CRT_PARSE_C`、`MBEDTLS_PEM_PARSE_C`；示例还要求 `MBEDTLS_FS_IO`（读 CA 文件）；base64 鉴权需 `MBEDTLS_BASE64_C` |
| 依赖 | `psa_crypto_init()` |
| 平台注意 | 示例为 POSIX 倾向（用到 `gethostname`、`unistd.h`、文件 IO）；STARTTLS 升级 socket 的模式本身可移植 |

## 分步说明

STARTTLS 与"连接即 TLS"的关键区别：**先用普通 socket 跑明文协议握手，协商出 TLS 后再把同一个 `mbedtls_net_context` 绑定到 SSL 上下文做 TLS 握手**。明文阶段直接用 `mbedtls_net_send/recv` 读写 socket；TLS 阶段通过 `mbedtls_ssl_set_bio` 把同一个 socket 交给 SSL 层。

### 1. 建立 TCP 连接并装载 CA（与普通 TLS 客户端一致）

```c
#include "mbedtls/net_sockets.h"
#include "mbedtls/ssl.h"
#include "mbedtls/base64.h"

mbedtls_net_context server_fd;
mbedtls_ssl_context ssl;
mbedtls_ssl_config conf;
mbedtls_x509_crt cacert;

mbedtls_net_init(&server_fd);
mbedtls_ssl_init(&ssl);
mbedtls_ssl_config_init(&conf);
mbedtls_x509_crt_init(&cacert);

psa_crypto_init();

mbedtls_x509_crt_parse_file(&cacert, "/etc/ssl/certs/ca-bundle.crt");

/* 仅建立 TCP 连接，此时还未上 TLS */
mbedtls_net_connect(&server_fd, "smtp.example.com", "587",
                    MBEDTLS_NET_PROTO_TCP);

/* 准备 SSL 配置（先不绑定 BIO） */
mbedtls_ssl_config_defaults(&conf, MBEDTLS_SSL_IS_CLIENT,
                            MBEDTLS_SSL_TRANSPORT_STREAM,
                            MBEDTLS_SSL_PRESET_DEFAULT);
mbedtls_ssl_conf_ca_chain(&conf, &cacert, NULL);
mbedtls_ssl_conf_dbg(&conf, my_debug, stdout);
mbedtls_ssl_setup(&ssl, &conf);
mbedtls_ssl_set_hostname(&ssl, "smtp.example.com");
```

### 2. 明文阶段：读 banner、EHLO、STARTTLS

此阶段直接用 `mbedtls_net_send` / `mbedtls_net_recv` 在原始 socket 上读写 SMTP 文本，并按行解析三位状态码。

```c
/* 复用 ssl_mail_client.c 的 write_and_get_response：在原始 socket 上发文本并读 SMTP 状态码 */
/* 读服务器 banner（期望 220） */
int code = write_and_get_response(&server_fd, (unsigned char *)"", 0);
if (code < 200 || code > 299) { /* 失败处理 */ }

/* 发 EHLO（期望 250） */
char hostname[32];
gethostname(hostname, sizeof(hostname));
unsigned char buf[1024];
int len = sprintf((char *)buf, "EHLO %s\r\n", hostname);
code = write_and_get_response(&server_fd, buf, len);
if (code < 200 || code > 299) { /* 失败处理 */ }

/* 发 STARTTLS，期望 220 Ready to start TLS */
len = sprintf((char *)buf, "STARTTLS\r\n");
code = write_and_get_response(&server_fd, buf, len);
if (code < 200 || code > 299) { /* 服务器不支持 STARTTLS，失败处理 */ }
```

### 3. 就地升级：把同一 socket 绑定到 SSL 并握手

收到 `220` 后，立即把同一个 `server_fd` 绑定到 SSL 的 BIO，然后跑 TLS 握手。**socket 不关闭、不重连**——TLS 在原连接之上建立。

```c
/* 关键：复用 server_fd，不重新 net_connect */
mbedtls_ssl_set_bio(&ssl, &server_fd,
                    mbedtls_net_send, mbedtls_net_recv, NULL);

/* TLS 握手（处理 WANT_READ/WANT_WRITE） */
while ((ret = mbedtls_ssl_handshake(&ssl)) != 0) {
    if (ret != MBEDTLS_ERR_SSL_WANT_READ &&
        ret != MBEDTLS_ERR_SSL_WANT_WRITE) {
        /* 致命握手错误 */
        break;
    }
}

/* 校验证书 */
uint32_t flags = mbedtls_ssl_get_verify_result(&ssl);
if (flags != 0) {
    /* 证书校验问题，生产环境应中止 */
}
```

### 4. TLS 阶段：继续 SMTP 会话（经 SSL 层）

升级后所有 SMTP 命令都经 `mbedtls_ssl_write/read` 走加密通道。`write_ssl_and_get_response` 与明文版逻辑相同，只是把 `net_send/recv` 换成 `ssl_write/read`。

```c
/* AUTH LOGIN（base64 编码用户名口令，需 MBEDTLS_BASE64_C） */
len = sprintf((char *)buf, "AUTH LOGIN\r\n");
code = write_ssl_and_get_response(&ssl, buf, len);   /* 期望 334 */

unsigned char b64[1024]; size_t olen;
mbedtls_base64_encode(b64, sizeof(b64), &olen,
                      (const unsigned char *)"user", 4);
len = sprintf((char *)buf, "%s\r\n", b64);
code = write_ssl_and_get_response(&ssl, buf, len);   /* 期望 334 */

mbedtls_base64_encode(b64, sizeof(b64), &olen,
                      (const unsigned char *)"password", 8);
len = sprintf((char *)buf, "%s\r\n", b64);
code = write_ssl_and_get_response(&ssl, buf, len);   /* 期望 235 */

/* 发信 */
len = sprintf((char *)buf, "MAIL FROM:<from@example.com>\r\n");
code = write_ssl_and_get_response(&ssl, buf, len);   /* 期望 250 */
len = sprintf((char *)buf, "RCPT TO:<to@example.com>\r\n");
code = write_ssl_and_get_response(&ssl, buf, len);   /* 期望 250 */
len = sprintf((char *)buf, "DATA\r\n");
code = write_ssl_and_get_response(&ssl, buf, len);   /* 期望 354 */
/* ... 写邮件正文，以 \r\n.\r\n 结束 ... */

mbedtls_ssl_close_notify(&ssl);
```

### 5. 备选：SMTPS（连接即 TLS，mode=0）

若用 submission-over-TLS（端口 465），则跳过明文阶段，连上后立即 `set_bio` + 握手，TLS 建立后再读 SMTP banner 与发 EHLO——所有 SMTP 文本都在 TLS 内。`ssl_mail_client.c` 用 `opt.mode` 区分：`MODE_SSL_TLS`(0) 走此路径，`MODE_STARTTLS`(1) 走上文升级路径。

### 6. 释放资源

```c
mbedtls_ssl_close_notify(&ssl);
mbedtls_net_free(&server_fd);
mbedtls_x509_crt_free(&cacert);
mbedtls_ssl_free(&ssl);
mbedtls_ssl_config_free(&conf);
mbedtls_psa_crypto_free();
```

## 常见错误

| 错误 | 原因 | 解决方法 |
|---|---|---|
| STARTTLS 后握手失败 | 升级前未正确读到 `220`，或 socket 有残留未读数据 | 确保读完 `220` 响应行（含 `\r\n`）后再 `set_bio` + 握手；清空缓冲区 |
| 服务器拒绝 STARTTLS | 服务器 `EHLO` 响应未列出 `STARTTLS` 扩展 | 先解析 `EHLO` 响应确认支持 STARTTLS，不支持则降级或中止 |
| `AUTH LOGIN` 失败 | base64 编码错误或未启用 `MBEDTLS_BASE64_C` | 启用 `MBEDTLS_BASE64_C`；按服务器提示编码用户名/口令 |
| 升级后 SMTP 命令无响应 | 升级后仍用 `net_send/recv` 而非 `ssl_write/read` | TLS 阶段所有读写必须走 `mbedtls_ssl_write/read` |
| 证书校验 flags 非零 | CA 不全或主机名不匹配 | 用完整 CA 链；`set_hostname` 设为邮件服务器实际域名 |
| 明文阶段读到 TLS 握手字节 | 先 `set_bio` 再发 STARTTLS（顺序错） | 严格按"明文读写 → 收 220 → set_bio → 握手"顺序 |
| 移植到无 FS 平台报错 | 示例依赖 `MBEDTLS_FS_IO` 读 CA 文件 | 改用 `mbedtls_x509_crt_parse` 从内存 PEM 加载，或关闭 `MBEDTLS_FS_IO` 并预置 CA |

## 参考

- 示例: `programs/ssl/ssl_mail_client.c` — SMTP-over-TLS / SMTP-STARTTLS 客户端（`mode=0` SMTPS，`mode=1` STARTTLS；含 `AUTH LOGIN` base64 鉴权）
- 文档: `programs/README.md` — 将 `ssl_mail_client.c` 描述为 "a simple SMTP-over-TLS or SMTP-STARTTLS client"
- 头文件: `include/mbedtls/ssl.h`、`include/mbedtls/net_sockets.h`、`include/mbedtls/base64.h`
