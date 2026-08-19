# 事件回调完整骨架

> **适用摘要**: 处理 ESP-MQTT 全部事件类型，包括 CONNECTED/DISCONNECTED/SUBSCRIBED/UNSUBSCRIBED/PUBLISHED/DATA/ERROR，并解析错误句柄。

> Version: selected repo/component version; align with the user project and dependency manifest.
> Evidence: `repos/esp-mqtt/resources/`, source/examples in `repos/esp-mqtt/`, and this recipe path `repos/esp-mqtt/recipes/event_handling.md`.
> Validation: draft metadata added from repository routing; verify APIs, Kconfig symbols, and component dependencies against the selected version.

## 触发意图

- "MQTT 事件处理"
- "MQTT_EVENT_DATA 怎么处理"
- "解析 MQTT 错误"
- "esp_mqtt_event_t 字段"

## 前置条件

| 条件 | 要求 |
|---|---|
| 参考示例 | `examples/tcp/main/app_main.c`、`examples/ssl_mutual_auth/main/app_main.c` |

## 分步说明

### 1. 事件 ID 枚举（`include/mqtt_client.h` 中 `esp_mqtt_event_id_t`）

| 事件 | 触发时机 | 关键字段 |
|---|---|---|
| `MQTT_EVENT_BEFORE_CONNECT` | 客户端已初始化、即将连接 | — |
| `MQTT_EVENT_CONNECTED` | 连接成功 | `session_present` |
| `MQTT_EVENT_DISCONNECTED` | 连接中断 / 失败 | — |
| `MQTT_EVENT_SUBSCRIBED` | broker 确认订阅 | `msg_id`、`data`（broker 响应） |
| `MQTT_EVENT_UNSUBSCRIBED` | broker 确认取消订阅 | `msg_id` |
| `MQTT_EVENT_PUBLISHED` | broker 确认发布（仅 QoS1/2） | `msg_id` |
| `MQTT_EVENT_DATA` | 收到 PUBLISH | `msg_id`、`topic`/`topic_len`、`data`/`data_len`、`current_data_offset`、`total_data_len`、`retain`、`qos`、`dup` |
| `MQTT_EVENT_ERROR` | 出错 | `error_handle` |
| `MQTT_EVENT_DELETED` | outbox 消息过期删除（需 `CONFIG_MQTT_REPORT_DELETED_MESSAGES`） | `msg_id` |

### 2. 完整回调骨架（来自仓库示例）

```c
static void log_error_if_nonzero(const char *message, int error_code)
{
    if (error_code != 0) {
        ESP_LOGE(TAG, "Last error %s: 0x%x", message, error_code);
    }
}

static void mqtt_event_handler(void *handler_args, esp_event_base_t base,
                               int32_t event_id, void *event_data)
{
    esp_mqtt_event_handle_t event = event_data;
    esp_mqtt_client_handle_t client = event->client;
    int msg_id;
    switch ((esp_mqtt_event_id_t)event_id) {
    case MQTT_EVENT_CONNECTED:
        ESP_LOGI(TAG, "MQTT_EVENT_CONNECTED");
        msg_id = esp_mqtt_client_subscribe(client, "topic/qos0", 0);
        break;
    case MQTT_EVENT_DISCONNECTED:
        ESP_LOGI(TAG, "MQTT_EVENT_DISCONNECTED");
        break;
    case MQTT_EVENT_SUBSCRIBED:
        ESP_LOGI(TAG, "MQTT_EVENT_SUBSCRIBED, msg_id=%d", event->msg_id);
        break;
    case MQTT_EVENT_UNSUBSCRIBED:
        ESP_LOGI(TAG, "MQTT_EVENT_UNSUBSCRIBED, msg_id=%d", event->msg_id);
        break;
    case MQTT_EVENT_PUBLISHED:
        ESP_LOGI(TAG, "MQTT_EVENT_PUBLISHED, msg_id=%d", event->msg_id);
        break;
    case MQTT_EVENT_DATA:
        printf("TOPIC=%.*s\r\n", event->topic_len, event->topic);
        printf("DATA=%.*s\r\n", event->data_len, event->data);
        break;
    case MQTT_EVENT_ERROR:
        ESP_LOGI(TAG, "MQTT_EVENT_ERROR");
        if (event->error_handle->error_type == MQTT_ERROR_TYPE_TCP_TRANSPORT) {
            log_error_if_nonzero("reported from esp-tls", event->error_handle->esp_tls_last_esp_err);
            log_error_if_nonzero("reported from tls stack", event->error_handle->esp_tls_stack_err);
            log_error_if_nonzero("captured as transport's socket errno",
                                 event->error_handle->esp_transport_sock_errno);
            ESP_LOGI(TAG, "Last errno string (%s)", strerror(event->error_handle->esp_transport_sock_errno));
        }
        break;
    default:
        ESP_LOGI(TAG, "Other event id:%d", event->event_id);
        break;
    }
}
```

### 3. 错误类型（`esp_mqtt_error_type_t`）

| error_type | 含义 | 相关字段 |
|---|---|---|
| `MQTT_ERROR_TYPE_NONE` | 无错误 | — |
| `MQTT_ERROR_TYPE_TCP_TRANSPORT` | 传输层 / esp-tls 错误 | `esp_tls_last_esp_err`、`esp_tls_stack_err`、`esp_tls_cert_verify_flags`、`esp_transport_sock_errno` |
| `MQTT_ERROR_TYPE_CONNECTION_REFUSED` | broker 拒绝连接 | `connect_return_code` |
| `MQTT_ERROR_TYPE_SUBSCRIBE_FAILED` | 订阅失败 | — |

`connect_return_code`（`esp_mqtt_connect_return_code_t`）：`MQTT_CONNECTION_ACCEPTED(0)`、`..._REFUSE_PROTOCOL`、`..._ID_REJECTED`、`..._SERVER_UNAVAILABLE`、`..._BAD_USERNAME`、`..._NOT_AUTHORIZED`。

### 4. 注册所有事件

```c
esp_mqtt_client_register_event(client, ESP_EVENT_ANY_ID, mqtt_event_handler, NULL);
```

## 常见错误

| 错误 | 原因 | 解决方法 |
|---|---|---|
| topic/data 打印乱码 | 用 `%s` 打印 | 改 `%.*s` 配 `topic_len`/`data_len` |
| 订阅后无 DATA | QoS 不匹配或 topic 写错 | 核对 topic 与发布端 QoS |
| 错误信息读不出 | 未判断 error_type 直接读字段 | 先 switch `error_type` 再读对应字段 |

## 参考

- `examples/tcp/main/app_main.c`
- `examples/ssl_mutual_auth/main/app_main.c`
- `docs/en/index.rst`（Events / 错误与断开关系表）
