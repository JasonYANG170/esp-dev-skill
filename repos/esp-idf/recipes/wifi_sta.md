# Wi-Fi STA 连接

> **适用摘要**: 以 Station 模式连接 AP，含事件循环注册、`esp_wifi_start`、连接成功判定（`IP_EVENT_STA_GOT_IP`）与重连（适配自 getting_started/station）。

## 触发意图

- "WiFi 连网"
- "STA 模式"
- "连路由器"
- "获取 IP 地址"
- "esp_wifi_connect"

## 前置条件

| 条件 | 要求 |
|---|---|
| 组件 | `esp_wifi`、`esp_event`、`esp_netif`、`nvs_flash` |
| 头文件 | `esp_wifi.h`、`esp_event.h`、`esp_netif.h`、`nvs_flash.h` |
| 参考 | `examples/wifi/getting_started/station` |

## 分步说明

### 完整初始化流程（适配自 station_example_main.c）

```c
#include <string.h>
#include "esp_wifi.h"
#include "esp_event.h"
#include "esp_netif.h"
#include "esp_log.h"
#include "nvs_flash.h"
#include "freertos/event_groups.h"

#define WIFI_SSID       "myssid"
#define WIFI_PASS       "mypassword"
#define MAX_RETRY       5

static EventGroupHandle_t s_wifi_eg;
#define CONNECTED_BIT  BIT0
#define FAIL_BIT       BIT1
static int s_retry = 0;
static const char *TAG = "wifi";

static void on_event(void *arg, esp_event_base_t base, int32_t id, void *data)
{
    if (base == WIFI_EVENT && id == WIFI_EVENT_STA_START) {
        esp_wifi_connect();
    } else if (base == WIFI_EVENT && id == WIFI_EVENT_STA_DISCONNECTED) {
        if (s_retry < MAX_RETRY) {
            esp_wifi_connect();
            s_retry++;
            ESP_LOGI(TAG, "retry connect");
        } else {
            xEventGroupSetBits(s_wifi_eg, FAIL_BIT);
        }
    } else if (base == IP_EVENT && id == IP_EVENT_STA_GOT_IP) {
        ip_event_got_ip_t *e = (ip_event_got_ip_t *)data;
        ESP_LOGI(TAG, "got ip:" IPSTR, IP2STR(&e->ip_info.ip));
        s_retry = 0;
        xEventGroupSetBits(s_wifi_eg, CONNECTED_BIT);
    }
}

static void wifi_init_sta(void)
{
    s_wifi_eg = xEventGroupCreate();

    ESP_ERROR_CHECK(esp_netif_init());
    ESP_ERROR_CHECK(esp_event_loop_create_default());
    esp_netif_create_default_wifi_sta();

    wifi_init_config_t cfg = WIFI_INIT_CONFIG_DEFAULT();
    ESP_ERROR_CHECK(esp_wifi_init(&cfg));

    esp_event_handler_instance_t any_id, got_ip;
    ESP_ERROR_CHECK(esp_event_handler_instance_register(WIFI_EVENT, ESP_EVENT_ANY_ID,
                                                        &on_event, NULL, &any_id));
    ESP_ERROR_CHECK(esp_event_handler_instance_register(IP_EVENT, IP_EVENT_STA_GOT_IP,
                                                        &on_event, NULL, &got_ip));

    wifi_config_t wc = {
        .sta = {
            .ssid = WIFI_SSID,
            .password = WIFI_PASS,
            .threshold.authmode = WIFI_AUTH_WPA2_PSK,
        },
    };
    ESP_ERROR_CHECK(esp_wifi_set_mode(WIFI_MODE_STA));
    ESP_ERROR_CHECK(esp_wifi_set_config(WIFI_IF_STA, &wc));
    ESP_ERROR_CHECK(esp_wifi_start());

    /* 等待连接成功或失败 */
    EventBits_t bits = xEventGroupWaitBits(s_wifi_eg,
            CONNECTED_BIT | FAIL_BIT, pdFALSE, pdFALSE, portMAX_DELAY);
    if (bits & CONNECTED_BIT) ESP_LOGI(TAG, "connected");
    else                       ESP_LOGE(TAG, "failed");
}

void app_main(void)
{
    /* NVS 必须先初始化 */
    esp_err_t ret = nvs_flash_init();
    if (ret == ESP_ERR_NVS_NO_FREE_PAGES || ret == ESP_ERR_NVS_NEW_VERSION_FOUND) {
        ESP_ERROR_CHECK(nvs_flash_erase());
        ret = nvs_flash_init();
    }
    ESP_ERROR_CHECK(ret);

    wifi_init_sta();
}
```

### 关键 API

```c
esp_err_t esp_netif_init(void);
esp_err_t esp_event_loop_create_default(void);
esp_netif_t* esp_netif_create_default_wifi_sta(void);
esp_err_t esp_wifi_init(const wifi_init_config_t *config);
esp_err_t esp_wifi_set_mode(wifi_mode_t mode);
esp_err_t esp_wifi_set_config(wifi_interface_t interface, wifi_config_t *conf);
esp_err_t esp_wifi_start(void);
esp_err_t esp_wifi_connect(void);
esp_err_t esp_event_handler_instance_register(esp_event_base_t event_base,
                                              int32_t event_id,
                                              esp_event_handler_t event_handler,
                                              void *arg,
                                              esp_event_handler_instance_t *instance);
```

关键事件：`WIFI_EVENT_STA_START` → 调 `esp_wifi_connect()`；`WIFI_EVENT_STA_DISCONNECTED` → 重连；`IP_EVENT_STA_GOT_IP` → 已联网。

## 常见错误

| 错误 | 原因 | 解决方法 |
|---|---|---|
| 不触发 `STA_START` | 未建默认 netif/事件循环 | 先 `esp_netif_init` + `esp_event_loop_create_default` + `create_default_wifi_sta` |
| `nvs_flash_init` 失败致 wifi 崩 | NVS 未就绪 | 处理 `NO_FREE_PAGES`/`NEW_VERSION_FOUND` 后重试 |
| 连不上 AP | SSID/密码错或信号差 | 查 `WIFI_EVENT_STA_DISCONNECTED` 的 reason；靠近 AP |
| 一直重连 | 密码错但无限重试 | 加 `MAX_RETRY` 上限后置 `FAIL_BIT` |
| 拿不到 IP | 路由器 DHCP 关 | 检查路由；或改静态 IP |

## 参考

- `examples/wifi/getting_started/station` — 完整 STA 连接（含 WPA3 选项）
- `examples/wifi/scan` — 扫描周围 AP
- ESP-IDF `components/esp_wifi/include/esp_wifi.h`、`components/esp_event/include/esp_event.h`
