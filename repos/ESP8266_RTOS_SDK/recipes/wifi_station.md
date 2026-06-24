# WiFi Station 连接 AP

> **适用摘要**: 把 ESP8266 配为 Station 连接到指定 AP，基于 esp_event 处理连接/断开/拿 IP，带最大重试次数，用事件组阻塞等待联网结果。

## 触发意图

- "ESP8266 连 WiFi"
- "WiFi station"
- "连 AP 拿 IP"
- "STA 模式"

## 前置条件

| 条件 | 要求 |
|---|---|
| 参考示例 | `examples/wifi/getting_started/station/` |
| 分区表 | 默认 "Single factory app" 即可（含 nvs 分区） |
| 配置 | menuconfig 的 `Example Configuration` 里填 SSID / 密码 / 最大重试（由 `Kconfig.projbuild` 提供） |

## 分步说明

### 1. 初始化顺序（app_main）

```c
#include "nvs_flash.h"
#include "esp_wifi.h"
#include "esp_event.h"
#include "esp_log.h"
#include "freertos/FreeRTOS.h"
#include "freertos/task.h"
#include "freertos/event_groups.h"
#include "lwip/sys.h"

void app_main()
{
    // 1. NVS（WiFi 前必做）
    ESP_ERROR_CHECK(nvs_flash_init());

    // 2. 初始化 WiFi（内部已 tcpip_adapter_init + event_loop_create_default）
    wifi_init_sta();
}
```

### 2. wifi_init_sta 完整实现（改编自示例）

```c
#define EXAMPLE_ESP_WIFI_SSID      CONFIG_ESP_WIFI_SSID
#define EXAMPLE_ESP_WIFI_PASS      CONFIG_ESP_WIFI_PASSWORD
#define EXAMPLE_ESP_MAXIMUM_RETRY  CONFIG_ESP_MAXIMUM_RETRY

#define WIFI_CONNECTED_BIT BIT0
#define WIFI_FAIL_BIT      BIT1

static EventGroupHandle_t s_wifi_event_group;
static int s_retry_num = 0;
static const char *TAG = "wifi station";

static void event_handler(void* arg, esp_event_base_t event_base,
                          int32_t event_id, void* event_data)
{
    if (event_base == WIFI_EVENT && event_id == WIFI_EVENT_STA_START) {
        esp_wifi_connect();                                  // 启动后开始连接
    } else if (event_base == WIFI_EVENT && event_id == WIFI_EVENT_STA_DISCONNECTED) {
        if (s_retry_num < EXAMPLE_ESP_MAXIMUM_RETRY) {       // 带计数重连
            esp_wifi_connect();
            s_retry_num++;
            ESP_LOGI(TAG, "retry to connect to the AP");
        } else {
            xEventGroupSetBits(s_wifi_event_group, WIFI_FAIL_BIT);
        }
    } else if (event_base == IP_EVENT && event_id == IP_EVENT_STA_GOT_IP) {
        ip_event_got_ip_t* event = (ip_event_got_ip_t*) event_data;
        ESP_LOGI(TAG, "got ip:%s", ip4addr_ntoa(&event->ip_info.ip));
        s_retry_num = 0;
        xEventGroupSetBits(s_wifi_event_group, WIFI_CONNECTED_BIT);
    }
}

void wifi_init_sta(void)
{
    s_wifi_event_group = xEventGroupCreate();

    tcpip_adapter_init();
    ESP_ERROR_CHECK(esp_event_loop_create_default());

    wifi_init_config_t cfg = WIFI_INIT_CONFIG_DEFAULT();
    ESP_ERROR_CHECK(esp_wifi_init(&cfg));

    ESP_ERROR_CHECK(esp_event_handler_register(WIFI_EVENT, ESP_EVENT_ANY_ID, &event_handler, NULL));
    ESP_ERROR_CHECK(esp_event_handler_register(IP_EVENT, IP_EVENT_STA_GOT_IP, &event_handler, NULL));

    wifi_config_t wifi_config = {
        .sta = {
            .ssid = EXAMPLE_ESP_WIFI_SSID,
            .password = EXAMPLE_ESP_WIFI_PASS,
        },
    };
    if (strlen((char *)wifi_config.sta.password)) {
        wifi_config.sta.threshold.authmode = WIFI_AUTH_WPA2_PSK;   // 要求 WPA2 以上
    }

    ESP_ERROR_CHECK(esp_wifi_set_mode(WIFI_MODE_STA));
    ESP_ERROR_CHECK(esp_wifi_set_config(ESP_IF_WIFI_STA, &wifi_config));
    ESP_ERROR_CHECK(esp_wifi_start());

    // 阻塞等待 CONNECTED 或 FAIL
    EventBits_t bits = xEventGroupWaitBits(s_wifi_event_group,
            WIFI_CONNECTED_BIT | WIFI_FAIL_BIT, pdFALSE, pdFALSE, portMAX_DELAY);

    if (bits & WIFI_CONNECTED_BIT) {
        ESP_LOGI(TAG, "connected to ap SSID:%s", EXAMPLE_ESP_WIFI_SSID);
    } else if (bits & WIFI_FAIL_BIT) {
        ESP_LOGI(TAG, "Failed to connect to SSID:%s", EXAMPLE_ESP_WIFI_SSID);
    }

    ESP_ERROR_CHECK(esp_event_handler_unregister(IP_EVENT, IP_EVENT_STA_GOT_IP, &event_handler));
    ESP_ERROR_CHECK(esp_event_handler_unregister(WIFI_EVENT, ESP_EVENT_ANY_ID, &event_handler));
    vEventGroupDelete(s_wifi_event_group);
}
```

### 3. 关键 API

| API | 作用 |
|---|---|
| `esp_wifi_init(const wifi_init_config_t *)` | 初始化 WiFi 底层 |
| `esp_wifi_set_mode(wifi_mode_t)` | 设 `WIFI_MODE_STA` |
| `esp_wifi_set_config(ESP_IF_WIFI_STA, &wifi_config)` | 配 STA 的 SSID/密码 |
| `esp_wifi_start()` | 启动，触发 `WIFI_EVENT_STA_START` |
| `esp_wifi_connect()` | 关联 AP（成功后由 `IP_EVENT_STA_GOT_IP` 通知拿 IP） |

## 常见错误

| 错误 | 原因 | 解决方法 |
|---|---|---|
| `esp_wifi_init` abort | 漏 `nvs_flash_init()` | app_main 第一步先 NVS |
| 一直连不上 | SSID/密码错 / authmode 过高 | 检查密码；降低 `threshold.authmode` |
| 拿不到 IP | 漏 `tcpip_adapter_init()` + event_loop | 按顺序补齐 |
| 断线不重连 | 没在 `STA_DISCONNECTED` 里 `esp_wifi_connect()` | 加带计数重连逻辑 |
| 连上但 socket 失败 | 在 connect 返回后就发请求 | 等 `IP_EVENT_STA_GOT_IP` 再发 |
| SSID 是中文/特殊字符 | SSID 数组不够大 | `wifi_config.sta.ssid` 32 字节，注意结尾 |

## 参考

- `examples/wifi/getting_started/station/` — 官方 station 示例（`main/station_example_main.c`、`Kconfig.projbuild`）
- `docs/en/api-reference/wifi/esp_wifi.rst` — WiFi 概述
- `resources/api_reference.md` — WiFi 函数签名速查
