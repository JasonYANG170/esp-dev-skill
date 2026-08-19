# SmartConfig 一键配网

> **适用摘要**: 用 SmartConfig（Esptouch / AirKiss / Esptouch v2）让 ESP8266 通过手机 App 获取 SSID 与密码并连上 AP；基于 `SC_EVENT` 事件流处理。

> Version: ESP8266 RTOS SDK version used by the project.
> Evidence: `repos/ESP8266_RTOS_SDK/resources/`, source/examples in `repos/ESP8266_RTOS_SDK/`, and this recipe path `repos/ESP8266_RTOS_SDK/recipes/smartconfig.md`.
> Validation: draft metadata added from repository routing; verify APIs, Kconfig symbols, and component dependencies against the selected version.

## 触发意图

- "一键配网"
- "smartconfig"
- "esptouch"
- "AirKiss"
- "手机配网"

## 前置条件

| 条件 | 要求 |
|---|---|
| 参考示例 | `examples/wifi/smart_config/` |
| 配置 | menuconfig 的 `Example Configuration` 选 SmartConfig 类型（`CONFIG_ESP_SMARTCONFIG_TYPE`，由 `Kconfig.projbuild` 提供） |
| 手机 App | EspTouch / EspTouch V2（iOS/Android） |

## 分步说明

### 1. 初始化 WiFi 与事件（改编自示例）

```c
#include "esp_wifi.h"
#include "esp_event.h"
#include "esp_log.h"
#include "esp_smartconfig.h"
#include "smartconfig_ack.h"
#include "nvs_flash.h"
#include "tcpip_adapter.h"
#include "freertos/FreeRTOS.h"
#include "freertos/task.h"
#include "freertos/event_groups.h"

#define EXAMPLE_ESP_SMARTCOFNIG_TYPE  CONFIG_ESP_SMARTCONFIG_TYPE

static const int CONNECTED_BIT = BIT0;
static const int ESPTOUCH_DONE_BIT = BIT1;
static EventGroupHandle_t s_wifi_event_group;
static const char* TAG = "smartconfig_example";
```

### 2. 事件 handler（同时处理 WIFI / IP / SC 三种 base）

```c
static void event_handler(void* arg, esp_event_base_t event_base,
                          int32_t event_id, void* event_data)
{
    if (event_base == WIFI_EVENT && event_id == WIFI_EVENT_STA_START) {
        xTaskCreate(smartconfig_example_task, "smartconfig_task", 4096, NULL, 3, NULL);
    } else if (event_base == WIFI_EVENT && event_id == WIFI_EVENT_STA_DISCONNECTED) {
        esp_wifi_connect();
        xEventGroupClearBits(s_wifi_event_group, CONNECTED_BIT);
    } else if (event_base == IP_EVENT && event_id == IP_EVENT_STA_GOT_IP) {
        xEventGroupSetBits(s_wifi_event_group, CONNECTED_BIT);
    } else if (event_base == SC_EVENT && event_id == SC_EVENT_SCAN_DONE) {
        ESP_LOGI(TAG, "Scan done");
    } else if (event_base == SC_EVENT && event_id == SC_EVENT_FOUND_CHANNEL) {
        ESP_LOGI(TAG, "Found channel");
    } else if (event_base == SC_EVENT && event_id == SC_EVENT_GOT_SSID_PSWD) {
        ESP_LOGI(TAG, "Got SSID and password");
        smartconfig_event_got_ssid_pswd_t* evt = (smartconfig_event_got_ssid_pswd_t*)event_data;
        wifi_config_t wifi_config = {0};
        memcpy(wifi_config.sta.ssid, evt->ssid, sizeof(wifi_config.sta.ssid));
        memcpy(wifi_config.sta.password, evt->password, sizeof(wifi_config.sta.password));
        wifi_config.sta.bssid_set = evt->bssid_set;
        if (wifi_config.sta.bssid_set) {
            memcpy(wifi_config.sta.bssid, evt->bssid, sizeof(wifi_config.sta.bssid));
        }
        if (evt->type == SC_TYPE_ESPTOUCH_V2) {
            uint8_t rvd_data[33] = {0};
            ESP_ERROR_CHECK(esp_smartconfig_get_rvd_data(rvd_data, sizeof(rvd_data)));
            ESP_LOGI(TAG, "RVD_DATA:%s", rvd_data);
        }
        ESP_ERROR_CHECK(esp_wifi_disconnect());
        ESP_ERROR_CHECK(esp_wifi_set_config(ESP_IF_WIFI_STA, &wifi_config));
        ESP_ERROR_CHECK(esp_wifi_connect());
    } else if (event_base == SC_EVENT && event_id == SC_EVENT_SEND_ACK_DONE) {
        xEventGroupSetBits(s_wifi_event_group, ESPTOUCH_DONE_BIT);
    }
}
```

### 3. WiFi 初始化与 smartconfig 任务

```c
static void initialise_wifi(void)
{
    tcpip_adapter_init();
    s_wifi_event_group = xEventGroupCreate();
    ESP_ERROR_CHECK(esp_event_loop_create_default());

    wifi_init_config_t cfg = WIFI_INIT_CONFIG_DEFAULT();
    ESP_ERROR_CHECK(esp_wifi_init(&cfg));
    ESP_ERROR_CHECK(esp_wifi_set_ps(WIFI_PS_NONE));                 // 配网期间关闭省电
    ESP_ERROR_CHECK(esp_event_handler_register(WIFI_EVENT, ESP_EVENT_ANY_ID, &event_handler, NULL));
    ESP_ERROR_CHECK(esp_event_handler_register(IP_EVENT, IP_EVENT_STA_GOT_IP, &event_handler, NULL));
    ESP_ERROR_CHECK(esp_event_handler_register(SC_EVENT, ESP_EVENT_ANY_ID, &event_handler, NULL));

    ESP_ERROR_CHECK(esp_wifi_set_mode(WIFI_MODE_STA));
    ESP_ERROR_CHECK(esp_wifi_start());
}

static void smartconfig_example_task(void* parm)
{
    ESP_ERROR_CHECK(esp_smartconfig_set_type(EXAMPLE_ESP_SMARTCOFNIG_TYPE));
    smartconfig_start_config_t cfg = SMARTCONFIG_START_CONFIG_DEFAULT();
    ESP_ERROR_CHECK(esp_smartconfig_start(&cfg));

    while (1) {
        EventBits_t uxBits = xEventGroupWaitBits(s_wifi_event_group,
                CONNECTED_BIT | ESPTOUCH_DONE_BIT, true, false, portMAX_DELAY);
        if (uxBits & CONNECTED_BIT) ESP_LOGI(TAG, "WiFi Connected to ap");
        if (uxBits & ESPTOUCH_DONE_BIT) {
            ESP_LOGI(TAG, "smartconfig over");
            esp_smartconfig_stop();
            vTaskDelete(NULL);
        }
    }
}

void app_main()
{
    ESP_ERROR_CHECK(nvs_flash_init());
    initialise_wifi();
}
```

### 4. 关键 API

| API | 作用 |
|---|---|
| `esp_smartconfig_set_type(smartconfig_type_t)` | 设类型：`SC_TYPE_ESPTOUCH` / `SC_TYPE_AIRKISS` / `SC_TYPE_ESPTOUCH_V2` / `SC_TYPE_ESPTOUCH_AIRKISS` |
| `esp_smartconfig_start(const smartconfig_start_config_t *)` | 启动配网（用 `SMARTCONFIG_START_CONFIG_DEFAULT()`） |
| `esp_smartconfig_stop(void)` | 完成后停止 |
| `esp_smartconfig_get_rvd_data(buf, len)` | 仅 v2，取附带数据 |

## 常见错误

| 错误 | 原因 | 解决方法 |
|---|---|---|
| App 一直配不上 | 手机与 ESP 不在同一 2.4GHz / 距离远 | 手机连 2.4GHz；靠近设备；关省电 `WIFI_PS_NONE` |
| 拿到 SSID 但连不上 | 密码字符截断 / authmode | 检查 `wifi_config.sta` 填充 |
| 重复配网不停 | 没在 `SEND_ACK_DONE` 里 `stop` | 置 `ESPTOUCH_DONE_BIT` 后 `esp_smartconfig_stop()` |
| `esp_wifi_init` abort | 漏 NVS | 先 `nvs_flash_init()` |
| v2 取不到 rvd_data | 非 v2 类型调用 | 仅 `SC_TYPE_ESPTOUCH_V2` 才取 |

## 参考

- `examples/wifi/smart_config/` — 官方 smartconfig 示例（`main/smartconfig_main.c`、`Kconfig.projbuild`）
- `components/esp8266/include/esp_smartconfig.h` — `esp_smartconfig_*` 原型
- `docs/en/api-reference/wifi/esp_smartconfig.rst`
