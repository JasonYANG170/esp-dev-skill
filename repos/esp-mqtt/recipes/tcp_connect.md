# MQTT over TCP 最小连接

> **适用摘要**: 使用 ESP-MQTT 通过纯 TCP（`mqtt://`）连接 broker，完成 init/register/start 与事件回调的骨架，是所有其它传输方式的基础。

## 触发意图

- "MQTT 连接 broker"
- "esp_mqtt 初始化"
- "MQTT over TCP 示例"
- "怎么启动 mqtt client"

## 前置条件

| 条件 | 要求 |
|---|---|
| ESP-IDF | >= 5.3 |
| 组件依赖 | `espressif/mqtt`（或本地 `mqtt` 目录） |
| 联网 | Wi-Fi 或 Ethernet 已连接（`example_connect()`） |
| 参考示例 | `examples/tcp/` |

## 分步说明

### 1. 声明依赖与 CMake

`idf_component.yml`：
```yaml
dependencies:
  espressif/mqtt: "*"
```

`main/CMakeLists.txt`：
```cmake
idf_component_register(SRCS "app_main.c"
                       PRIV_REQUIRES mqtt nvs_flash esp_netif
                       INCLUDE_DIRS ".")
```

### 2. 启动序列（来自 `examples/tcp/main/app_main.c`）

```c
#include <stdio.h>
#include <inttypes.h>
#include "esp_system.h"
#include "nvs_flash.h"
#include "esp_event.h"
#include "esp_netif.h"
#include "protocol_examples_common.h"
#include "esp_log.h"
#include "mqtt_client.h"

static const char *TAG = "mqtt_example";

static void mqtt_event_handler(void *handler_args, esp_event_base_t base,
                               int32_t event_id, void *event_data)
{
    esp_mqtt_event_handle_t event = event_data;
    esp_mqtt_client_handle_t client = event->client;
    switch ((esp_mqtt_event_id_t)event_id) {
    case MQTT_EVENT_CONNECTED:
        ESP_LOGI(TAG, "MQTT_EVENT_CONNECTED");
        esp_mqtt_client_subscribe(client, "topic/qos0", 0);
        break;
    case MQTT_EVENT_DISCONNECTED:
        ESP_LOGI(TAG, "MQTT_EVENT_DISCONNECTED");
        break;
    case MQTT_EVENT_DATA:
        printf("TOPIC=%.*s\r\n", event->topic_len, event->topic);
        printf("DATA=%.*s\r\n", event->data_len, event->data);
        break;
    default:
        ESP_LOGI(TAG, "Other event id:%d", event->event_id);
        break;
    }
}

static void mqtt_app_start(void)
{
    esp_mqtt_client_config_t mqtt_cfg = {
        .broker.address.uri = CONFIG_BROKER_URL,   // 例如 mqtt://mqtt.eclipseprojects.io
    };
    esp_mqtt_client_handle_t client = esp_mqtt_client_init(&mqtt_cfg);
    esp_mqtt_client_register_event(client, ESP_EVENT_ANY_ID, mqtt_event_handler, NULL);
    esp_mqtt_client_start(client);
}

void app_main(void)
{
    ESP_ERROR_CHECK(nvs_flash_init());
    ESP_ERROR_CHECK(esp_netif_init());
    ESP_ERROR_CHECK(esp_event_loop_create_default());
    ESP_ERROR_CHECK(example_connect());
    mqtt_app_start();
}
```

### 3. 关键点

- `esp_mqtt_client_config_t` 用指定式初始化（designated initializer），未设字段取默认值。
- `ESP_EVENT_ANY_ID` 注册所有 MQTT 事件；事件 base 由客户端内部使用默认事件循环派发。
- 默认端口 1883（`mqtt://`）。

## 常见错误

| 错误 | 原因 | 解决方法 |
|---|---|---|
| 一直无任何事件 | 未建默认事件循环 / 未联网 | 先 `esp_event_loop_create_default()` 与 `example_connect()` |
| 连接即断开 | broker URI 写错或网络不通 | 检查 scheme 与端口，确认联网 |
| 编译找不到 mqtt_client.h | 未声明 mqtt 依赖 | `idf_component.yml` 加 `espressif/mqtt` 或 CMake `PRIV_REQUIRES mqtt` |

## 参考

- `examples/tcp/main/app_main.c`
- `examples/tcp/README.md`
- `docs/en/index.rst`（Broker / Address 节）
