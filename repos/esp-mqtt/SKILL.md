---
name: esp-mqtt-skill
description: >-
  AI Skill for the Espressif ESP-MQTT client component. Use when developing ESP-IDF
  firmware that publishes/subscribes over MQTT (TCP, TLS/SSL, WebSocket, WebSocket Secure)
  and MQTT 5.0, including event handling, last-will, QoS/outbox, mutual-auth TLS, PSK,
  digital-signature, and custom outbox scenarios.
  Trigger words: "esp-mqtt", "MQTT", "MQTT 5", "mqtt client", "ESP32", "ESP-IDF",
  "subscribe", "publish", "broker", "last will", "QoS", "TLS", "WebSocket", "消息队列", "订阅", "发布", "遗嘱消息"
license: Apache-2.0
metadata:
  author: Community
  version: "1.1.0"
---

# esp-mqtt-skill

面向 Espressif **ESP-MQTT** 客户端组件的 AI Skill。ESP-MQTT 是 ESP-IDF 中实现 MQTT 协议（v3.1.1 与 v5.0）的客户端库，支持 MQTT over TCP、MQTT over SSL（mbedTLS）、MQTT over WebSocket 与 WebSocket Secure，可创建多个客户端实例，支持订阅/发布、认证、遗嘱消息（LWT）、keepalive 心跳以及全部三种 QoS 等级。本 Skill 提供场景驱动的 recipes、真实 API 参考、Kconfig 配置速查与高频陷阱，所有 API、结构体、宏、配置项与代码片段均来自仓库源码与 `docs/`。

## Core Principles

1. **绝不臆造 API** — 所有函数、结构体、宏必须能在 `resources/api_reference.md` 或仓库头文件中查到；查不到即视为不存在。
2. **ESP-MQTT 是 ESP-IDF 组件** — 通过 `idf_component.yml` 声明依赖 `espressif/mqtt`，或将仓库克隆为 `mqtt` 目录；`PRIV_REQUIRES mqtt`。
3. **事件驱动模型** — 客户端基于默认事件循环派发 `MQTT_EVENT_*`；必须用 `esp_mqtt_client_register_event(client, ESP_EVENT_ANY_ID, handler, arg)` 注册处理函数。
4. **初始化三步走** — `esp_mqtt_client_init(&cfg)` → `esp_mqtt_client_register_event(...)` → `esp_mqtt_client_start(client)`，顺序固定。
5. **配置走 `esp_mqtt_client_config_t` 嵌套结构体** — 通过 `.broker.address.uri`、`.credentials.*`、`.session.*`、`.network.*`、`.buffer.*`、`.outbox.limit` 等字段进行指定式初始化。
6. **URI 决定传输层** — scheme 取值 `mqtt`/`mqtts`/`ws`/`wss`，分别对应 TCP/SSL/WebSocket/WebSocket Secure；URI 优先级高于 hostname+port。
7. **QoS>0 消息进 outbox** — QoS 1/2 的 PUBLISH 会进入内存 outbox 等待 ACK；可用 `esp_mqtt_client_get_outbox_size()` 监控、用 `outbox.limit` 限流；返回 `-2` 表示 outbox 已满。
8. **TLS 校验务必配置** — 安全连接需设置 `broker.verification.certificate`（PEM/DER）或 `use_global_ca_store` / `crt_bundle_attach`；`skip_cert_common_name_check=true` 会降低安全性。
9. **大消息会分片** — 报文超过 `buffer.size` 时会触发多次 `MQTT_EVENT_DATA`，仅在首个事件携带 topic；需用 `current_data_offset` 与 `total_data_len` 重组。
10. **不要在事件回调里调用 `stop`/`destroy`** — `esp_mqtt_client_stop()` 与 `esp_mqtt_client_destroy()` 明确禁止在事件处理函数中调用。
11. **MQTT 5 需开启 Kconfig** — `CONFIG_MQTT_PROTOCOL_5=y` 后才能使用 `esp_mqtt5_*` API 与 `session.protocol_ver = MQTT_PROTOCOL_V_5`。
12. **证书/密钥用 embed 方式** — 通过 `EMBED_TXTFILES` / `EMBED_BINFILES` 嵌入并以 `_binary_*_start` / `_end` 符号引用，避免内存管理问题。

## When to Use

**Applicable:**
- 在 ESP-IDF 项目中创建 ESP-MQTT 客户端并连接 broker
- 实现订阅、发布、QoS 0/1/2、retain、last will
- 配置 TLS：服务端证书校验、双向认证（client cert + key）、PSK、数字签名外设、ECDSA 外设
- 使用 MQTT over WebSocket / WebSocket Secure
- 使用 MQTT v5.0：连接属性、用户属性、发布/订阅属性、reason code、共享订阅
- 管理不稳定网络下的 outbox 限流与重连策略
- 自定义 outbox 实现（`CONFIG_MQTT_CUSTOM_OUTBOX`）

**Not applicable:**
- 与 MQTT 客户端无关的通用 C / FreeRTOS 编程
- 实现 MQTT broker（服务端）—— 本组件只是 client
- 非 ESP-IDF 工具链下的 MQTT 库（如 Paho、pubsubclient）
- 裸机 / RTOS 之外的传输层细节（请参考 esp-tls / tcp_transport 组件）

---

## Scenario Quick Reference (Recipes)

当用户意图匹配下列场景时，**优先阅读对应 recipe** —— 它包含完整调用链、分步说明、常见错误与可复制代码。

### 基础连接

| recipe | 场景 |
|---|---|
| `recipes/tcp_connect.md` | MQTT over TCP 最小连接：init/register/start 与事件处理 |
| `recipes/event_handling.md` | 完整事件回调骨架（CONNECTED/SUBSCRIBED/DATA/ERROR 等） |
| `recipes/publish_subscribe.md` | 订阅、发布、QoS 等级、retain、多主题订阅 |

### 传输与安全

| recipe | 场景 |
|---|---|
| `recipes/tls_mutual_auth.md` | mqtts:// 双向 TLS 认��（client cert + key + server CA） |
| `recipes/tls_server_cert.md` | mqtts:// 单向校验（仅服务端 CA 证书，PEM 嵌入） |
| `recipes/tls_digital_signature.md` | mqtts:// 数字签名（DS）外设认证：私钥不出硬件，经 `esp_secure_cert` 分区 + `ds_data` 字段（ESP32-S2/S3/C3/C5/C6/H2/P4） |
| `recipes/psk_auth.md` | mqtts:// 使用 PSK 预共享密钥认证 |
| `recipes/ws_wss.md` | WebSocket（ws://）与 WebSocket Secure（wss://）连接 |

### MQTT 5 与高级特性

| recipe | 场景 |
|---|---|
| `recipes/mqtt5.md` | MQTT v5.0：协议版本、连接属性、用户属性、共享订阅、reason code |
| `recipes/last_will.md` | 遗嘱消息（LWT）：topic/msg/qos/retain 配置 |
| `recipes/outbox_qos.md` | QoS 1/2 outbox 管理、限流、enqueue、不稳定网络处理 |
| `recipes/custom_outbox.md` | 自定义 outbox 实现（CONFIG_MQTT_CUSTOM_OUTBOX + CMake 追加源码） |

---

## 关键配置速查（Kconfig，位于 `Kconfig`，菜单 `Component config` > `ESP-MQTT Configurations`）

| 配置项 | 类型 | 默认 | 说明 |
|---|---|---|---|
| `CONFIG_MQTT_PROTOCOL_311` | bool | y | 启用 MQTT 3.1.1 |
| `CONFIG_MQTT_PROTOCOL_5` | bool | n | 启用 MQTT 5.0（使用 `esp_mqtt5_*` API 必开） |
| `CONFIG_MQTT_TRANSPORT_SSL` | bool | y | 启用 mqtts:// |
| `CONFIG_MQTT_TRANSPORT_WEBSOCKET` | bool | y | 启用 ws://（依赖 WS_TRANSPORT） |
| `CONFIG_MQTT_TRANSPORT_WEBSOCKET_SECURE` | bool | y | 启用 wss://（依赖上两项） |
| `CONFIG_MQTT_SKIP_PUBLISH_IF_DISCONNECTED` | bool | n | 断开时不再入队发布消息 |
| `CONFIG_MQTT_REPORT_DELETED_MESSAGES` | bool | n | outbox 过期删除消息时上报 `MQTT_EVENT_DELETED` |
| `CONFIG_MQTT_MSG_ID_INCREMENTAL` | bool | n | msg_id 递增而非随机 |
| `CONFIG_MQTT_BUFFER_SIZE` | int | 1024 | 收发缓冲大小（需 `CONFIG_MQTT_USE_CUSTOM_CONFIG`） |
| `CONFIG_MQTT_TASK_STACK_SIZE` | int | 6144 | MQTT 任务栈大小 |
| `CONFIG_MQTT_TASK_PRIORITY` | int | 5 | MQTT 任务优先级 |
| `CONFIG_MQTT_EVENT_QUEUE_SIZE` | int | 1 | 事件队列深度（>1 允许多事件排队） |
| `CONFIG_MQTT_POLL_READ_TIMEOUT_MS` | int | 1000 | 传输层 poll 读超时 |
| `CONFIG_MQTT_OUTBOX_EXPIRED_TIMEOUT_MS` | int | 30000 | outbox 消息过期时间 |
| `CONFIG_MQTT_CUSTOM_OUTBOX` | bool | n | 启用自定义 outbox 实现 |
| `CONFIG_MQTT_OUTBOX_DATA_ON_EXTERNAL_MEMORY` | bool | n | outbox 数据放外部内存 |
| `CONFIG_MQTT_TOPIC_PRESENT_ALL_DATA_EVENTS` | bool | n | 大消息分片时每个 DATA 事件都携带 topic |
| `CONFIG_MQTT_DISABLE_API_LOCKS` | bool | n | 关闭 API 互斥锁（单任务访问时可用） |
| `CONFIG_MQTT_TCP_DEFAULT_PORT` / `SSL_DEFAULT_PORT` / `WS_DEFAULT_PORT` / `WSS_DEFAULT_PORT` | int | 1883 / 8883 / 80 / 443 | 各传输默认端口 |

> 注：`MQTT_BUFFER_SIZE` / `MQTT_TASK_STACK_SIZE` / `*_DEFAULT_PORT` 等项依赖 `CONFIG_MQTT_USE_CUSTOM_CONFIG=y`；运行时通常直接通过 `esp_mqtt_client_config_t` 的 `buffer.size`、`task.stack_size`、`broker.address.port` 覆盖。

## 默认端口与 scheme

| scheme | 传输 | 默认端口 | Kconfig |
|---|---|---|---|
| `mqtt` | MQTT over TCP | 1883 | — |
| `mqtts` | MQTT over SSL/TLS | 8883 | `CONFIG_MQTT_TRANSPORT_SSL` |
| `ws` | MQTT over WebSocket | 80 | `CONFIG_MQTT_TRANSPORT_WEBSOCKET` |
| `wss` | MQTT over WebSocket Secure | 443 | `CONFIG_MQTT_TRANSPORT_WEBSOCKET_SECURE` |

对应宏（`include/mqtt_client.h`）：`MQTT_OVER_TCP_SCHEME`、`MQTT_OVER_SSL_SCHEME`、`MQTT_OVER_WS_SCHEME`、`MQTT_OVER_WSS_SCHEME`。

## 客户端状态机（`esp_mqtt_client_connection_state_t`）

```
MQTT_CLIENT_STATE_NOT_INITIALIZED
        │  esp_mqtt_client_init()
        ▼
MQTT_CLIENT_STATE_NOT_STARTED
        │  esp_mqtt_client_start()
        ▼
MQTT_CLIENT_STATE_DISCONNECTED  ─────┐  连接建立成功
        │                            │
        ▼                            │
MQTT_CLIENT_STATE_CONNECTED         │  连接断开/出错且未禁用自动重连
        │                            │
        └───────────────────────────►│
                                       ▼
                         MQTT_CLIENT_STATE_WAITING_RECONNECT
                                       │  reconnect_timeout_ms 后重试
                                       ▼
                             (回到 DISCONNECTED 尝试连接)
```

> 禁用自动重连（`network.disable_auto_reconnect=true`）后，断开将停留在 DISCONNECTED，需用 `esp_mqtt_client_reconnect()`（仅在等待重连时生效）或重新 `esp_mqtt_client_start()`。

---

## Critical Pitfalls (Must Read)

以下是最常见的错误，违反任一条都会导致固件无法正常工作。

### 1. 必须先注册事件再 start

```c
// ❌ WRONG — 先 start 后注册，会丢失 BEFORE_CONNECT / 早期事件
esp_mqtt_client_start(client);
esp_mqtt_client_register_event(client, ESP_EVENT_ANY_ID, handler, NULL);

// ✅ CORRECT — 先注册再启动
esp_mqtt_client_handle_t client = esp_mqtt_client_init(&cfg);
esp_mqtt_client_register_event(client, ESP_EVENT_ANY_ID, handler, NULL);
esp_mqtt_client_start(client);
```

### 2. 缺少默认事件循环与网络初始化

```c
// ❌ WRONG — 跳过 netif/event loop 初始化，MQTT 无法派发事件
void app_main(void) {
    esp_mqtt_client_config_t cfg = { .broker.address.uri = "mqtt://..." };
    esp_mqtt_client_start(esp_mqtt_client_init(&cfg));
}

// ✅ CORRECT — 先建事件循环与默认 netif，再联网
ESP_ERROR_CHECK(nvs_flash_init());
ESP_ERROR_CHECK(esp_netif_init());
ESP_ERROR_CHECK(esp_event_loop_create_default());
ESP_ERROR_CHECK(example_connect());   // Wi-Fi / Ethernet
mqtt_app_start();
```

### 3. TLS 安全连接未配置任何校验

```c
// ❌ WRONG — mqtts:// 但 verification 全空，握手常失败且不安全
.broker.address.uri = "mqtts://broker.example.com",

// ✅ CORRECT — 至少提供 CA 证书（PEM 嵌入）
.broker.address.uri = "mqtts://broker.example.com",
.broker.verification.certificate = (const char *)server_cert_pem_start,
// 或 .broker.verification.use_global_ca_store = true;
// 或 .broker.verification.crt_bundle_attach = esp_crt_bundle_attach,
```

### 4. 用 %s 直接打印 topic/data 而非 %.*s

```c
// ❌ WRONG — topic/data 不是以 \0 结尾，%s 会越界
printf("TOPIC=%s DATA=%s\n", event->topic, event->data);

// ✅ CORRECT — 用长度限定符
printf("TOPIC=%.*s\r\n", event->topic_len, event->topic);
printf("DATA=%.*s\r\n", event->data_len, event->data);
```

### 5. publish 返回值未区分 -1 / -2

```c
// ❌ WRONG — 只看是否 < 0，无法区分 outbox 满
int id = esp_mqtt_client_publish(client, "t", "d", 0, 1, 0);
if (id < 0) { /* 误判为一般错误 */ }

// ✅ CORRECT — -1 失败，-2 outbox 已满，>=0 成功（QoS0 恒为 0）
int id = esp_mqtt_client_publish(client, "t", "d", 0, 1, 0);
if (id == -1) { /* 发送失败 */ }
else if (id == -2) { /* outbox 已满，应降速或丢弃 */ }
```

### 6. QoS0 在断开时 publish 直接失败

```c
// ❌ WRONG — 断开状态下 QoS0 publish 返回 -1（不会入队）
if (!connected) {
    esp_mqtt_client_publish(client, "t", "d", 0, 0, 0);  // 丢失
}

// ✅ CORRECT — 离线缓存用 enqueue + store=true
int id = esp_mqtt_client_enqueue(client, "t", "d", 0, 0, 0, true);  // store=true 允许 QoS0 入队
// 或在 MQTT_EVENT_CONNECTED 回调里再 publish
```

### 7. 大消息分片时只在第一个事件读 topic

```c
// ❌ WRONG — 第二个 DATA 事件 topic 为 NULL，topic_len=0 仍访问
case MQTT_EVENT_DATA:
    process(event->topic, event->topic_len);  // 分片后半段崩溃

// ✅ CORRECT — 缓存首事件 topic，按 current_data_offset/total_data_len 重组
case MQTT_EVENT_DATA:
    if (event->current_data_offset == 0) {
        save_topic(event->topic, event->topic_len);
    }
    append_payload(event->data, event->data_len);
    if (event->current_data_offset + event->data_len == event->total_data_len) {
        deliver_message();  // 重组完成
    }
```

### 8. 在事件回调里调用 stop / destroy

```c
// ❌ WRONG — 文档明确禁止在事件处理函数中调用，会死锁
case MQTT_EVENT_CONNECTED:
    esp_mqtt_client_stop(client);   // 永久阻塞

// ✅ CORRECT — 通过标志位在独立任务里停止
case MQTT_EVENT_CONNECTED:
    xTaskNotifyGive(stop_task_handle);
// 在 stop_task（非 mqtt 任务）中：
esp_mqtt_client_stop(client);
```

### 9. MQTT5 API 在未开 CONFIG_MQTT_PROTOCOL_5 时不可用

```c
// ❌ WRONG — 未使能 CONFIG_MQTT_PROTOCOL_5，esp_mqtt5_* 与 event->property 编译失败
esp_mqtt5_client_set_publish_property(client, &prop);

// ✅ CORRECT — 先在 menuconfig / sdkconfig 启用，并设置协议版本
// CONFIG_MQTT_PROTOCOL_5=y
.session.protocol_ver = MQTT_PROTOCOL_V_5,
// 之后才能调用 esp_mqtt5_client_set_connect_property() 等
```

### 10. user_property 分配内存后忘记释放

```c
// ❌ WRONG — set_user_property 内部 malloc，泄露
esp_mqtt5_client_set_user_property(&publish_property.user_property, arr, N);
esp_mqtt5_client_set_publish_property(client, &publish_property);

// ✅ CORRECT — set 之后立即 delete
esp_mqtt5_client_set_user_property(&publish_property.user_property, arr, N);
esp_mqtt5_client_set_publish_property(client, &publish_property);
esp_mqtt5_client_delete_user_property(publish_property.user_property);
publish_property.user_property = NULL;
```

### 11. 在回调中处理 MQTT5 event->property 但未判空

```c
// ❌ WRONG — 非 MQTT5 构建或非数据事件时 property 可能为空
ESP_LOGI(TAG, "resp=%.*s", event->property->response_topic_len, event->property->response_topic);

// ✅ CORRECT — 仅在 CONFIG_MQTT_PROTOCOL_5 且对应事件中访问
#ifdef CONFIG_MQTT_PROTOCOL_5
if (event->property && event->property->user_property) {
    print_user_property(event->property->user_property);
}
#endif
```

### 12. keepalive 设为 0 不能关闭 keepalive

```c
// ❌ WRONG — keepalive=0 使用默认值，并非关闭
.session.keepalive = 0,

// ✅ CORRECT — 显式 disable_keepalive 才能关闭
.session.disable_keepalive = true,
// 注意：keepalive 默认 120s，客户端会以设定值的一半间隔主动通信
```

### 13. 多实例但 buffer/outbox 未单独配置导致内存吃紧

```c
// ❌ WRONG — 多个客户端都用默认 1024 缓冲 + 无限 outbox，堆内存耗尽
for (int i=0;i<5;i++) esp_mqtt_client_init(&default_cfg);

// ✅ CORRECT — 逐实例配置 buffer 与 outbox.limit
.buffer = { .size = 512, .out_size = 512 },
.outbox = { .limit = 4096 },  // 每实例 outbox 上限 4KB
```

---

## Execution Workflow

| Step | Name | Description |
|------|------|-------------|
| 1 | 需求 | 确认传输（tcp/mqtts/ws/wss）、是否 MQTT5、安全方式（CA/双向/PSK/DS）、QoS |
| 2 | Recipe | 在 `recipes/` 中匹配场景，先阅读对应 recipe |
| 3 | 依赖 | `idf_component.yml` 增加 `espressif/mqtt: "*"`，CMake `PRIV_REQUIRES mqtt` |
| 4 | 初始化 | `nvs_flash_init()` → `esp_netif_init()` → `esp_event_loop_create_default()` → 联网 |
| 5 | 客户端 | 填充 `esp_mqtt_client_config_t` → `init` → `register_event(ESP_EVENT_ANY_ID,...)` → `start` |
| 6 | 校验 | Kconfig（`CONFIG_MQTT_PROTOCOL_5`、`CONFIG_MQTT_CUSTOM_OUTBOX` 等）、证书嵌入、buffer/outbox |
| 7 | 构建 | `idf.py set-target <chip>` → `idf.py menuconfig` → `idf.py build` |
| 8 | 烧录 | `idf.py -p PORT flash monitor` |
| 9 | 调试 | 关注 `MQTT_EVENT_ERROR` 的 `error_handle->error_type` 与 `connect_return_code` |

### Step 5 Detail — 选择起点示例

根据需求选择仓库中最近的示例作为起点（见 `resources/example_list.md`）：

- 纯 TCP → `examples/tcp/`
- 服务端 CA 单向校验 → `examples/ssl/`
- 双向认证（client cert+key） → `examples/ssl_mutual_auth/`
- PSK 认证 → `examples/ssl_psk/`
- 数字签名外设（DS） → `examples/ssl_ds/`
- WebSocket → `examples/ws/` 或 `examples/wss/`
- MQTT 5 → `examples/mqtt5/`
- 自定义 outbox → `examples/custom_outbox/`

---

## Failure Strategies

| 情况 | 处理 |
|---|---|
| API 在 resources 中查不到 | 立即停止，告知用户该 API 不存在 |
| 握手失败 / 证书校验错误 | 检查 `error_handle->esp_tls_last_esp_err`、`esp_tls_cert_verify_flags`；确认 CA 与 hostname（或 `common_name`） |
| `MQTT_CONNECTION_REFUSE_*` | 读 `error_handle->connect_return_code`，对应协议/ID/认证/授权拒绝 |
| outbox 持续增长（返回 -2） | 设 `outbox.limit`，监控 `esp_mqtt_client_get_outbox_size()`，必要时降为 QoS0 |
| 连接不稳定反复重连 | 调 `network.reconnect_timeout_ms`；必要时 `disable_auto_reconnect=true` 自管 |
| 大消息丢失/截断 | 增大 `buffer.size`；处理分片 `current_data_offset`/`total_data_len` |
| MQTT5 编译报错 | 确认 `CONFIG_MQTT_PROTOCOL_5=y` 且 `session.protocol_ver=MQTT_PROTOCOL_V_5` |
| 回调里调用 stop/destroy 死锁 | 改为通知独立任务再调用 |

## References

- 场景 recipes → `recipes/` 目录
- API 速查 → `resources/api_reference.md`
- 配置速查 → `resources/config_reference.md`
- 陷阱汇总 → `resources/pitfalls.md`
- 示例清单 → `resources/example_list.md`
- 仓库源码 → https://github.com/espressif/esp-mqtt
- 官方文档 → https://docs.espressif.com/projects/esp-mqtt/en/latest/esp32/
