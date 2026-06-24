# ESP-MQTT 高频陷阱汇总

> 来自仓库 `docs/en/index.rst`、`include/*.h` 函数文档与各 example 的真实注意事项。

## 初始化与生命周期

1. **顺序：init → register_event → start**。先 start 后注册会丢失 `MQTT_EVENT_BEFORE_CONNECT` 及早期事件。
2. **必须在 mqtt 之前初始化网络栈**：`nvs_flash_init()` → `esp_netif_init()` → `esp_event_loop_create_default()` → 联网（`example_connect()`）。
3. **`esp_mqtt_client_stop` / `esp_mqtt_client_destroy` 禁止在事件回调中调用**（头文件文档明确禁止），否则死锁；改在独立任务中调用。
4. **`esp_mqtt_client_reconnect` 仅在等待重连状态有效**，活动连接时返回 `ESP_FAIL`。`disable_auto_reconnect=true` 时也不能用它，需重新 `start`/`esp_mqtt_connect`。

## 数据与打印

5. **topic / data 不是 NULL 结尾**，必须用 `%.*s` + `topic_len`/`data_len` 打印，否则越界。
6. **大消息分片**：超过 `buffer.size` 触发多个 `MQTT_EVENT_DATA`，仅首事件带 topic；用 `current_data_offset` + `total_data_len` 重组。开启 `CONFIG_MQTT_TOPIC_PRESENT_ALL_DATA_EVENTS` 可每片带 topic（会额外分配内存）。

## QoS 与 outbox

7. **QoS0 不入 outbox**，断开时 publish 返回 -1；要离线缓存用 `esp_mqtt_client_enqueue(..., store=true)`。
8. **publish / subscribe 返回 -2 = outbox 已满**，必须处理（降速、丢弃或增 `outbox.limit`）。
9. **库默认始终重传未确认的 QoS1/2**（即使 MQTT 规范只要求 clean session=0 时），重传间隔 = `session.message_retransmit_timeout`（默认 1000ms）。
10. **outbox 过期**：超过 `CONFIG_MQTT_OUTBOX_EXPIRED_TIMEOUT_MS`（默认 30000ms）删除；`CONFIG_MQTT_REPORT_DELETED_MESSAGES=y` 时发 `MQTT_EVENT_DELETED`。
11. **outbox 容量估算**：`floor(limit / (topic_len + payload_len + ~4..6 开销))`。

## TLS / 安全

12. **mqtts:// 必须配 verification**：`certificate` / `use_global_ca_store` / `crt_bundle_attach` / `psk_hint_key` 至少一项，否则握手失败。
13. **PEM vs DER**：PEM 是 NULL 结尾字符串（`certificate_len=0`）；DER 必须给 `certificate_len` 实际长度。
14. **双向认证**：`credentials.authentication.certificate` + `.key` 同时设置；DS 外设用 `.ds_data`，ECDSA 用 `.use_ecdsa_peripheral` + `.ecdsa_key_efuse_blk`。
15. **PSK 仅在无其它校验方式时启用**，且需 IDF>=4.1。
16. **`skip_cert_common_name_check=true` 降低安全性**（易受 MITM）；如需指定 CN 用 `common_name`。
17. **证书用 embed 嵌入**（`EMBED_TXTFILES` / `EMBED_BINFILES`），符号 `_binary_<file>_start/_end`，避免运行期内存管理。

## MQTT 5

18. **`esp_mqtt5_*` API 需 `CONFIG_MQTT_PROTOCOL_5=y`**，且 `session.protocol_ver = MQTT_PROTOCOL_V_5`。
19. **`esp_mqtt5_client_set_user_property` 内部 malloc**，每次 set 后必须 `esp_mqtt5_client_delete_user_property`，否则泄露。
20. **publish/subscribe/unsubscribe/disconnect 的 property 为一次性**（不存储），每次操作前重新 set。
21. **访问 `event->property` 前判空**，且仅在 MQTT5 构建下；非数据事件可能无 property。
22. **CONNACK 服务端能力**在 `event->property->server`（仅 `MQTT_EVENT_CONNECTED`）：`max_qos`、`retain_available`、`wildcard_subscribe_available`、`shared_subscribe_available` 等。

## keepalive 与重连

23. **`keepalive=0` 不能关闭 keepalive**（用默认值）；要关闭设 `disable_keepalive=true`。
24. **keepalive 默认 120s**，客户端以设定值一半的间隔主动通信。
25. **默认事件队列为 1**（`CONFIG_MQTT_EVENT_QUEUE_SIZE`），处理慢可能丢早期事件；需排队可调大。

## 状态机

26. **状态查询**用 `esp_mqtt_client_get_state()`，返回 `esp_mqtt_client_connection_state_t`。
27. **`esp_mqtt_client_get_transport` 仅在 `MQTT_EVENT_BEFORE_CONNECT` 中调用**，返回句柄在 `destroy` 后失效。

## 事件 / 错误

28. **`MQTT_EVENT_DISCONNECTED` 总是最后一个事件**（当 ERROR + DISCONNECT 同时产生）。
29. **`esp_mqtt_client_stop` 不产生任何事件**（与 disconnect 不同）。
30. **解析错误先 switch `error_type`**：`MQTT_ERROR_TYPE_TCP_TRANSPORT` 读 `esp_tls_*` / `esp_transport_sock_errno`；`MQTT_ERROR_TYPE_CONNECTION_REFUSED` 读 `connect_return_code`。
