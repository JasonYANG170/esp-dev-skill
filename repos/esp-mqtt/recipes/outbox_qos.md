# QoS 1/2 与 Outbox 管理

> **适用摘要**: 理解 ESP-MQTT 的内存 outbox 机制，处理不稳定网络下的消息堆积、限流、重传与过期，避免 `-2` 与丢消息。

> Version: selected repo/component version; align with the user project and dependency manifest.
> Evidence: `repos/esp-mqtt/resources/`, source/examples in `repos/esp-mqtt/`, and this recipe path `repos/esp-mqtt/recipes/outbox_qos.md`.
> Validation: draft metadata added from repository routing; verify APIs, Kconfig symbols, and component dependencies against the selected version.

## 触发意图

- "outbox 太大"
- "publish 返回 -2"
- "QoS1 重传"
- "不稳定网络 MQTT"

## 前置条件

| 条件 | 要求 |
|---|---|
| 参考示例 | `examples/custom_outbox/main/app_main.c`（enqueue） |
| 文档 | `docs/en/index.rst`（Outbox / Considerations 节） |

## 分步说明

### 1. outbox 工作原理（来自 `docs/en/index.rst`）

- QoS 1/2 的 PUBLISH（及 QoS2 握手）进入内存 outbox 等待 ACK。
- 连接时 FIFO 发送，重连后以 DUP 重传，收到 PUBACK/PUBREL/PUBCOMP 后移除。
- 重传间隔 = `session.message_retransmit_timeout`（默认 1000ms）。
- 超过 `CONFIG_MQTT_OUTBOX_EXPIRED_TIMEOUT_MS`（默认 30000ms）消息过期删除；若 `CONFIG_MQTT_REPORT_DELETED_MESSAGES=y` 则发 `MQTT_EVENT_DELETED`。

### 2. 限制 outbox 容量（`outbox_config_t.limit`）

```c
esp_mqtt_client_config_t mqtt_cfg = {
    .broker.address.uri = "mqtt://broker.example.com",
    .outbox.limit = 4096,   // 字节预算
};
```

> outbox 能容纳消息数 ≈ `floor(limit / (topic_len + payload_len + ~4..6 开销))`。

### 3. 处理 publish 返回值

```c
int msg_id = esp_mqtt_client_publish(client, topic, data, len, qos, retain);
if (msg_id == -1) {
    // 发送失败
} else if (msg_id == -2) {
    // outbox 已满，应降速 / 丢弃 / 等待
} else {
    // 成功（QoS0 恒为 0）
}
```

### 4. 主动监控 outbox 大小（来自 `docs/en/index.rst`）

```c
int outbox_size = esp_mqtt_client_get_outbox_size(client);
if (outbox_size > OUTBOX_MAX_ALLOWED) {
    if (message_is_important) {
        esp_mqtt_client_publish(client, topic, data, len, qos, retain);
    } else {
        // 丢弃或降低采样率
    }
} else {
    esp_mqtt_client_publish(client, topic, data, len, qos, retain);
}
```

### 5. 三种缓解策略（来自文档）

1. 改用 QoS 0（不入 outbox，但有丢失风险）。
2. 设 `outbox.limit` 并处理 `-2`。
3. 用 `esp_mqtt_client_get_outbox_size()` 手动决策。

### 6. publish vs enqueue

| API | 发送上下文 | QoS0 离线行为 |
|---|---|---|
| `esp_mqtt_client_publish` | 调用任务，立即发送 | 失败返回 -1 |
| `esp_mqtt_client_enqueue(..., store=true)` | MQTT 任务，先入 outbox | 可缓存（store=true） |

## 常见错误

| 错误 | 原因 | 解决方法 |
|---|---|---|
| 持续返回 -2 | outbox 满 | 增大 `outbox.limit` 或降速 / 降 QoS |
| 重连后重发旧消息 | QoS1/2 默认重传 | 属正常；设 `disable_clean_session` 控制会话行为 |
| 内存持续增长 | 未限流 | 用 `get_outbox_size()` 监控 + `outbox.limit` |
| 想关闭重传 | — | 库默认始终重传，可改用 QoS0 |

## 参考

- `examples/custom_outbox/main/app_main.c`（enqueue 用法）
- `docs/en/index.rst`（Outbox (QoS persistence) / Considerations 节）
- `include/mqtt_client.h`（`esp_mqtt_client_get_outbox_size`、`outbox_config_t`）
