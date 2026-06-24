# BSD Socket 网络通信

> **适用摘要**：WASM 应用用标准 BSD socket API（`socket/connect/send/recv` 等）实现 TCP 客户端/服务端；在 `__wasi__` 环境下需包含 `wasi_socket_ext.h`，并配置特殊 sdkconfig.defaults。

## 触发意图

- "WASM TCP 客户端"
- "socket 网络"
- "esp-wdf 联网"
- "wasi_socket_ext"

## 前置条件

| 条件 | 要求 |
|---|---|
| sdkconfig.defaults | `CONFIG_WAMR_APP_FRAMEWORK=y` + `CONFIG_COMPILER_WASI_NO_USE_STDLIB=n` + `CONFIG_COMPILER_WASI_USE_SHARED_MEMORY=y` |
| 头文件 | `<sys/socket.h>`、`<arpa/inet.h>`、`<netinet/in.h>`、`<unistd.h>`；`__wasi__` 下额外 `#include <wasi_socket_ext.h>` |
| Kconfig | `CONFIG_EXAMPLE_IPV4`/`CONFIG_EXAMPLE_IPV6`、`CONFIG_EXAMPLE_IPV4_ADDR`/`CONFIG_EXAMPLE_IPV6_ADDR`、`CONFIG_EXAMPLE_PORT` |
| 参考 | `examples/protocols/sockets/tcp_client/`、`socket-api/tcp_client/`、`socket-api/tcp_server/` |

## 分步说明

### 1. sdkconfig.defaults（取自 examples/protocols/sockets/tcp_client）

```
CONFIG_WAMR_APP_FRAMEWORK=y
CONFIG_COMPILER_WASI_NO_USE_STDLIB=n
CONFIG_COMPILER_WASI_USE_SHARED_MEMORY=y
```

> 必须关闭 `NO_USE_STDLIB`（改用 libc-wasi 提供 socket 符号）并打开 `USE_SHARED_MEMORY`（多线程 socket 需要）。

### 2. TCP 客户端（取自 examples/protocols/sockets/tcp_client，简化）

```c
#include <arpa/inet.h>
#include <netinet/in.h>
#include <pthread.h>
#include <stdio.h>
#include <string.h>
#include <sys/socket.h>
#include <unistd.h>
#ifdef __wasi__
#include <wasi_socket_ext.h>
#endif
#include "esp_log.h"
#include "sdkconfig.h"

#define HOST_IP_ADDR CONFIG_EXAMPLE_IPV4_ADDR
#define PORT         CONFIG_EXAMPLE_PORT

static const char *payload = "Message from ESP32 ";

static void *tcp_client_task(void *pvParameters)
{
    char rx_buffer[128];
    struct sockaddr_in dest_addr;
    inet_pton(AF_INET, HOST_IP_ADDR, &dest_addr.sin_addr);
    dest_addr.sin_family = AF_INET;
    dest_addr.sin_port = htons(PORT);

    int sock = socket(AF_INET, SOCK_STREAM, IPPROTO_IP);
    if (sock < 0) { return NULL; }

    if (connect(sock, (struct sockaddr *)&dest_addr, sizeof(dest_addr)) != 0) {
        close(sock); return NULL;
    }

    while (1) {
        send(sock, payload, strlen(payload), 0);
        int len = recv(sock, rx_buffer, sizeof(rx_buffer) - 1, 0);
        if (len < 0) break;
        rx_buffer[len] = 0;
        printf("Received: %s", rx_buffer);
    }

    shutdown(sock, 0);
    close(sock);
    return NULL;
}

int main(int argc, char *argv[])
{
    pthread_t t;
    pthread_create(&t, NULL, tcp_client_task, NULL);
    pthread_join(t, NULL);
    return 0;
}
```

### 3. 关键点

- `socket-api/` 下的示例（`tcp_client`、`tcp_server`、`send_recv`）演示更多 `setsockopt` 选项（`SO_REUSEADDR`、`SO_REUSEPORT`、`SO_SNDBUF`、`SO_RCVBUF`）与 `getsockname`/`getpeername`。
- 网络任务建议放 `pthread` 内运行，主线程 `pthread_join` 等待。

## 常见错误

| 错误 | 原因 | 解决方法 |
|---|---|---|
| 链接报 socket 符号未定义 | `NO_USE_STDLIB=y` | sdkconfig.defaults 设 `CONFIG_COMPILER_WASI_NO_USE_STDLIB=n` |
| 多线程 socket 崩溃 | 未共享内存 | 设 `CONFIG_COMPILER_WASI_USE_SHARED_MEMORY=y` |
| `__wasi__` 下连接 API 缺失 | 未包含扩展头 | `#ifdef __wasi__` `#include <wasi_socket_ext.h>` |
| 连接立即失败 | 目标 IP/端口错 | 核对 `CONFIG_EXAMPLE_IPV4_ADDR`/`CONFIG_EXAMPLE_PORT` |

## 参考

- `examples/protocols/sockets/tcp_client/main/tcp_client.c`
- `examples/protocols/sockets/socket-api/tcp_client/main/tcp_client.c`
- `examples/protocols/sockets/tcp_client/sdkconfig.defaults`
- `components/wamr/lib-socket/inc/wasi_socket_ext.h`
- `resources/config_reference.md` —— 第一节 编译器选项
- `resources/pitfalls.md` —— 第 4 条 网络示例三件齐备
