# WebSocket 与 WebSocket Secure（ws:// / wss://）

> **适用摘要**: 通过 WebSocket 传输连接 MQTT broker，对应 `examples/ws/`（ws）与 `examples/wss/`（wss）。常用于穿越 80/443 防火墙或走 CDN/反代。

> Version: selected repo/component version; align with the user project and dependency manifest.
> Validation: draft metadata added from repository routing; verify APIs, Kconfig symbols, and component dependencies against the selected version.

## 触发意图

- "MQTT over WebSocket"
- "ws:// MQTT"
- "wss:// MQTT"
- "443 端口 MQTT"

## 前置条件

| 条件 | 要求 |
|---|---|
| Kconfig | `CONFIG_MQTT_TRANSPORT_WEBSOCKET=y`（ws）；`CONFIG_MQTT_TRANSPORT_WEBSOCKET_SECURE=y`（wss，依赖 SSL+WS） |
| 参考示例 | `examples/ws/`、`examples/wss/` |

## 分步说明

### 1. ws:// 配置（来自 `examples/ws/main/app_main.c`）

```c
const esp_mqtt_client_config_t mqtt_cfg = {
    .broker.address.uri = CONFIG_BROKER_URI,   // 例如 ws://mqtt.eclipseprojects.io:80/mqtt
};
esp_mqtt_client_handle_t client = esp_mqtt_client_init(&mqtt_cfg);
esp_mqtt_client_register_event(client, ESP_EVENT_ANY_ID, mqtt_event_handler, NULL);
esp_mqtt_client_start(client);
```

URI 中的 `path`（如 `/mqtt`）对 WebSocket 很重要；若分字段配置，用 `broker.address.path`。

### 2. wss:// 配置

```c
const esp_mqtt_client_config_t mqtt_cfg = {
    .broker.address.uri = "wss://broker.example.com:443/mqtt",
    .broker.verification.certificate = (const char *)server_cert_pem_start,
};
```

wss = WebSocket + TLS，必须额外配置 `verification`（同 `tls_server_cert.md` / `tls_mutual_auth.md`）。

### 3. scheme 与端口

| scheme | 默认端口 | Kconfig |
|---|---|---|
| `ws` | 80 | `CONFIG_MQTT_TRANSPORT_WEBSOCKET` |
| `wss` | 443 | `CONFIG_MQTT_TRANSPORT_WEBSOCKET_SECURE` |

### 4. 日志建议

WebSocket 示例额外开启 `transport_ws` 日志：
```c
esp_log_level_set("transport_ws", ESP_LOG_VERBOSE);
```

## 常见错误

| 错误 | 原因 | 解决方法 |
|---|---|---|
| 握手 4xx | path 错误 | URI 加正确的 `/mqtt` 路径或设 `broker.address.path` |
| ws 不工作 | 未开 Kconfig | `CONFIG_MQTT_TRANSPORT_WEBSOCKET=y` |
| wss 证书错误 | 未配 verification | 同 TLS recipe 配置 CA |
| 连接被代理拦截 | 防火墙 | WebSocket 通常走 80/443 更易穿透 |

## 参考

- `examples/ws/main/app_main.c`
- `examples/wss/main/app_main.c`
- `examples/ws/README.md`、`examples/wss/README.md`
- `docs/en/index.rst`（Broker / Address 节，ws/wss 样例）
