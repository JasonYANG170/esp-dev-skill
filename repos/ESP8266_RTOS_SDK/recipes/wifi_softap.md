# WiFi SoftAP 热点

> **适用摘要**: 把 ESP8266 配为 SoftAP 开放热点，监听 station 的连接/离开事件，设置最大连接数与认证模式。

## 触发意图

- "ESP8266 做热点"
- "WiFi AP / SoftAP"
- "SoftAP 模式"
- "开放热点"

## 前置条件

| 条件 | 要求 |
|---|---|
| 参考示例 | `examples/wifi/getting_started/softAP/` |
| 配置 | menuconfig 的 `Example Configuration` 填 SSID / 密码 / 最大连接数（`Kconfig.projbuild`） |

## 分步说明

### 1. app_main

```c
#include "nvs_flash.h"
#include "esp_wifi.h"
#include "esp_event.h"
#include "esp_log.h"

void app_main()
{
    ESP_ERROR_CHECK(nvs_flash_init());
    ESP_LOGI("wifi softAP", "ESP_WIFI_MODE_AP");
    wifi_init_softap();
}
```

### 2. wifi_init_softap 完整实现（改编自示例）

```c
#define EXAMPLE_ESP_WIFI_SSID      CONFIG_ESP_WIFI_SSID
#define EXAMPLE_ESP_WIFI_PASS      CONFIG_ESP_WIFI_PASSWORD
#define EXAMPLE_MAX_STA_CONN       CONFIG_ESP_MAX_STA_CONN

static const char *TAG = "wifi softAP";

static void wifi_event_handler(void* arg, esp_event_base_t event_base,
                               int32_t event_id, void* event_data)
{
    if (event_id == WIFI_EVENT_AP_STACONNECTED) {
        wifi_event_ap_staconnected_t* event = (wifi_event_ap_staconnected_t*) event_data;
        ESP_LOGI(TAG, "station "MACSTR" join, AID=%d", MAC2STR(event->mac), event->aid);
    } else if (event_id == WIFI_EVENT_AP_STADISCONNECTED) {
        wifi_event_ap_stadisconnected_t* event = (wifi_event_ap_stadisconnected_t*) event_data;
        ESP_LOGI(TAG, "station "MACSTR" leave, AID=%d", MAC2STR(event->mac), event->aid);
    }
}

void wifi_init_softap()
{
    tcpip_adapter_init();
    ESP_ERROR_CHECK(esp_event_loop_create_default());

    wifi_init_config_t cfg = WIFI_INIT_CONFIG_DEFAULT();
    ESP_ERROR_CHECK(esp_wifi_init(&cfg));

    ESP_ERROR_CHECK(esp_event_handler_register(WIFI_EVENT, ESP_EVENT_ANY_ID, &wifi_event_handler, NULL));

    wifi_config_t wifi_config = {
        .ap = {
            .ssid = EXAMPLE_ESP_WIFI_SSID,
            .ssid_len = strlen(EXAMPLE_ESP_WIFI_SSID),
            .password = EXAMPLE_ESP_WIFI_PASS,
            .max_connection = EXAMPLE_MAX_STA_CONN,
            .authmode = WIFI_AUTH_WPA_WPA2_PSK
        },
    };
    if (strlen(EXAMPLE_ESP_WIFI_PASS) == 0) {
        wifi_config.ap.authmode = WIFI_AUTH_OPEN;            // 无密码 → 开放
    }

    ESP_ERROR_CHECK(esp_wifi_set_mode(WIFI_MODE_AP));
    ESP_ERROR_CHECK(esp_wifi_set_config(ESP_IF_WIFI_AP, &wifi_config));
    ESP_ERROR_CHECK(esp_wifi_start());

    ESP_LOGI(TAG, "wifi_init_softap finished. SSID:%s", EXAMPLE_ESP_WIFI_SSID);
}
```

### 3. 关键 API / 配置项

| 项 | 说明 |
|---|---|
| `WIFI_MODE_AP` | SoftAP 模式（APSTA 则用 `WIFI_MODE_APSTA`） |
| `wifi_config.ap.ssid` | SSID（≤32 字节） |
| `wifi_config.ap.max_connection` | 最大 station 数 |
| `wifi_config.ap.authmode` | `WIFI_AUTH_OPEN`（无密码）/ `WIFI_AUTH_WPA_WPA2_PSK` 等 |
| `WIFI_EVENT_AP_STACONNECTED` / `WIFI_EVENT_AP_STADISCONNECTED` | station 上下线事件 |
| `MAC2STR(mac)` / `MACSTR` | 打印 MAC 的便捷宏（6 字节） |

## 常见错误

| 错误 | 原因 | 解决方法 |
|---|---|---|
| 密码小于 8 位被拒 | WPA 要求 ≥8 位 | 加长密码或改 `WIFI_AUTH_OPEN` |
| `esp_wifi_init` abort | 漏 NVS | 先 `nvs_flash_init()` |
| 手机搜不到热点 | SSID 隐藏 / 信道 | 检查 SSID 长度、用 2.4GHz 手机 |
| 连上拿不到 IP | 漏 `tcpip_adapter_init()` | 补 tcpip_adapter / event_loop |
| `max_connection` 不生效 | 数值超过底层限制 | 用合理值（通常 ≤8） |

## 参考

- `examples/wifi/getting_started/softAP/` — 官方 softAP 示例（`main/softap_example_main.c`）
- `docs/en/api-reference/wifi/esp_wifi.rst`
- `resources/api_reference.md`
