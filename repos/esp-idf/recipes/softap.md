# Wi-Fi SoftAP 热点

> **适用摘要**: 以 SoftAP 模式创建 Wi-Fi 热点，配置 SSID/密码/认证/最大连接数，处理 STA 接入事件（适配自 getting_started/softAP）。

> Version: ESP-IDF version used by the project.
> Evidence: `repos/esp-idf/resources/`, source/examples in `repos/esp-idf/`, and this recipe path `repos/esp-idf/recipes/softap.md`.
> Validation: draft metadata added from repository routing; verify APIs, Kconfig symbols, and component dependencies against the selected version.

## 触发意图

- "WiFi 热点"
- "SoftAP 模式"
- "开 AP"
- "配网热点"
- "WIFI_MODE_AP"

## 前置条件

| 条件 | 要求 |
|---|---|
| 组件 | `esp_wifi`、`esp_event`、`esp_netif`、`nvs_flash` |
| 头文件 | `esp_wifi.h`、`esp_event.h`、`esp_netif.h` |
| 参考 | `examples/wifi/getting_started/softAP` |

## 分步说明

### 初始化热点（适配自 softap_example_main.c）

```c
#include "esp_wifi.h"
#include "esp_event.h"
#include "esp_netif.h"
#include "esp_log.h"
#include "nvs_flash.h"

#define AP_SSID         "myap"
#define AP_PASS         "12345678"
#define AP_CHANNEL      1
#define AP_MAX_CONN     4

static void on_event(void *arg, esp_event_base_t base, int32_t id, void *data)
{
    if (id == WIFI_EVENT_AP_STACONNECTED) {
        wifi_event_ap_staconnected_t *e = (wifi_event_ap_staconnected_t *)data;
        ESP_LOGI("ap", "station join mac=%02x:%02x:%02x:%02x:%02x:%02x",
                 e->mac[0], e->mac[1], e->mac[2], e->mac[3], e->mac[4], e->mac[5]);
    } else if (id == WIFI_EVENT_AP_STADISCONNECTED) {
        wifi_event_ap_stadisconnected_t *e = (wifi_event_ap_stadisconnected_t *)data;
        ESP_LOGI("ap", "station leave mac=%02x:%02x:%02x:%02x:%02x:%02x",
                 e->mac[0], e->mac[1], e->mac[2], e->mac[3], e->mac[4], e->mac[5]);
    }
}

static void wifi_init_softap(void)
{
    ESP_ERROR_CHECK(esp_netif_init());
    ESP_ERROR_CHECK(esp_event_loop_create_default());
    esp_netif_create_default_wifi_ap();

    wifi_init_config_t cfg = WIFI_INIT_CONFIG_DEFAULT();
    ESP_ERROR_CHECK(esp_wifi_init(&cfg));

    ESP_ERROR_CHECK(esp_event_handler_instance_register(WIFI_EVENT,
                                                        ESP_EVENT_ANY_ID, &on_event, NULL, NULL));

    wifi_config_t wc = {
        .ap = {
            .ssid = AP_SSID,
            .ssid_len = strlen(AP_SSID),
            .channel = AP_CHANNEL,
            .password = AP_PASS,
            .max_connection = AP_MAX_CONN,
            .authmode = WIFI_AUTH_WPA2_PSK,
        },
    };
    /* 无密码则降级 OPEN */
    if (strlen(AP_PASS) == 0) wc.ap.authmode = WIFI_AUTH_OPEN;

    ESP_ERROR_CHECK(esp_wifi_set_mode(WIFI_MODE_AP));
    ESP_ERROR_CHECK(esp_wifi_set_config(WIFI_IF_AP, &wc));
    ESP_ERROR_CHECK(esp_wifi_start());
}

void app_main(void)
{
    esp_err_t ret = nvs_flash_init();
    if (ret == ESP_ERR_NVS_NO_FREE_PAGES || ret == ESP_ERR_NVS_NEW_VERSION_FOUND) {
        ESP_ERROR_CHECK(nvs_flash_erase());
        ret = nvs_flash_init();
    }
    ESP_ERROR_CHECK(ret);
    wifi_init_softap();
}
```

### 关键事件

- `WIFI_EVENT_AP_START` — AP 已启动
- `WIFI_EVENT_AP_STACONNECTED` — 有 STA 接入（含 MAC）
- `WIFI_EVENT_AP_STADISCONNECTED` — STA 离开

### 关键 API

```c
esp_netif_t* esp_netif_create_default_wifi_ap(void);
esp_err_t esp_wifi_set_mode(wifi_mode_t mode);          /* WIFI_MODE_AP / WIFI_MODE_APSTA */
esp_err_t esp_wifi_set_config(wifi_interface_t interface, wifi_config_t *conf); /* WIFI_IF_AP */
```

> STA + AP 同时工作用 `WIFI_MODE_APSTA`；详见 `examples/wifi/softap_sta`。

## 常见错误

| 错误 | 原因 | 解决方法 |
|---|---|---|
| 用 `create_default_wifi_sta` 当 AP | netif 类型不符 | AP 用 `esp_netif_create_default_wifi_ap()` |
| 密码<8 位连不上 | WPA2 要求 ≥8 字符 | 用 ≥8 位密码或设 `WIFI_AUTH_OPEN` |
| 手机搜不到 | 信道/频段 | 选 1~13（2.4G）；确认未启用隐藏 SSID |
| 最大连接超限 | `max_connection` 太小 | 增大 `.max_connection`（芯片上限内） |
| DHCP 不分配 IP | 未启用默认 | `create_default_wifi_ap` 已默认启 DHCP；勿再手动禁 |

## 参考

- `examples/wifi/getting_started/softAP` — 完整 SoftAP
- `examples/wifi/softap_sta` — STA + AP 同存
- ESP-IDF `components/esp_wifi/include/esp_wifi.h`
