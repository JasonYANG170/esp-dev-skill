# MQTT 5.0 协议

> **适用摘要**: 使用 MQTT v5.0：设置协议版本、连接属性、用户属性、共享订阅、读取 reason code 与 CONNACK 服务端属性，对应 `examples/mqtt5/`。

> Validation: draft metadata added from repository routing; verify APIs, Kconfig symbols, and component dependencies against the selected version.

## 触发意图

- "MQTT 5"
- "MQTT v5 用户属性"
- "共享订阅 MQTT5"
- "MQTT5 reason code"

## 前置条件

| 条件 | 要求 |
|---|---|
| Kconfig | `CONFIG_MQTT_PROTOCOL_5=y`（默认 n） |
| broker | 需支持 MQTT 5.0 |
| 参考示例 | `examples/mqtt5/` |

## 分步说明

### 1. 启用协议版本与基础配置（来自 `examples/mqtt5/main/app_main.c`）

```c
esp_mqtt_client_config_t mqtt5_cfg = {
    .broker.address.uri = CONFIG_BROKER_URL,
    .session.protocol_ver = MQTT_PROTOCOL_V_5,
    .network.disable_auto_reconnect = true,
    .session.last_will.topic = "topic/will",
    .session.last_will.msg = "i will leave",
    .session.last_will.msg_len = 12,
    .session.last_will.qos = 1,
    .session.last_will.retain = true,
};
```

`esp_mqtt_protocol_ver_t` 取值：`MQTT_PROTOCOL_UNDEFINED`、`MQTT_PROTOCOL_V_3_1`、`MQTT_PROTOCOL_V_3_1_1`、`MQTT_PROTOCOL_V_5`。

### 2. 设置连接属性（`esp_mqtt5_connection_property_config_t`）

```c
esp_mqtt5_connection_property_config_t connect_property = {
    .session_expiry_interval = 10,
    .maximum_packet_size = 1024,
    .receive_maximum = 65535,
    .topic_alias_maximum = 2,
    .request_resp_info = true,
    .request_problem_info = true,
    .will_delay_interval = 10,
    .payload_format_indicator = true,
    .message_expiry_interval = 10,
    .response_topic = "test/response",
    .correlation_data = "123456",
    .correlation_data_len = 6,
};

esp_mqtt5_user_property_item_t user_property_arr[] = {
    {"board", "esp32"}, {"u", "user"}, {"p", "password"}
};
esp_mqtt5_client_set_user_property(&connect_property.user_property, user_property_arr,
                                   sizeof(user_property_arr)/sizeof(esp_mqtt5_user_property_item_t));
esp_mqtt5_client_set_connect_property(client, &connect_property);
// set 之后必须 delete（内部已 malloc 副本）
esp_mqtt5_client_delete_user_property(connect_property.user_property);
```

### 3. 发布属性（一次性）

```c
static esp_mqtt5_publish_property_config_t publish_property = {
    .payload_format_indicator = 1,
    .message_expiry_interval = 1000,
    .topic_alias = 0,
    .response_topic = "topic/test/response",
    .correlation_data = "123456",
    .correlation_data_len = 6,
};
// 每次 publish 前设置（不存储）
esp_mqtt5_client_set_user_property(&publish_property.user_property, user_property_arr, N);
esp_mqtt5_client_set_publish_property(client, &publish_property);
esp_mqtt_client_publish(client, "topic/qos1", "data_3", 0, 1, 1);
esp_mqtt5_client_delete_user_property(publish_property.user_property);
publish_property.user_property = NULL;
```

### 4. 订阅属性与共享订阅

```c
static esp_mqtt5_subscribe_property_config_t subscribe_property = {
    .subscribe_id = 25555,
    .is_share_subscribe = true,
    .share_name = "group1",   // 对应 $share/group1/<topic>
};
esp_mqtt5_client_set_user_property(&subscribe_property.user_property, user_property_arr, N);
esp_mqtt5_client_set_subscribe_property(client, &subscribe_property);
esp_mqtt_client_subscribe(client, "topic/qos0", 0);
esp_mqtt5_client_delete_user_property(subscribe_property.user_property);
```

取消订阅同理，用 `esp_mqtt5_client_set_unsubscribe_property`；断开用 `esp_mqtt5_client_set_disconnect_property`。

### 5. 事件中读取属性与 reason code

```c
case MQTT_EVENT_DATA:
    if (event->property && event->property->user_property) {
        print_user_property(event->property->user_property);
    }
    ESP_LOGI(TAG, "TOPIC=%.*s", event->topic_len, event->topic);
    break;
case MQTT_EVENT_ERROR:
    ESP_LOGI(TAG, "MQTT5 return code is %d", event->error_handle->connect_return_code);
    break;
```

`event->property` 为 `esp_mqtt5_event_property_t*`，包含 `payload_format_indicator`、`response_topic`、`correlation_data`、`content_type`、`subscribe_id`、`user_property`，以及 `server`（`esp_mqtt5_server_resp_property_t`，CONNACK 返回，仅在 CONNECTED 有效：`maximum_packet_size`、`receive_maximum`、`topic_alias_maximum`、`max_qos`、`retain_available`、`wildcard_subscribe_available`、`shared_subscribe_available`、`response_info` 等）。

### 6. user_property 工具函数

| 函数 | 作用 |
|---|---|
| `esp_mqtt5_client_set_user_property(handle, item[], n)` | 分配并填充（内部 malloc） |
| `esp_mqtt5_client_get_user_property(handle, *item, *n)` | 读取（调用方需释放 key/value） |
| `esp_mqtt5_client_get_user_property_count(handle)` | 返回条数 |
| `esp_mqtt5_client_delete_user_property(handle)` | 释放列表与自身 |

打印示例（来自仓库示例 `print_user_property`）：分配 count 个 `esp_mqtt5_user_property_item_t`，`get` 后逐个 `free((char*)t->key)` 与 `free((char*)t->value)`，最后 `free(item)`。

## 常见错误

| 错误 | 原因 | 解决方法 |
|---|---|---|
| 编译找不到 `esp_mqtt5_*` | 未开 Kconfig | `CONFIG_MQTT_PROTOCOL_5=y` |
| 内存泄露 | set 后未 delete | 每次 set 后配对 `esp_mqtt5_client_delete_user_property` |
| 共享订阅无效 | broker 不支持 | 查 CONNACK `server.shared_subscribe_available` |
| reason code 全 0 | broker 用默认 | 用 `event->reason_code` / `error_handle->connect_return_code` |

## 参考

- `examples/mqtt5/main/app_main.c`
- `examples/mqtt5/README.md`
- `include/mqtt5_client.h`（全部 `esp_mqtt5_*` API 与结构体）
- `docs/en/index.rst`（Configuration 节）
