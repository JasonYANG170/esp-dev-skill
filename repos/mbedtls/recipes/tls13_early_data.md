# TLS 1.3 早期数据（0-RTT）收发与拒绝处理

> **适用摘要**: 在 TLS 1.3 中收发早期数据（0-RTT data）。客户端用 `mbedtls_ssl_write_early_data()` 在握手首飞即发送应用数据、用 `mbedtls_ssl_get_early_data_status()` 判断服务端是否接受；服务端用 `mbedtls_ssl_conf_early_data()` 开启接收、在握手/read/write 返回 `MBEDTLS_ERR_SSL_RECEIVED_EARLY_DATA` 时用 `mbedtls_ssl_read_early_data()` 读取。早期数据 API 与普通 `ssl_read/write` 语义差异较大（独有的 `CANNOT_WRITE_EARLY_DATA` / `RECEIVED_EARLY_DATA` / `EARLY_DATA_STATUS_REJECTED` 分支），需单独处理。

> Version: selected repo/component version; align with the user project and dependency manifest.
> Evidence: `repos/mbedtls/resources/`, source/examples in `repos/mbedtls/`, and this recipe path `repos/mbedtls/recipes/tls13_early_data.md`.
> Validation: draft metadata added from repository routing; verify APIs, Kconfig symbols, and component dependencies against the selected version.

## 触发意图

- "TLS 1.3 0-RTT"
- "early data 收发"
- "0-RTT 重放 / 拒绝处理"
- "write_early_data / read_early_data"
- "TLS 1.3 降低首包延迟"

## 前置条件

| 条件 | 要求 |
|---|---|
| 参考示例 | `programs/ssl/ssl_client2.c`（`early_data=` 选项）、`programs/ssl/ssl_server2.c`（`early_data=` / `max_early_data_size=`） |
| 配置宏 | `MBEDTLS_SSL_PROTO_TLS1_3`、`MBEDTLS_SSL_EARLY_DATA`；`MBEDTLS_SSL_SESSION_TICKETS`；且 `MBEDTLS_SSL_TLS1_3_KEY_EXCHANGE_MODE_PSK_ENABLED` 或 `_PSK_EPHEMERAL_ENABLED` 至少启用一个 |
| 依赖 | `psa_crypto_init()`；服务端须已发送允许 early data 的 NewSessionTicket，客户端须持可用 PSK/ticket |
| 安全前提 | 早期数据**非前向安全**且**无重放保护**——仅用于幂等请求，禁止用于鉴权或产生副作用的操作 |

## 分步说明

早期数据是 TLS 1.3 中客户端用 PSK 在握手完成前即发送的应用数据（RFC 8446 第 2.3 节）。其 API 与普通 `ssl_read/write` 并存但语义独立：客户端的 `write_early_data` 只会"驱动握手到无法再发早期数据为止"（而非完成握手）；服务端则在普通握手/read/write 中以 `MBEDTLS_ERR_SSL_RECEIVED_EARLY_DATA` 这一返回码通知"收到早期数据，请先读出来再继续"。

### 1. 配置：开启早期数据（客户端与服务端）

```c
#include "mbedtls/ssl.h"

/* 客户端与服务端默认都关闭早期数据，需显式开启 */
mbedtls_ssl_conf_early_data(&conf, MBEDTLS_SSL_EARLY_DATA_ENABLED);

/* 仅服务端：设置 NewSessionTicket 中声明的最大早期数据字节数 */
mbedtls_ssl_conf_max_early_data_size(&conf, 4096);
/*   未调用时默认为 MBEDTLS_SSL_MAX_EARLY_DATA_SIZE (1024)。
 *   注意：超过该限额时，恢复连接的服务端会直接终止连接。
 *   此设置只影响此后签发的新 ticket，不约束已签发的旧 ticket。 */
```

### 2. 客户端：发送早期数据

`write_early_data` 必须在一个刚 `ssl_setup()` 或 `session_reset()` 过的上下文上、握手尚未完成时调用。它返回已写入字节数（可能小于 `len`），并在无法再写早期数据时返回 `MBEDTLS_ERR_SSL_CANNOT_WRITE_EARLY_DATA`——此时该 SSL 上下文此后永远不能再写早期数据，应改用普通 `mbedtls_ssl_write`。

```c
/* 循环写入早期数据；与 ssl_write 类似需处理 WANT_READ/WANT_WRITE */
int write_early_data(mbedtls_ssl_context *ssl,
                     const unsigned char *buf, size_t len,
                     size_t *written)
{
    int ret;
    *written = 0;
    while (*written < len) {
        ret = mbedtls_ssl_write_early_data(ssl,
                                           buf + *written,
                                           len - *written);
        if (ret < 0 &&
            ret != MBEDTLS_ERR_SSL_WANT_READ &&
            ret != MBEDTLS_ERR_SSL_WANT_WRITE) {
            return ret;          /* 通常是 CANNOT_WRITE_EARLY_DATA，或致命错误 */
        }
        *written += ret;          /* ret > 0：已写入字节数 */
    }
    return 0;
}
```

### 3. 客户端：判断服务端是否接受早期数据

`write_early_data` 即便返回正值也**不**表示服务端会接受早期数据——它只表示数据已发出。要确认是否被接受，必须等握手完成后调用 `mbedtls_ssl_get_early_data_status()`。

```c
size_t early_written = 0;
ret = write_early_data(&ssl, app_data, app_data_len, &early_written);
if (ret < 0 && ret != MBEDTLS_ERR_SSL_CANNOT_WRITE_EARLY_DATA) {
    /* 致命错误，处理 */
}

/* get_early_data_status 前置条件：握手必须完成 */
while (!mbedtls_ssl_is_handshake_over(&ssl)) {
    ret = mbedtls_ssl_handshake(&ssl);
    if (ret < 0 &&
        ret != MBEDTLS_ERR_SSL_WANT_READ &&
        ret != MBEDTLS_ERR_SSL_WANT_WRITE) {
        /* 致命错误 */
        break;
    }
}

ret = mbedtls_ssl_get_early_data_status(&ssl);
if (ret == MBEDTLS_SSL_EARLY_DATA_STATUS_REJECTED) {
    /* 服务端拒绝了早期数据：必须把已发出的那部分作为普通应用数据重发 */
    early_written = 0;
}
/* 此后用 mbedtls_ssl_write 补发 buf + early_written 之后的内容 */
```

### 4. 服务端：接收早期数据

服务端启用 `conf_early_data` 后，`handshake()` / `read()` / `write()` 可能返回 `MBEDTLS_ERR_SSL_RECEIVED_EARLY_DATA` 表示"握手进行中收到早期数据"。此时必须先调用 `read_early_data()` 读取，再继续原操作（通常 `continue` 握手循环）。

```c
unsigned char early_buf[1024];
size_t total_read = 0;

while ((ret = mbedtls_ssl_handshake(&ssl)) != 0) {
    if (ret == MBEDTLS_ERR_SSL_RECEIVED_EARLY_DATA) {
        ret = mbedtls_ssl_read_early_data(&ssl,
                                          early_buf + total_read,
                                          sizeof(early_buf) - total_read);
        if (ret < 0) {
            /* 致命错误（如 CANNOT_READ_EARLY_DATA） */
            break;
        }
        total_read += ret;        /* ret > 0：本次读到的字节数 */
        continue;                 /* 继续握手 */
    }
    if (ret != MBEDTLS_ERR_SSL_WANT_READ &&
        ret != MBEDTLS_ERR_SSL_WANT_WRITE) {
        break;                    /* 致命错误或完成 */
    }
}
```

### 5. 完整客户端收发流（发早期数据 + 拒绝后重发）

综合上面三步，一个"尽量发早期数据，被拒就重发为普通数据"的完整模式：

```c
size_t early_written = 0;

/* (a) 尽量发早期数据 */
ret = write_early_data(&ssl, payload, payload_len, &early_written);
if (ret < 0 && ret != MBEDTLS_ERR_SSL_CANNOT_WRITE_EARLY_DATA) {
    goto error;
}

/* (b) 推进握手到完成（get_early_data_status 的硬性前置条件） */
while (!mbedtls_ssl_is_handshake_over(&ssl)) {
    ret = mbedtls_ssl_handshake(&ssl);
    if (ret < 0 &&
        ret != MBEDTLS_ERR_SSL_WANT_READ &&
        ret != MBEDTLS_ERR_SSL_WANT_WRITE) {
        goto error;
    }
}

/* (c) 查询服务端是否接受 */
if (mbedtls_ssl_get_early_data_status(&ssl)
    == MBEDTLS_SSL_EARLY_DATA_STATUS_REJECTED) {
    early_written = 0;            /* 早期数据被拒，全部重发为普通数据 */
}

/* (d) 把"未成功发为早期数据"的剩余部分作为普通应用数据发出 */
size_t done = early_written;
while (done < payload_len) {
    ret = mbedtls_ssl_write(&ssl, payload + done, payload_len - done);
    if (ret < 0 &&
        ret != MBEDTLS_ERR_SSL_WANT_READ &&
        ret != MBEDTLS_ERR_SSL_WANT_WRITE) {
        goto error;
    }
    done += ret;
}
```

## 常见错误

| 错误 | 原因 | 解决方法 |
|---|---|---|
| `CANNOT_WRITE_EARLY_DATA` 一调用就返回 | 未 `conf_early_data(ENABLED)`，或未配置允许 early data 的 PSK/ticket | 先 `conf_early_data`，并确认 PSK/ticket 允许 early data（服务端 `max_early_data_size` > 0） |
| `CANNOT_WRITE_EARLY_DATA` 发了一部分后返回 | 已到达 PSK 允许的 early data 上限，或已收到服务端 Finished/被拒 | 正常，改用普通 `mbedtls_ssl_write` 发送剩余部分 |
| `get_early_data_status` 返回 `PSA_ERROR_INVALID_ARGUMENT` | 握手未完成就调用，或在服务端调用 | 先用 `mbedtls_ssl_is_handshake_over` 确认握手结束；该 API 仅客户端可用 |
| 客户端发了数据但服务端没收到 | 服务端拒绝早期数据（`EARLY_DATA_STATUS_REJECTED`），但客户端未重发 | 查询 status，被拒时把 `early_written` 归零后用 `ssl_write` 重发 |
| 服务端 `handshake` 返回 `RECEIVED_EARLY_DATA` 被误判为致命错误 | 该码是中间态而非错误 | 调 `read_early_data` 读取后 `continue` 握手循环 |
| `CANNOT_READ_EARLY_DATA` | 在非 `RECEIVED_EARLY_DATA` 上下文调用 `read_early_data` | 只在 `handshake/read/write` 返回 `RECEIVED_EARLY_DATA` 后立即调用 |
| 早期数据被重放导致重复副作用 | 0-RTT 无重放保护，Mbed TLS 未实现 RFC 8446 第 8 节防重放 | 只用早期数据发幂等请求（如纯查询），鉴权/写操作必须等握手完成 |
| 编译报 `mbedtls_ssl_..._early_data` 未定义 | 未启用 `MBEDTLS_SSL_EARLY_DATA` | 在 `mbedtls_config.h` 启用该宏及其前置依赖（ticket + PSK 或 PSK-ephemeral 模式） |

## 参考

- 文档: `docs/tls13-early-data.md` — 早期数据收发完整流程与代码示例
- 文档: `docs/architecture/tls13-support.md` — TLS 1.3 能力清单（early data 列为支持）与早期数据安全属性说明
- 示例: `programs/ssl/ssl_client2.c`（`early_data=` 选项，`write_early_data` 写循环）
- 示例: `programs/ssl/ssl_server2.c`（`early_data=` / `max_early_data_size=`，`read_early_data` 读分支）
- 头文件: `include/mbedtls/ssl.h`（`mbedtls_ssl_conf_early_data`、`_conf_max_early_data_size`、`_write_early_data`、`_read_early_data`、`_get_early_data_status`、`_is_handshake_over` 及 `MBEDTLS_ERR_SSL_*EARLY_DATA*` / `MBEDTLS_SSL_EARLY_DATA_*` 定义）
