# PSK 预共享密钥认证（mqtts://）

> **适用摘要**: 在不支持证书或资源受限时，使用 TLS-PSK 预共享密钥认证 broker，对应 `examples/ssl_psk/`。PSK 仅在无其它校验方式时启用。

## 触发意图

- "MQTT PSK 认证"
- "预共享密钥 TLS"
- "不用证书的 mqtts"

## 前置条件

| 条件 | 要求 |
|---|---|
| Kconfig | `CONFIG_MQTT_TRANSPORT_SSL=y`；IDF>=4.1（`MQTT_SUPPORTED_FEATURE_PSK_AUTHENTICATION`） |
| broker | 需支持 PSK（如 mosquitto 配 `psk_file`） |
| 参考示例 | `examples/ssl_psk/` |

## 分步说明

### 1. 定义 PSK（来自 `examples/ssl_psk/main/app_main.c`）

```c
// mosquitto psk_file 内容示例:  hint:BAD123
static const uint8_t s_key[] = { 0xBA, 0xD1, 0x23 };

static const psk_hint_key_t psk_hint_key = {
    .key = s_key,
    .key_size = sizeof(s_key),
    .hint = "hint"
};
```

> `psk_hint_key_t` 定义在 `esp_tls.h`。

### 2. 配置客户端

```c
const esp_mqtt_client_config_t mqtt_cfg = {
    .broker.address.uri = EXAMPLE_BROKER_URI,   // mqtts://<broker>
    .broker.verification.psk_hint_key = &psk_hint_key,
};
esp_mqtt_client_handle_t client = esp_mqtt_client_init(&mqtt_cfg);
esp_mqtt_client_register_event(client, ESP_EVENT_ANY_ID, mqtt_event_handler, NULL);
esp_mqtt_client_start(client);
```

### 3. 要点

- PSK 仅在 `verification` 中无 `certificate` / `use_global_ca_store` / `crt_bundle_attach` 等其它校验时才生效。
- PSK 不校验 broker 身份，安全性弱于证书，仅在受控环境使用。

## 常见错误

| 错误 | 原因 | 解决方法 |
|---|---|---|
| 同时配了证书与 PSK | 证书优先，PSK 被忽略 | 仅保留 `psk_hint_key` |
| 握手失败 | hint/key 与 broker 不一致 | 核对 `psk_file` 的 hint 与 key |
| 编译找不到 psk_hint_key_t | 未 include esp_tls.h | `#include "esp_tls.h"` |

## 参考

- `examples/ssl_psk/main/app_main.c`
- `examples/ssl_psk/README.md`
- `include/mqtt_supported_features.h`（`MQTT_SUPPORTED_FEATURE_PSK_AUTHENTICATION`，IDF>=4.1）
