# 多进程 fork() 并发 TLS 服务器

> **适用摘要**: 用 POSIX `fork()` 实现"每客户端一进程"的并发 TLS 服务器。与 `tls_server.md` 的单连接模型、`ssl_pthread_server.c` 的每客户端一线程模型不同，进程级隔离让单个客户端的崩溃不会污染监听上下文或其他客户端。需要在 accept 后 `fork()`，父进程关 client_fd 继续监听，子进程关 listen_fd 专属服务该客户端。仅限 Unix/POSIX 环境（不支持 Windows）。

## 触发意图

- "fork 并发 TLS 服务器"
- "每客户端一进程"
- "进程隔离 TLS server"
- "ssl_fork_server"
- "SIGCHLD 僵尸进程"

## 前置条件

| 条件 | 要求 |
|---|---|
| 参考示例 | `programs/ssl/ssl_fork_server.c` |
| 配置宏 | `MBEDTLS_NET_C`、`MBEDTLS_SSL_SRV_C`、`MBEDTLS_X509_CRT_PARSE_C`、`MBEDTLS_PEM_PARSE_C` |
| 平台 | POSIX（提供 `fork`、`signal`、`SIGCHLD`、`unistd.h`）；**不支持 Windows**（示例在 `_WIN32` 下直接打印提示并退出） |
| 依赖 | `psa_crypto_init()` |

## 分步说明

fork 模型的核心：**监听 socket 与 SSL 配置在 fork 前一次性建立，被子进程继承**；每个客户端由独立进程服务，父进程通过 `SIGCHLD` 的 `SIG_IGN` 自动回收僵尸子进程。与线程模型（`tls_server.md` / `ssl_pthread_server.c`）相比，无需 `session_reset` 复用上下文——每个子进程有自己独立的 `ssl` 上下文，但共享同一份只读的 `conf`（含证书/私钥）。

### 1. 父进程：装载证书、配置、绑定监听（fork 前）

证书、私钥、`conf` 在 fork 前准备好，子进程继承其只读副本。

```c
#include "mbedtls/build_info.h"
#include "mbedtls/platform.h"
#include "mbedtls/ssl.h"
#include "mbedtls/net_sockets.h"
#include <signal.h>
#include <unistd.h>

mbedtls_net_context listen_fd, client_fd;
mbedtls_ssl_context ssl;
mbedtls_ssl_config conf;
mbedtls_x509_crt srvcert;
mbedtls_pk_context pkey;

mbedtls_net_init(&listen_fd);
mbedtls_net_init(&client_fd);
mbedtls_ssl_init(&ssl);
mbedtls_ssl_config_init(&conf);
mbedtls_x509_crt_init(&srvcert);
mbedtls_pk_init(&pkey);

psa_crypto_init();

/* 关键：忽略 SIGCHLD，内核自动回收僵尸子进程（免去 waitpid 循环） */
signal(SIGCHLD, SIG_IGN);

/* 装载服务器证书链与私钥 */
mbedtls_x509_crt_parse(&srvcert, srv_crt_pem, srv_crt_len);
mbedtls_x509_crt_parse(&srvcert, ca_pem, ca_len);
mbedtls_pk_parse_key(&pkey, srv_key_pem, srv_key_len, NULL, 0);

mbedtls_ssl_config_defaults(&conf, MBEDTLS_SSL_IS_SERVER,
                            MBEDTLS_SSL_TRANSPORT_STREAM,
                            MBEDTLS_SSL_PRESET_DEFAULT);
mbedtls_ssl_conf_ca_chain(&conf, srvcert.next, NULL);
mbedtls_ssl_conf_own_cert(&conf, &srvcert, &pkey);
mbedtls_ssl_conf_dbg(&conf, my_debug, stdout);

/* 绑定监听端口 */
mbedtls_net_bind(&listen_fd, NULL, "4433", MBEDTLS_NET_PROTO_TCP);
```

### 2. accept + fork 循环（父进程关 client_fd，子进程关 listen_fd）

每来一个客户端：accept → fork。父进程立即关闭 client_fd（继续监听下一个），子进程关闭 listen_fd（不再监听）后专属服务该客户端。

```c
while (1) {
    /* 每轮重新初始化 client_fd 与 ssl（子进程各自独立） */
    mbedtls_net_init(&client_fd);
    mbedtls_ssl_init(&ssl);

    mbedtls_net_accept(&listen_fd, &client_fd, NULL, 0, NULL);

    pid_t pid = fork();
    if (pid < 0) {
        /* fork 失败 */
        break;
    }

    if (pid != 0) {
        /* 父进程：不需要 client_fd，关闭后继续 accept */
        mbedtls_net_close(&client_fd);
        continue;
    }

    /* 子进程：不再监听，关闭 listen_fd */
    mbedtls_net_close(&listen_fd);
    pid = getpid();

    /* 在子进程中专属服务该客户端（见下一步） */
    /* ... mbedtls_ssl_setup + set_bio + handshake + read/write ... */
    /* 服务完成后 goto exit / _exit() */
}
```

### 3. 子进程：setup + 握手 + 读写

子进程用继承的 `conf` 做 `ssl_setup`（每个子进程独立的 ssl 上下文），绑定继承的 `client_fd`，然后正常握手与读写。

```c
/* 子进程专属服务 */
ret = mbedtls_ssl_setup(&ssl, &conf);
mbedtls_ssl_set_bio(&ssl, &client_fd,
                    mbedtls_net_send, mbedtls_net_recv, NULL);

/* 握手 */
while ((ret = mbedtls_ssl_handshake(&ssl)) != 0) {
    if (ret != MBEDTLS_ERR_SSL_WANT_READ &&
        ret != MBEDTLS_ERR_SSL_WANT_WRITE) {
        /* 子进程握手失败，退出 */
        break;
    }
}

/* 读 HTTP 请求 */
do {
    ret = mbedtls_ssl_read(&ssl, buf, sizeof(buf) - 1);
    if (ret == MBEDTLS_ERR_SSL_WANT_READ ||
        ret == MBEDTLS_ERR_SSL_WANT_WRITE) { continue; }
    if (ret <= 0) { break; }     /* PEER_CLOSE_NOTIFY / CONN_RESET 等 */
    /* 处理请求 */
} while (1);

/* 写响应 */
while ((ret = mbedtls_ssl_write(&ssl, resp, resp_len)) <= 0) {
    if (ret != MBEDTLS_ERR_SSL_WANT_READ &&
        ret != MBEDTLS_ERR_SSL_WANT_WRITE) { break; }
}

mbedtls_ssl_close_notify(&ssl);
/* 子进程退出（exit 段会释放资源） */
```

### 4. 释放资源（父进程退出时）

```c
/* 父进程退出路径 */
mbedtls_net_free(&listen_fd);
mbedtls_x509_crt_free(&srvcert);
mbedtls_pk_free(&pkey);
mbedtls_ssl_config_free(&conf);
mbedtls_psa_crypto_free();
/* 注意：每个子进程退出时也应释放各自持有的 ssl/client_fd/资源 */
```

## 常见错误

| 错误 | 原因 | 解决方法 |
|---|---|---|
| `_WIN32` 下直接退出 | fork 模型不支持 Windows | 仅在 POSIX 平台使用；Windows 改用线程模型（见 `recipes/tls_server.md`） |
| 子进程退出后变僵尸 | 父进程未回收 | fork 前 `signal(SIGCHLD, SIG_IGN)` 让内核自动回收，或在循环中 `waitpid` |
| 父进程 fd 泄漏 | accept 后未在父进程关 client_fd | 父进程分支内 `mbedtls_net_close(&client_fd)` 后再 `continue` |
| 子进程仍占用 listen_fd | 未在子进程关 listen_fd | 子进程分支内 `mbedtls_net_close(&listen_fd)` |
| 子进程间证书/私钥被误改 | 误以为 conf 可写 | fork 后 conf 是写时复制副本，子进程只读用，不要修改共享 conf |
| fork 后 `ssl_setup` 失败 | 在 fork 前 setup（上下文被多进程共享会错乱） | 在**子进程内** `ssl_setup`，每个子进程独立 ssl 上下文 |
| 大量子进程耗尽资源 | 无并发上限 | 加 accept 计数或信号量限流；或改用线程池 |

## 参考

- 示例: `programs/ssl/ssl_fork_server.c` — 每客户端一进程的 HTTPS 服务器（`signal(SIGCHLD, SIG_IGN)`、fork 后父关 client_fd / 子关 listen_fd）
- 文档: `programs/README.md` — 将该示例描述为 "one process per client ... requires a Unix/POSIX environment implementing the `fork` system call"
- 对比: `recipes/tls_server.md`（单连接 + `ssl_pthread_server.c` 线程模型）；线程模型每客户端复用 ssl 上下文需 `session_reset`，fork 模型每子进程独立 ssl 无需 reset
- 头文件: `include/mbedtls/ssl.h`、`include/mbedtls/net_sockets.h`
