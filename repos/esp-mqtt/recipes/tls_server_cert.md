# 服务端证书单向校验（mqtts://）

> **适用摘要**: 仅用服务端 CA 证书校验 broker 身份（不提供客户端证书），对应 `examples/ssl/`，是最常见的 TLS 用法。

> Version: selected repo/component version; align with the user project and dependency manifest.
> Validation: draft metadata added from repository routing; verify APIs, Kconfig symbols, and component dependencies against the selected version.

## 触发意图

- "MQTT SSL 连接"
- "mqtts CA 证书"
- "单向 TLS"
- "用证书包校验 broker"

## 前置条件

| 条件 | 要求 |
|---|---|
| Kconfig | `CONFIG_MQTT_TRANSPORT_SSL=y` |
| 文件 | broker 的 CA 证书（PEM） |
| 参考示例 | `examples/ssl/` |

## 分步说明

### 1. 获取服务端证书（来自 `docs/en/index.rst`）

```bash
openssl s_client -showcerts -connect mqtt.eclipseprojects.io:8883 < /dev/null \
  2> /dev/null | openssl x509 -outform PEM > mqtt_eclipse_org.pem
```

### 2. 嵌入 PEM

`main/CMakeLists.txt`：
```cmake
idf_component_register(SRCS "app_main.c"
                       INCLUDE_DIRS "."
                       PRIV_REQUIRES mqtt esp_partition nvs_flash esp_netif app_update
                       EMBED_TXTFILES mqtt_eclipse_org.pem)
```

> `examples/ssl/` 的 CMake 包含 `esp_partition` 与 `app_update`（用于证书包 / DS 场景）。

### 3. 配置（来自 `docs/en/index.rst`）

```c
extern const uint8_t mqtt_eclipse_org_pem_start[] asm("_binary_mqtt_eclipse_org_pem_start");
extern const uint8_t mqtt_eclipse_org_pem_end[]   asm("_binary_mqtt_eclipse_org_pem_end");

const esp_mqtt_client_config_t mqtt_cfg = {
    .broker = {
        .address.uri = "mqtts://mqtt.eclipseprojects.io:8883",
        .verification.certificate = (const char *)mqtt_eclipse_org_pem_start,
    },
};
esp_mqtt_client_handle_t client = esp_mqtt_client_init(&mqtt_cfg);
esp_mqtt_client_register_event(client, ESP_EVENT_ANY_ID, mqtt_event_handler, client);
esp_mqtt_client_start(client);
```

### 4. 替代校验方式

| 方式 | 配置 |
|---|---|
| 全局 CA 存储 | `.broker.verification.use_global_ca_store = true`（参考 esp-tls 文档初始化全局 store） |
| ESP x509 证书包 | `.broker.verification.crt_bundle_attach = esp_crt_bundle_attach`（需 `CONFIG_MBEDTLS_CERTIFICATE_BUNDLE=y`，IDF>=4.4） |
| 自定义 CN | `.broker.verification.common_name = "broker.example.com"` |

## 常见错误

| 错误 | 原因 | 解决方法 |
|---|---|---|
| 证书过期 | PEM 文件陈旧 | 重新用 openssl 获取最新证书 |
| `certificate_len` 误设 | PEM 应为 NULL 结尾字符串 | PEM 时留 `certificate_len=0` |
| 中间证书缺失 | 仅嵌了叶子证书 | 嵌入完整证书链或用证书包 |

## 参考

- `examples/ssl/main/app_main.c`
- `examples/ssl/README.md`
- `docs/en/index.rst`（Verification 节）
