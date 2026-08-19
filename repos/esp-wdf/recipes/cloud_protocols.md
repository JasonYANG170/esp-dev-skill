# 云服务与配网（HTTP / MQTT / RainMaker / Wi-Fi Provisioning）

> **适用摘要**：WASM 应用通过 ESP-WDF 扩展适配层使用 ESP-IDF 风格 API：ESP HTTP Client 发起请求、ESP-MQTT 收发消息、ESP-RainMaker 上报设备、Wi-Fi Provisioning 配网。各模块由对应 Kconfig 开关启用（默认均为 `y`）。

> Evidence: `repos/esp-wdf/resources/`, source/examples in `repos/esp-wdf/`, and this recipe path `repos/esp-wdf/recipes/cloud_protocols.md`.
> Validation: draft metadata added from repository routing; verify APIs, Kconfig symbols, and component dependencies against the selected version.

## 触发意图

- "WASM HTTP 请求"
- "MQTT 客户端"
- "RainMaker 设备"
- "Wi-Fi 配网"

## 前置条件

| 模块 | Kconfig | 头文件（应用层） | 参考示例 |
|---|---|---|---|
| HTTP Client | `CONFIG_WDF_EXT_WASM_APP_HTTP_CLIENT=y` | `esp_http_client.h` | `examples/protocols/esp_http_client/` |
| MQTT | `CONFIG_WDF_EXT_WASM_APP_MQTT=y` | `mqtt_client.h` | `examples/protocols/mqtt/generic/` |
| RainMaker | `CONFIG_WDF_EXT_WASM_APP_RMAKER=y` | `esp_rmaker_core.h` 等 | `examples/rainmaker/switch/` |
| Wi-Fi Provisioning | `CONFIG_WDF_EXT_WASM_APP_WIFI_PROVISIONING=y` | 配网 API | `examples/provisioning/wifi_prov_mgr/` |

## 分步说明

### 1. MQTT（取自 examples/protocols/mqtt/generic 要点）

```c
#include <stdio.h>
#include "sdkconfig.h"
#include "mqtt_client.h"

/* 事件处理回调（节选） */
static void mqtt_event_handler(void *handler_args, esp_event_base_t base,
                               int32_t event_id, void *event_data)
{
    esp_mqtt_event_handle_t event = event_data;
    esp_mqtt_client_handle_t client = event->client;
    switch ((esp_mqtt_event_id_t)event_id) {
        case MQTT_EVENT_CONNECTED:
            esp_mqtt_client_subscribe(client, "/topic/qos0", 0);
            esp_mqtt_client_subscribe(client, "/topic/qos1", 1);
            esp_mqtt_client_publish(client, "/topic/qos0", "data", 4, 0, 0);
            break;
        case MQTT_EVENT_DATA:
            printf("TOPIC=%.*s\r\n", event->topic_len, event->topic);
            printf("DATA=%.*s\r\n", event->data_len, event->data);
            break;
        default: break;
    }
}
```

证书以嵌入二进制方式提供（`_binary_<name>_pem_start/end`），示例用 `mqtt_eclipseprojects_io.pem`。

### 2. ESP-RainMaker 开关（取自 examples/rainmaker/switch 要点）

```c
#include <esp_rmaker_core.h>
#include <esp_rmaker_standard_types.h>
#include <esp_rmaker_standard_params.h>
#include <esp_rmaker_standard_devices.h>
#include "app_wifi.h"

/* 云端写回调 */
static esp_err_t write_cb(const esp_rmaker_device_t *device, const esp_rmaker_param_t *param,
            const esp_rmaker_param_val_t val, void *priv_data, esp_rmaker_write_ctx_t *ctx)
{
    if (strcmp(esp_rmaker_param_get_name(param), ESP_RMAKER_DEF_POWER_NAME) == 0) {
        /* 回写并上报新状态 */
        esp_rmaker_param_update_and_report(param, val);
    }
    return ESP_OK;
}

int main(int argc, char *argv[])
{
    app_wifi_init();                 /* 先初始化 Wi-Fi */
    /* esp_rmaker_node_init 须在 app_wifi_start 之前 */
    /* 注册标准 Switch 设备、绑定 write_cb、esp_rmaker_start、app_wifi_start */
    return 0;
}
```

关键顺序：`app_wifi_init()` → `esp_rmaker_node_init()` → 注册设备/参数/回调 → `esp_rmaker_start()` → `app_wifi_start()`。

### 3. ESP HTTP Client 与 Wi-Fi Provisioning

- HTTP Client 示例（`examples/protocols/esp_http_client`）演示 GET/POST、自定义 header、TLS 证书（`howsmyssl_com_root_cert.pem`、`postman_root_cert.pem`）。
- 配网示例（`examples/provisioning/wifi_prov_mgr`）演示 softap/ble 二维码配网流程。

应用层调用 `esp_http_client_*` 系列；底层由 `http_client_wasm_api.h` 定义的函数 ID（`HTTP_CLIENT_INIT/PERFORM/CLOSE/SET_URL/...`）桥接到宿主。

## 常见错误

| 错误 | 原因 | 解决方法 |
|---|---|---|
| 模块 API 未定义 | 对应 Kconfig 关闭 | 开启 `WDF_EXT_WASM_APP_MQTT/HTTP_CLIENT/RMAKER/WIFI_PROVISIONING` |
| RainMaker 启动失败 | 初始化顺序错 | `app_wifi_init` → `esp_rmaker_node_init` → `esp_rmaker_start` → `app_wifi_start` |
| MQTT 连不上 broker | 证书/URI 错 | 嵌入正确 PEM；menuconfig 设 broker URI |
| HTTP 请求无响应 | 未 `perform`/未处理事件 | `esp_http_client_perform` 后读状态码 |

## 参考

- `examples/protocols/mqtt/generic/main/app_main.c`
- `examples/protocols/esp_http_client/main/esp_http_client_example.c`
- `examples/rainmaker/switch/main/switch.c`
- `examples/provisioning/wifi_prov_mgr/main/wifi_prov_mgr.c`
- `components/extended_wasm_app/esp_http_client/private_include/http_client_wasm_api.h`
- `components/extended_wasm_app/Kconfig`
- `resources/api_reference.md` —— 第 5.2、5.3、5.4 节
