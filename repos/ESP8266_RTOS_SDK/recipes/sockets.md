# BSD Socket：TCP/UDP 服务端客户端与组播

> **适用摘要**: 用 lwIP BSD socket API 实现 TCP 服务端（`socket`→`bind`→`listen`→`accept`→`recv/send`）、TCP 客户端（`connect`）、UDP 收发与 IPv4 组播（`setsockopt IP_ADD_MEMBERSHIP/IP_MULTICAST_TTL`）。本 recipe 补充 `http_request.md` 没覆盖的服务端/组播场景。

> Version: ESP8266 RTOS SDK version used by the project.
> Evidence: `repos/ESP8266_RTOS_SDK/resources/`, source/examples in `repos/ESP8266_RTOS_SDK/`, and this recipe path `repos/ESP8266_RTOS_SDK/recipes/sockets.md`.
> Validation: draft metadata added from repository routing; verify APIs, Kconfig symbols, and component dependencies against the selected version.

## 触发意图

- "TCP 服务端 / TCP server"
- "UDP 收发 / 组播 / multicast"
- "socket bind listen accept"
- "设备间 socket 通信"

## 前置条件

| 条件 | 要求 |
|---|---|
| 参考示例 | `examples/protocols/sockets/{tcp_client,tcp_server,udp_client,udp_server,udp_multicast}` |
| 联网 | 先 `example_connect()`（依赖 `protocol_examples_common`） |
| 配置 | menuconfig → `Example Configuration` 选 `CONFIG_EXAMPLE_IPV4`（或 IPV6）、设 `CONFIG_EXAMPLE_PORT` |
| 栈 | socket 任务栈建议 4096（DNS/socket 占栈） |
| 头文件 | `lwip/sockets.h`、`lwip/netdb.h`、`lwip/err.h` |

## 分步说明

### 1. app_main 初始化（通用）

```c
#include "nvs_flash.h"
#include "esp_netif.h"
#include "esp_event.h"
#include "protocol_examples_common.h"
#include "freertos/FreeRTOS.h"
#include "freertos/task.h"

void app_main(void)
{
    ESP_ERROR_CHECK(nvs_flash_init());
    ESP_ERROR_CHECK(esp_netif_init());
    ESP_ERROR_CHECK(esp_event_loop_create_default());
    ESP_ERROR_CHECK(example_connect());

    xTaskCreate(tcp_server_task, "tcp_server", 4096, NULL, 5, NULL);
}
```

### 2. TCP 服务端（bind/listen/accept，改编自 tcp_server 示例）

```c
#include <string.h>
#include "lwip/sockets.h"
#include <lwip/netdb.h>
#include "esp_log.h"

#define PORT CONFIG_EXAMPLE_PORT          // 默认 3333
static const char *TAG = "example";

static void tcp_server_task(void *pv)
{
    char rx_buffer[128];
    char addr_str[128];

    while (1) {
        struct sockaddr_in destAddr = {0};
        destAddr.sin_addr.s_addr = htonl(INADDR_ANY);
        destAddr.sin_family      = AF_INET;
        destAddr.sin_port        = htons(PORT);
        inet_ntoa_r(destAddr.sin_addr, addr_str, sizeof(addr_str) - 1);

        int listen_sock = socket(AF_INET, SOCK_STREAM, IPPROTO_IP);
        if (listen_sock < 0) { ESP_LOGE(TAG, "socket: errno %d", errno); break; }

        if (bind(listen_sock, (struct sockaddr *)&destAddr, sizeof(destAddr)) != 0) {
            ESP_LOGE(TAG, "bind: errno %d", errno); break;
        }
        if (listen(listen_sock, 1) != 0) {                // backlog=1
            ESP_LOGE(TAG, "listen: errno %d", errno); break;
        }
        ESP_LOGI(TAG, "listening on %s:%d", addr_str, PORT);

        struct sockaddr_in sourceAddr;
        uint addrLen = sizeof(sourceAddr);
        int sock = accept(listen_sock, (struct sockaddr *)&sourceAddr, &addrLen);
        if (sock < 0) { ESP_LOGE(TAG, "accept: errno %d", errno); break; }
        close(listen_sock);                               // 只接一个连接

        while (1) {
            int len = recv(sock, rx_buffer, sizeof(rx_buffer) - 1, 0);
            if (len < 0)  { ESP_LOGE(TAG, "recv: errno %d", errno); break; }
            if (len == 0) { ESP_LOGI(TAG, "conn closed");   break; }

            rx_buffer[len] = 0;
            inet_ntoa_r(((struct sockaddr_in *)&sourceAddr)->sin_addr.s_addr,
                        addr_str, sizeof(addr_str) - 1);
            ESP_LOGI(TAG, "Recv %d bytes from %s: %s", len, addr_str, rx_buffer);

            if (send(sock, rx_buffer, len, 0) < 0) {       // 回显
                ESP_LOGE(TAG, "send: errno %d", errno); break;
            }
        }
        shutdown(sock, 0);
        close(sock);
    }
    vTaskDelete(NULL);
}
```

### 3. TCP 客户端（connect，改编自 tcp_client 示例）

```c
static void tcp_client_task(void *pv)
{
    char rx_buffer[128];
    const char *payload = "Message from ESP8266 ";

    while (1) {
        struct sockaddr_in destAddr = {0};
        destAddr.sin_addr.s_addr = inet_addr(CONFIG_EXAMPLE_IPV4_ADDR);  // 如 "192.168.1.10"
        destAddr.sin_family      = AF_INET;
        destAddr.sin_port        = htons(PORT);

        int sock = socket(AF_INET, SOCK_STREAM, IPPROTO_IP);
        if (sock < 0) { ESP_LOGE(TAG, "socket: errno %d", errno); break; }

        if (connect(sock, (struct sockaddr *)&destAddr, sizeof(destAddr)) != 0) {
            ESP_LOGE(TAG, "connect: errno %d", errno);
            close(sock); vTaskDelay(2000/portTICK_PERIOD_MS); continue;
        }

        while (1) {
            if (send(sock, payload, strlen(payload), 0) < 0) break;
            int len = recv(sock, rx_buffer, sizeof(rx_buffer) - 1, 0);
            if (len <= 0) break;
            rx_buffer[len] = 0;
            ESP_LOGI(TAG, "Received %d bytes: %s", len, rx_buffer);
            vTaskDelay(2000 / portTICK_PERIOD_MS);
        }
        shutdown(sock, 0); close(sock);
    }
    vTaskDelete(NULL);
}
```

### 4. UDP IPv4 组播（加入 IGMP 组 + 收发）

udp_multicast 示例的核心是建一个 UDP socket，`IP_ADD_MEMBERSHIP` 加入组，并用 `select()` 等待：

```c
#include "lwip/sockets.h"
#include "tcpip_adapter.h"

#define UDP_PORT            CONFIG_EXAMPLE_PORT
#define MULTICAST_TTL       CONFIG_EXAMPLE_MULTICAST_TTL
#define MULTICAST_IPV4_ADDR CONFIG_EXAMPLE_MULTICAST_IPV4_ADDR   // 如 "239.1.1.1"

// 加入 IPv4 组播组（必须先 bind）
static int create_multicast_ipv4_socket(void)
{
    struct sockaddr_in saddr = {0};
    int sock = socket(PF_INET, SOCK_DGRAM, IPPROTO_IP);
    if (sock < 0) return -1;

    saddr.sin_family      = PF_INET;
    saddr.sin_port        = htons(UDP_PORT);
    saddr.sin_addr.s_addr = htonl(INADDR_ANY);
    if (bind(sock, (struct sockaddr *)&saddr, sizeof(saddr)) < 0) { close(sock); return -1; }

    // TTL
    uint8_t ttl = MULTICAST_TTL;
    setsockopt(sock, IPPROTO_IP, IP_MULTICAST_TTL, &ttl, sizeof(ttl));

    // 加入组播组
    struct ip_mreq imreq = {0};
    imreq.imr_interface.s_addr = IPADDR_ANY;                 // 默认接口
    inet_aton(MULTICAST_IPV4_ADDR, &imreq.imr_multiaddr.s_addr);
    if (setsockopt(sock, IPPROTO_IP, IP_ADD_MEMBERSHIP, &imreq, sizeof(imreq)) < 0) {
        close(sock); return -1;
    }
    return sock;
}

static void mcast_task(void *pv)
{
    int sock = create_multicast_ipv4_socket();
    struct sockaddr_in sdest = {0};
    sdest.sin_family = PF_INET;
    sdest.sin_port   = htons(UDP_PORT);
    inet_aton(MULTICAST_IPV4_ADDR, &sdest.sin_addr.s_addr);

    while (1) {
        struct timeval tv = { .tv_sec = 2 };
        fd_set rfds; FD_ZERO(&rfds); FD_SET(sock, &rfds);
        int s = select(sock + 1, &rfds, NULL, NULL, &tv);

        if (s > 0 && FD_ISSET(sock, &rfds)) {
            char buf[48]; struct sockaddr_in6 raddr; socklen_t slen = sizeof(raddr);
            int len = recvfrom(sock, buf, sizeof(buf)-1, 0, (struct sockaddr*)&raddr, &slen);
            if (len > 0) { buf[len] = 0; ESP_LOGI("mcast", "recv: %s", buf); }
        } else if (s == 0) {
            // 超时：主动发一包
            const char *msg = "Multicast from ESP8266";
            sendto(sock, msg, strlen(msg), 0, (struct sockaddr*)&sdest, sizeof(sdest));
        }
    }
}
```

> UDP 普通收发（client/server）只需 `socket(AF_INET, SOCK_DGRAM, ...)` + `bind`/`sendto`/`recvfrom`，无需 `IP_ADD_MEMBERSHIP`。

### 5. 关键 setsockopt 选项

| level | optname | 作用 |
|---|---|---|
| `SOL_SOCKET` | `SO_RCVTIMEO` | 设 recv 超时（防永久阻塞） |
| `IPPROTO_IP` | `IP_ADD_MEMBERSHIP` | 加入组播组（`struct ip_mreq`） |
| `IPPROTO_IP` | `IP_MULTICAST_TTL` | 组播 TTL |
| `IPPROTO_IP` | `IP_MULTICAST_IF` | 指定组播源接口 IP |
| `IPPROTO_IP` | `IP_MULTICAST_LOOP` | 是否回环接收自己发的 |

### 6. 关键 API

| API | 作用 |
|---|---|
| `socket(AF_INET, SOCK_STREAM/SOCK_DGRAM, ...)` | 建 TCP/UDP socket |
| `bind(sock, addr, len)` | 绑定本地地址/端口 |
| `listen(sock, backlog)` | TCP：开始监听 |
| `accept(listen_sock, addr, &len)` | TCP：接受连接，返回新 sock |
| `connect(sock, addr, len)` | TCP：主动连接 |
| `send / recv` | 已连接 socket 收发 |
| `sendto / recvfrom` | UDP（带目标/源地址） |
| `select(...)` | 多路复用等待 |
| `inet_addr / inet_aton / inet_ntoa_r` | IP 串与数值互转 |
| `shutdown(sock, 0) + close(sock)` | 优雅关闭 |

## 常见错误

| 错误 | 原因 | 解决方法 |
|---|---|---|
| `bind` 失败 | 端口被占 / 还没拿到 IP | 换端口；先等 `IP_EVENT_STA_GOT_IP` |
| `accept` 永久阻塞 | 没 set 超时 | 先 `SO_RCVTIMEO` 或用 `select` |
| 组播收不到 | 没 `IP_ADD_MEMBERSHIP` / 地址非 224.0.0.0/4 | 先加入组；地址范围 `224.0.0.0`~`239.255.255.255` |
| 组播发不出 | TTL 太低 / 接口选错 | `IP_MULTICAST_TTL`≥2；`IP_MULTICAST_IF` 设 STA IP |
| 连接后立刻断 | 对端没监听 / 防火墙 | 确认对端 server 在跑 |
| `recv` 返回 0 | 对端关闭连接 | 正常，break 重连 |

## 参考

- `examples/protocols/sockets/tcp_server/main/tcp_server.c` — TCP 服务端（listen/accept/echo）
- `examples/protocols/sockets/tcp_client/main/tcp_client.c` — TCP 客户端
- `examples/protocols/sockets/udp_server` / `udp_client` — UDP 收发
- `examples/protocols/sockets/udp_multicast/main/udp_multicast_example_main.c` — IPv4/IPv6 组播（`IP_ADD_MEMBERSHIP` + `select`）
- `examples/protocols/sockets/README.md` — netcat 测试命令
- `recipes/http_request.md` — 最简单的 socket GET（单次请求）
