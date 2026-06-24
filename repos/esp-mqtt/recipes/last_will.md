# 遗嘱消息（LWT）

> **适用摘要**: 配置 Last Will and Testament，使客户端异常断开时由 broker 代发通知消息。MQTT 3.1.1 与 5.0 均支持。

## 触发意图

- "MQTT 遗嘱消息"
- "last will"
- "LWT 配置"
- "掉线通知"

## 前置条件

| 条件 | 要求 |
|---|---|
| 参考示例 | `examples/mqtt5/main/app_main.c`（含 LWT） |

## 分步说明

### 1. MQTT 3.1.1 / 5.0 通用 LWT（`session.last_will_t`）

```c
esp_mqtt_client_config_t mqtt_cfg = {
    .broker.address.uri = "mqtt://broker.example.com",
    .session.last_will = {
        .topic   = "topic/will",
        .msg     = "i will leave",
        .msg_len = 12,        // msg 非 NULL 结尾时必须给长度
        .qos     = 1,
        .retain  = true,
    },
};
```

字段（`include/mqtt_client.h` 中 `last_will_t`）：

| 字段 | 说明 |
|---|---|
| `topic` | LWT 主题 |
| `msg` | LWT 负载，可 NULL 结尾 |
| `msg_len` | 负载长度；msg 非 NULL 结尾时必填 |
| `qos` | LWT QoS |
| `retain` | 是否 retained |

### 2. MQTT 5 的 will 属性

MQTT5 中可在 `esp_mqtt5_connection_property_config_t` 设置：

| 字段 | 说明 |
|---|---|
| `will_delay_interval` | broker 延迟发布 will 的时间 |
| `message_expiry_interval` | will 消息过期 |
| `payload_format_indicator` | 负载格式 |
| `content_type` | MIME 内容类型 |
| `response_topic` | 请求/响应主题 |
| `correlation_data` / `correlation_data_len` | 关联数据 |
| `will_user_property` | will 的用户属性 |

```c
esp_mqtt5_connection_property_config_t connect_property = {
    .will_delay_interval = 10,
    .message_expiry_interval = 10,
    .payload_format_indicator = true,
    .response_topic = "test/response",
    // ...
};
// will_user_property 同 user_property 用法
esp_mqtt5_client_set_user_property(&connect_property.will_user_property, arr, N);
esp_mqtt5_client_set_connect_property(client, &connect_property);
esp_mqtt5_client_delete_user_property(connect_property.will_user_property);
```

### 3. 触发时机

- 客户端未发 DISCONNECT 而断开（网络中断、keepalive 超时）→ broker 发布 LWT。
- MQTT5 中 `esp_mqtt_client_disconnect` 若配置 `disconnect_reason = MQTT5_DISCONNECT_WITH_WILL(0x04)` 也会触发 will。

## 常见错误

| 错误 | 原因 | 解决方法 |
|---|---|---|
| will 不发布 | 客户端正常 DISCONNECT | LWT 仅在异常断开时发布 |
| msg 截断 | 非 NULL 结尾但未给 msg_len | 正确设置 `msg_len` |
| MQTT5 will 属性无效 | 未设 `protocol_ver=MQTT_PROTOCOL_V_5` | 启用 MQTT5 |

## 参考

- `examples/mqtt5/main/app_main.c`
- `include/mqtt_client.h`（`last_will_t`）
- `include/mqtt5_client.h`（`esp_mqtt5_connection_property_config_t`）
- `docs/en/index.rst`（Last Will and Testament 节）
