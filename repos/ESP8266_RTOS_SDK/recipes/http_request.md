# HTTP GET 请求（POSIX socket）

> **适用摘要**: 在已联网的 ESP8266 上，用 BSD socket（`getaddrinfo`/`socket`/`connect`/`write`/`read`）发起 HTTP GET 请求并打印响应。适用于明文 HTTP；HTTPS 见 `examples/protocols/https_mbedtls` 或 `esp_http_client`。

## 触发意图

- "HTTP 请求"
- "GET 下载网页"
- "socket 客户端"
- "发 HTTP"

## 前置条件

| 条件 | 要求 |
|---|---|
| 参考示例 | `examples/protocols/http_request/` |
| 联网 | 先完成 `recipes/wifi_station.md`（示例用 `protocol_examples_common` 的 `example_connect()`，依赖 `common_components`） |
| 栈 | HTTP 任务栈建议 ≥ 16384（DNS/socket 较吃栈） |

## 分步说明

### 1. app_main 初始化（改编自示例）

```c
#include "nvs_flash.h"
#include "esp_netif.h"
#include "esp_event.h"
#include "protocol_examples_common.h"   // 提供 example_connect()

void app_main()
{
    ESP_ERROR_CHECK(nvs_flash_init());
    ESP_ERROR_CHECK(esp_netif_init());                          // 新风格网络栈
    ESP_ERROR_CHECK(esp_event_loop_create_default());
    ESP_ERROR_CHECK(example_connect());                         // 连 WiFi（内部事件处理）

    xTaskCreate(&http_get_task, "http_get_task", 16384, NULL, 5, NULL);
}
```

### 2. http_get_task（getaddrinfo → socket → connect → write → read）

```c
#include <netdb.h>
#include <sys/socket.h>
#include "esp_log.h"
#include "freertos/FreeRTOS.h"
#include "freertos/task.h"

#define WEB_SERVER "example.com"
#define WEB_PORT   80
#define WEB_URL    "http://example.com/"

static const char *REQUEST = "GET " WEB_URL " HTTP/1.0\r\n"
    "Host: "WEB_SERVER"\r\n"
    "User-Agent: esp-idf/1.0 esp8266\r\n"
    "\r\n";

static void http_get_task(void *pv)
{
    const struct addrinfo hints = { .ai_family = AF_INET, .ai_socktype = SOCK_STREAM };
    struct addrinfo *res;
    char recv_buf[64];

    while (1) {
        int err = getaddrinfo(WEB_SERVER, "80", &hints, &res);
        if (err != 0 || res == NULL) {
            ESP_LOGE("example", "DNS lookup failed err=%d", err);
            vTaskDelay(1000 / portTICK_PERIOD_MS);
            continue;
        }

        int s = socket(res->ai_family, res->ai_socktype, 0);
        if (s < 0) { ESP_LOGE("example", "... allocate socket"); freeaddrinfo(res); continue; }

        if (connect(s, res->ai_addr, res->ai_addrlen) != 0) {
            ESP_LOGE("example", "... connect failed errno=%d", errno);
            close(s); freeaddrinfo(res); vTaskDelay(4000 / portTICK_PERIOD_MS); continue;
        }
        freeaddrinfo(res);

        if (write(s, REQUEST, strlen(REQUEST)) < 0) {
            ESP_LOGE("example", "... send failed"); close(s); continue;
        }

        // 接收超时，避免长时间阻塞
        struct timeval to = { .tv_sec = 5 };
        setsockopt(s, SOL_SOCKET, SO_RCVTIMEO, &to, sizeof(to));

        int r;
        do {
            bzero(recv_buf, sizeof(recv_buf));
            r = read(s, recv_buf, sizeof(recv_buf) - 1);
            for (int i = 0; i < r; i++) putchar(recv_buf[i]);
        } while (r > 0);

        ESP_LOGI("example", "... done reading. last=%d errno=%d", r, errno);
        close(s);
        for (int c = 10; c >= 0; c--) {
            vTaskDelay(1000 / portTICK_PERIOD_MS);
        }
    }
}
```

### 3. 关键点

| 点 | 说明 |
|---|---|
| `getaddrinfo(host, "80", &hints, &res)` | 阻塞式 DNS 解析 |
| `socket(AF_INET, SOCK_STREAM, 0)` | 建 TCP socket |
| `connect / write / read / close` | 标准 BSD API |
| `SO_RCVTIMEO` | 设接收超时，防止 `read` 永久阻塞 |
| `esp_netif_init()` + event_loop | 新风格；旧 WiFi 示例用 `tcpip_adapter_init()` |

> HTTPS 请改用 `components/esp_http_client` + `esp-tls`/mbedTLS，或参考 `examples/protocols/https_mbedtls`、`examples/protocols/https_request`。完整 HTTP 客户端（自动分块/重定向/事件）见 `examples/protocols/esp_http_client`。

## 常见错误

| 错误 | 原因 | 解决方法 |
|---|---|---|
| DNS 失败 | 还没拿到 IP | 等到联网成功再发请求 |
| `connect` 失败 ENODEV | 路由未就绪 | 延迟重试 |
| `read` 卡死 | 无超时 | `setsockopt SO_RCVTIMEO` |
| 栈溢出 panic | 任务栈太小 | 加到 16384 |
| `example_connect` 找不到 | 未加 `common_components` | 复制示例的依赖或自写连 WiFi |
| 想发 HTTPS 失败 | 用 socket 直连 443 | 改用 `esp_http_client`/mbedTLS |

## 参考

- `examples/protocols/http_request/` — 官方 HTTP GET 示例（`main/http_request_example_main.c`）
- `examples/protocols/esp_http_client/` — esp_http_client 高级用法
- `examples/protocols/https_mbedtls/` / `examples/protocols/https_request/` — HTTPS
- `examples/protocols/sockets/` — TCP/UDP/组播 socket 示例
