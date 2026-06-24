# 主机 Wi-Fi STA 应用（经 ESP-Hosted）

> **适用摘要**: 在 host 上运行一个标准的 ESP-IDF Wi-Fi Station 应用，底层经 ESP-Hosted RPC 透明转发到协处理器。应用代码与原生 ESP-IDF Wi-Fi 几乎一致——区别只在初始化阶段需要先 `esp_hosted_init()` / `esp_hosted_connect_to_slave()` 并等待传输就绪。

## 触发意图

- "host 上跑 Wi-Fi 连 AP"
- "ESP-Hosted wifi station"
- "iperf over ESP-Hosted"
- "esp_wifi over co-processor"
- "host 连 Wi-Fi"

## 前置条件

| 条件 | 要求 |
|---|---|
| 链路 | 已按 `bringup_spi_fd.md` / `bringup_sdio.md` / `bringup_uart.md` 完成 host+slave 搭建并出现 `Base transport is set-up` |
| 依赖 | host 工程已加 `espressif/esp_wifi_remote` + `espressif/esp_hosted`，已删 `esp-extconn` |
| 参考例程 | `examples/host_hosted_events/main/station_example.c`、`examples/host_network_split__power_save/` |

## 分步说明

### 1. 初始化顺序

```c
#include "esp_log.h"
#include "esp_event.h"
#include "esp_netif.h"
#include "esp_wifi.h"
#include "esp_hosted.h"

static const char *TAG = "host_sta";

void app_main(void)
{
    // 1. netif + 事件循环
    ESP_ERROR_CHECK(esp_netif_init());
    ESP_ERROR_CHECK(esp_event_loop_create_default());

    // 2. 注册 ESP_HOSTED_EVENT 处理（见 host_events_recovery.md）
    //    在 ESP_HOSTED_EVENT_TRANSPORT_UP 时 xSemaphoreGive(sem_hosted_is_up);

    // 3. 拉起 ESP-Hosted
    esp_hosted_init();
    esp_hosted_connect_to_slave();

    // 4. 等待传输就绪
    xSemaphoreTake(sem_hosted_is_up, portMAX_DELAY);
    ESP_LOGI(TAG, "ESP-Hosted is ready");

    // 5. 标准 ESP-IDF Wi-Fi（weak 定义由 esp_wifi_remote/esp_hosted 通过 RPC 实现）
    esp_netif_create_default_wifi_sta();
    wifi_init_config_t cfg = WIFI_INIT_CONFIG_DEFAULT();
    ESP_ERROR_CHECK(esp_wifi_init(&cfg));

    // 6. 注册 WIFI_EVENT / IP_EVENT 处理（与原生 ESP-IDF 相同）
    //    见下方事件处理示例
}
```

### 2. STA 配置与连接（节选自 `station_example.c`）

```c
#define EXAMPLE_ESP_WIFI_SSID      CONFIG_EXAMPLE_WIFI_SSID
#define EXAMPLE_ESP_WIFI_PASS      CONFIG_EXAMPLE_WIFI_PASSWORD
#define ESP_WIFI_SCAN_AUTH_MODE_THRESHOLD  WIFI_AUTH_WPA2_PSK

static int s_retry_num = 0;

static void event_handler(void *arg, esp_event_base_t event_base,
                          int32_t event_id, void *event_data)
{
    if (event_base == WIFI_EVENT && event_id == WIFI_EVENT_STA_START) {
        esp_wifi_connect();
    } else if (event_base == WIFI_EVENT &&
               event_id == WIFI_EVENT_STA_DISCONNECTED) {
        if (s_retry_num < CONFIG_EXAMPLE_WIFI_CONN_MAX_RETRY) {
            esp_wifi_connect();
            s_retry_num++;
        }
    } else if (event_base == IP_EVENT && event_id == IP_EVENT_STA_GOT_IP) {
        ip_event_got_ip_t *event = (ip_event_got_ip_t *)event_data;
        ESP_LOGI(TAG, "got ip:" IPSTR, IP2STR(&event->ip_info.ip));
        s_retry_num = 0;
        // 通知连接成功
    }
}

void example_wifi_init_sta(void)
{
    wifi_config_t wifi_config = {
        .sta = {
            .ssid = EXAMPLE_ESP_WIFI_SSID,
            .password = EXAMPLE_ESP_WIFI_PASS,
            .threshold.authmode = ESP_WIFI_SCAN_AUTH_MODE_THRESHOLD,
            .sae_pwe_h2e = WPA3_SAE_PWE_HUNT_AND_PECK,
        },
    };
    ESP_ERROR_CHECK(esp_wifi_set_mode(WIFI_MODE_STA));
    ESP_ERROR_CHECK(esp_wifi_set_config(WIFI_IF_STA, &wifi_config));
    ESP_ERROR_CHECK(esp_wifi_start());
}
```

> 上述 Wi-Fi API（`esp_wifi_init`/`set_mode`/`set_config`/`start`/`connect`）签名与原生 ESP-IDF 完全一致；它们的 RPC 实现（弱定义）在 `host/api/src/esp_wifi_weak.c`，对应的 RPC 命令（WifiInit/WifiStart/WifiConnect/...）见 `docs/implemented_rpcs.md`。

### 3. 高性能配置（P4 + C6，可选）

将下列项加入 `sdkconfig.defaults.esp32p4`（其他 host 合并到对应 `sdkconfig.defaults.<target>`）：

```text
# Wi-Fi Remote 缓冲与聚合
CONFIG_WIFI_RMT_STATIC_RX_BUFFER_NUM=16
CONFIG_WIFI_RMT_DYNAMIC_RX_BUFFER_NUM=64
CONFIG_WIFI_RMT_DYNAMIC_TX_BUFFER_NUM=64
CONFIG_WIFI_RMT_AMPDU_TX_ENABLED=y
CONFIG_WIFI_RMT_TX_BA_WIN=32
CONFIG_WIFI_RMT_AMPDU_RX_ENABLED=y
CONFIG_WIFI_RMT_RX_BA_WIN=32

# lwIP
CONFIG_LWIP_TCP_SND_BUF_DEFAULT=65534
CONFIG_LWIP_TCP_WND_DEFAULT=65534
CONFIG_LWIP_TCP_RECVMBOX_SIZE=64
CONFIG_LWIP_UDP_RECVMBOX_SIZE=64
CONFIG_LWIP_TCPIP_RECVMBOX_SIZE=64
CONFIG_LWIP_TCP_SACK_OUT=y
```

调整路径：`idf.py menuconfig -> Component config -> Wi-Fi Remote -> Wi-Fi configuration`。更多优化见 `docs/performance_optimization.md`。

### 4. 验证（iperf）

```
sta_scan
sta_connect <SSID> <password>
sta_ip
iperf -u -c <STA_IP> -t 60 -i 3     # Host TX UDP
iperf -u -s -i 3                    # Host RX UDP（对端发）
```

## 常见错误

| 错误 | 原因 | 解决方法 |
|---|---|---|
| `esp_wifi_init` 失败 | 未先 `esp_hosted_init`/connect 或未等 TRANSPORT_UP | 严格按初始化顺序，等传输就绪再调 Wi-Fi |
| 连接前应 disable native Wi-Fi | host 自带 Wi-Fi 与 esp_wifi_remote 冲突 | 关闭 host 原生 Wi-Fi caps |
| `esp-extconn` 仍在 | 与 esp_hosted 冲突 | 删除 main/idf_component.yml 中的 esp-extconn 块 |
| 连不上 AP | 协处理器射频未就绪 | 确认 slave 已正确烧录且 `Transport used :: ...` 已打印 |
| WPA3/SAE 失败 | 鉴权阈值/H2E 配置不当 | 参考例程的 `threshold.authmode` 与 `sae_pwe_h2e` 设置 |

## 参考

- `examples/host_hosted_events/main/station_example.c`（STA 连接完整流程）
- `examples/host_network_split__power_save/`（含 iperf、network split、低功耗）
- `docs/performance_optimization.md`
- `docs/implemented_rpcs.md`（Wi-Fi RPC 命令清单）
