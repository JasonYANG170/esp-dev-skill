# ESP-MQTT 配置参考

> Kconfig 项来自仓库 `Kconfig`（菜单 `Component config` > `ESP-MQTT Configurations`）；运行时配置来自 `esp_mqtt_client_config_t`（`include/mqtt_client.h`）。

## Kconfig 项

| 配置项 | 类型 | 默认 | 依赖 | 说明 |
|---|---|---|---|---|
| `CONFIG_MQTT_PROTOCOL_311` | bool | y | — | 启用 MQTT 3.1.1 |
| `CONFIG_MQTT_PROTOCOL_5` | bool | n | — | 启用 MQTT 5.0 |
| `CONFIG_MQTT_TRANSPORT_SSL` | bool | y | — | mqtts:// |
| `CONFIG_MQTT_TRANSPORT_WEBSOCKET` | bool | y | `WS_TRANSPORT` | ws:// |
| `CONFIG_MQTT_TRANSPORT_WEBSOCKET_SECURE` | bool | y | WS+SSL | wss:// |
| `CONFIG_MQTT_MSG_ID_INCREMENTAL` | bool | n | — | msg_id 递增（否则随机） |
| `CONFIG_MQTT_SKIP_PUBLISH_IF_DISCONNECTED` | bool | n | — | 断开时不入队发布 |
| `CONFIG_MQTT_REPORT_DELETED_MESSAGES` | bool | n | — | outbox 过期删除时发 `MQTT_EVENT_DELETED` |
| `CONFIG_MQTT_USE_CUSTOM_CONFIG` | bool | n | — | 开启下列 *_DEFAULT / BUFFER / TASK 等自定义项 |
| `CONFIG_MQTT_TCP_DEFAULT_PORT` | int | 1883 | USE_CUSTOM_CONFIG | TCP 默认端口 |
| `CONFIG_MQTT_SSL_DEFAULT_PORT` | int | 8883 | +SSL | SSL 默认端口 |
| `CONFIG_MQTT_WS_DEFAULT_PORT` | int | 80 | +WS | WS 默认端口 |
| `CONFIG_MQTT_WSS_DEFAULT_PORT` | int | 443 | +WSS | WSS 默认端口 |
| `CONFIG_MQTT_BUFFER_SIZE` | int | 1024 | USE_CUSTOM_CONFIG | 收发缓冲 |
| `CONFIG_MQTT_TASK_STACK_SIZE` | int | 6144 | USE_CUSTOM_CONFIG | 任务栈 |
| `CONFIG_MQTT_DISABLE_API_LOCKS` | bool | n | USE_CUSTOM_CONFIG | 关闭 API 互斥锁 |
| `CONFIG_MQTT_TASK_PRIORITY` | int | 5 | USE_CUSTOM_CONFIG | 任务优先级 |
| `CONFIG_MQTT_POLL_READ_TIMEOUT_MS` | int | 1000 | USE_CUSTOM_CONFIG | poll 读超时 |
| `CONFIG_MQTT_EVENT_QUEUE_SIZE` | int | 1 | USE_CUSTOM_CONFIG | 事件队列深度 |
| `CONFIG_MQTT_TASK_CORE_SELECTION_ENABLED` | bool | n | — | 核心选择 |
| `CONFIG_MQTT_USE_CORE_0` / `_CORE_1` | choice | — | CORE_SELECTION | 绑定核 |
| `CONFIG_MQTT_OUTBOX_DATA_ON_EXTERNAL_MEMORY` | bool | n | USE_CUSTOM_CONFIG | outbox 放外部内存 |
| `CONFIG_MQTT_CUSTOM_OUTBOX` | bool | n | — | 自定义 outbox 实现 |
| `CONFIG_MQTT_OUTBOX_EXPIRED_TIMEOUT_MS` | int | 30000 | USE_CUSTOM_CONFIG | outbox 过期时间 |
| `CONFIG_MQTT_TOPIC_PRESENT_ALL_DATA_EVENTS` | bool | n | USE_CUSTOM_CONFIG | 大消息每片都带 topic |

> 多数默认值无需 `CONFIG_MQTT_USE_CUSTOM_CONFIG` 也能通过 `esp_mqtt_client_config_t` 在运行时覆盖（如 `buffer.size`、`task.stack_size`、`broker.address.port`）。

## 运行时配置（`esp_mqtt_client_config_t` 主要字段，全表见 `api_reference.md`）

### broker.address（地址）
- `uri`：`scheme://hostname:port/path`，scheme ∈ {mqtt, mqtts, ws, wss}。URI 优先于 hostname/transport/port。
- `hostname` / `transport` / `port` / `path`：分字段配置。

### broker.verification（校验）
- `use_global_ca_store` / `crt_bundle_attach` / `certificate`(+`certificate_len`) / `psk_hint_key` / `skip_cert_common_name_check` / `common_name` / `alpn_protos` / `ciphersuites_list`。

### credentials（凭据）
- `username` / `client_id` / `set_null_client_id`。
- `authentication`：`password` / `certificate`(+len) / `key`(+len) / `key_password`(+len) / `use_secure_element` / `ds_data` / `use_ecdsa_peripheral` / `ecdsa_key_efuse_blk`。

### session（会话）
- `last_will.{topic,msg,msg_len,qos,retain}`。
- `disable_clean_session`（默认 false=clean）、`keepalive`（默认 120s）、`disable_keepalive`、`protocol_ver`、`message_retransmit_timeout`（默认 1000ms）。

### network（网络）
- `reconnect_timeout_ms`（默认 10000）、`timeout_ms`（默认 10000）、`refresh_connection_after_ms`、`disable_auto_reconnect`、`tcp_keep_alive_cfg`、`transport`（自定义传输）、`if_name`。

### task / buffer / outbox
- `task.priority` / `task.stack_size`。
- `buffer.size`（默认 1024）/ `buffer.out_size`（默认同 size）。
- `outbox.limit`（字节预算）。

## scheme / 默认端口

| scheme | 传输 | 默认端口 |
|---|---|---|
| `mqtt` | TCP | 1883 |
| `mqtts` | SSL/TLS | 8883 |
| `ws` | WebSocket | 80 |
| `wss` | WebSocket Secure | 443 |

## idf_component.yml 依赖

```yaml
dependencies:
  espressif/mqtt: "*"
  idf:
    version: ">=5.3"
```
