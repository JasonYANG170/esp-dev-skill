# 双向 TLS 认证（mqtts://）

> **适用摘要**: 使用客户端证书 + 私钥 + 服务端 CA 实现 mqtts:// 双向认证，对应 `examples/ssl_mutual_auth/`。

> Version: selected repo/component version; align with the user project and dependency manifest.
> Validation: draft metadata added from repository routing; verify APIs, Kconfig symbols, and component dependencies against the selected version.

## 触发意图

- "MQTT 双向认证"
- "mqtts client certificate"
- "MQTT 证书 + 私钥"
- "mutual TLS MQTT"

## 前置条件

| 条件 | 要求 |
|---|---|
| Kconfig | `CONFIG_MQTT_TRANSPORT_SSL=y`（默认开） |
| 文件 | 服务端 CA 证书、客户端证书、客户端私钥（PEM） |
| 参考示例 | `examples/ssl_mutual_auth/` |

## 分步说明

### 1. 嵌入证书文件

将 `client.crt`、`client.key`、服务端 `mosquitto_org.crt` 放入 `main/certs/`，在 `main/CMakeLists.txt`：
```cmake
idf_component_register(SRCS "app_main.c"
                       INCLUDE_DIRS "."
                       PRIV_REQUIRES mqtt esp_wifi nvs_flash
                       EMBED_TXTFILES certs/client.crt
                                      certs/client.key
                                      certs/mosquitto_org.crt)
```

### 2. 引用 embed 符号

```c
extern const uint8_t client_cert_pem_start[] asm("_binary_client_crt_start");
extern const uint8_t client_cert_pem_end[]   asm("_binary_client_crt_end");
extern const uint8_t client_key_pem_start[]  asm("_binary_client_key_start");
extern const uint8_t client_key_pem_end[]    asm("_binary_client_key_end");
extern const uint8_t server_cert_pem_start[] asm("_binary_mosquitto_org_crt_start");
extern const uint8_t server_cert_pem_end[]   asm("_binary_mosquitto_org_crt_end");
```

### 3. 配置客户端（来自 `examples/ssl_mutual_auth/main/app_main.c`）

```c
const esp_mqtt_client_config_t mqtt_cfg = {
    .broker.address.uri = "mqtts://test.mosquitto.org:8884",
    .broker.verification.certificate = (const char *)server_cert_pem_start,
    .credentials = {
        .authentication = {
            .certificate = (const char *)client_cert_pem_start,
            .key = (const char *)client_key_pem_start,
        },
    },
};
esp_mqtt_client_handle_t client = esp_mqtt_client_init(&mqtt_cfg);
esp_mqtt_client_register_event(client, ESP_EVENT_ANY_ID, mqtt_event_handler, NULL);
esp_mqtt_client_start(client);
```

### 4. PEM 与 DER

- PEM：以 NULL 结尾字符串，设 `certificate` 字段，`certificate_len` 留 0。
- DER：设 `certificate` + `certificate_len`（实际长度）。
- 私钥若加密：设 `key_password` 与 `key_password_len`。

### 5. 高级校验字段（`broker.verification_t`）

| 字段 | 作用 |
|---|---|
| `use_global_ca_store` | 使用全局 CA 存储 |
| `crt_bundle_attach` | 附加 ESP x509 证书包（`esp_crt_bundle_attach`） |
| `skip_cert_common_name_check` | 跳过 CN 校验（降低安全性） |
| `common_name` | 显式指定期望 CN（NULL 则用 hostname） |
| `alpn_protos` | ALPN 协议列表（NULL 结尾） |
| `ciphersuites_list` | IANA 密码套件 ID 数组（零结尾，需 IDF>=5.5） |

## 常见错误

| 错误 | 原因 | 解决方法 |
|---|---|---|
| TLS 握手失败 | 服务端 CA 不匹配 | 用 `openssl s_client -connect host:8883` 取正确 CA |
| `esp_tls_cert_verify_flags` 非零 | CN 与 hostname 不符 | 设 `common_name` 或修正 hostname |
| embed 符号未定义 | 文件名与符号不对应 | `_binary_<去点>_start` 规则；重命名文件后重新构建 |
| 私钥需密码 | 未提供 | 设 `key_password` / `key_password_len` |

## 参考

- `examples/ssl_mutual_auth/main/app_main.c`
- `examples/ssl_mutual_auth/README.md`
- `docs/en/index.rst`（Verification / Authentication 节）
