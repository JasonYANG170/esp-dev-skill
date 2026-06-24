# 发布、订阅与 QoS

> **适用摘要**: 使用 `esp_mqtt_client_publish` / `esp_mqtt_client_subscribe` / `unsubscribe`，理解三种 QoS、retain、多主题订阅与 publish/enqueue 返回值。

## 触发意图

- "MQTT 发布订阅"
- "QoS 0/1/2 区别"
- "订阅多个 topic"
- "MQTT retain 消息"

## 前置条件

| 条件 | 要求 |
|---|---|
| 客户端状态 | 已 `MQTT_EVENT_CONNECTED`（订阅/发布前） |
| 参考示例 | `examples/tcp/main/app_main.c` |

## 分步说明

### 1. 订阅单个主题（`esp_mqtt_client_subscribe_single`）

`include/mqtt_client.h` 提供 `_Generic` 宏 `esp_mqtt_client_subscribe`，按第二参数类型自动分派到 `subscribe_single`（`char*`）或 `subscribe_multiple`（`esp_mqtt_topic_t*`）。

```c
// 在 MQTT_EVENT_CONNECTED 中
int msg_id = esp_mqtt_client_subscribe(client, "topic/qos0", 0);  // QoS 0
msg_id = esp_mqtt_client_subscribe(client, "topic/qos1", 1);      // QoS 1
// 返回: msg_id 成功, -1 失败, -2 outbox 满
```

### 2. 订阅多个主题（`esp_mqtt_client_subscribe_multiple`）

```c
esp_mqtt_topic_t topic_list[] = {
    { .filter = "sensor/temp",  .qos = 1 },
    { .filter = "sensor/humid", .qos = 1 },
    { .filter = "cmd/+",        .qos = 2 },
};
int msg_id = esp_mqtt_client_subscribe_multiple(client, topic_list,
                                               sizeof(topic_list)/sizeof(topic_list[0]));
```

> C++ 中 `esp_mqtt_client_subscribe` 宏被重定义为 `subscribe_single`，多主题需直接调 `subscribe_multiple`。

### 3. 取消订阅

```c
int msg_id = esp_mqtt_client_unsubscribe(client, "topic/qos1");
```

### 4. 发布（`esp_mqtt_client_publish`）

```c
// QoS0，msg_id 恒为 0
int id = esp_mqtt_client_publish(client, "topic/qos0", "data", 0, 0, 0);
// 参数: client, topic, data(可 NULL=空负载), len(0=按字符串算长度), qos, retain
// QoS1
int id = esp_mqtt_client_publish(client, "topic/qos1", "data_3", 0, 1, 0);
```

返回值：`>=0` 成功（QoS0 恒 0），`-1` 失败，`-2` outbox 已满。

### 5. 入队发布（`esp_mqtt_client_enqueue`，非阻塞）

```c
// store=true 时连 QoS0 也入队，断开时缓存，重连后发送
int id = esp_mqtt_client_enqueue(client, "topic/qos1", "data", 0, 1, 0, true);
```

`enqueue` 把消息写入 outbox，实际发送在 MQTT 任务上下文；`publish` 在调用任务上下文立即发送。

### 6. QoS 行为对照

| QoS | publish 行为 | 是否入 outbox | 是否有 PUBLISHED 事件 |
|---|---|---|---|
| 0 | 发送一次，无 ACK | 否（断开即丢） | 否 |
| 1 | 至少一次，需 PUBACK | 是 | 是 |
| 2 | 恰好一次，需 PUBREC/PUBREL/PUBCOMP | 是 | 是 |

> ESP-MQTT 默认对未确认的 QoS1/2 PUBLISH 重传（DUP），重传间隔由 `session.message_retransmit_timeout`（默认 1000ms）控制；超过 `CONFIG_MQTT_OUTBOX_EXPIRED_TIMEOUT_MS`（默认 30000ms）消息过期删除。

### 7. retain 标志

```c
// 发布 retained 消息
esp_mqtt_client_publish(client, "status/online", "1", 0, 1, 1);  // retain=1
```

`MQTT_EVENT_DATA` 中 `event->retain` 指示该消息是否为 retained。

## 常见错误

| 错误 | 原因 | 解决方法 |
|---|---|---|
| QoS0 离线发布丢失 | QoS0 不入 outbox | 用 `enqueue(..., store=true)` 或 CONNECTED 后再发 |
| subscribe 返回 -2 | outbox 满 | 限流或调大 `outbox.limit` |
| 多主题订阅编译错（C++） | C++ 下宏退化为 single | 直接调用 `esp_mqtt_client_subscribe_multiple` |
| 反复收到重复消息 | QoS1 未确认触发重传 | 检查 broker 连接稳定性，必要时降 QoS |

## 参考

- `examples/tcp/main/app_main.c`
- `examples/custom_outbox/main/app_main.c`（enqueue 示例）
- `include/mqtt_client.h`（subscribe / publish / enqueue 函数文档）
- `docs/en/index.rst`（MQTT Message Retransmission 节）
