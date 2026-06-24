# ESP-MQTT API Quick Reference

> 全部签名来自 `include/mqtt_client.h`、`include/mqtt5_client.h`、`include/mqtt_supported_features.h`。MQTT5 相关 API 仅在 `CONFIG_MQTT_PROTOCOL_5=y` 时可用。

## 头文件

```c
#include "mqtt_client.h"     // 核心 API（include/mqtt_client.h）
// MQTT5: mqtt_client.h 会在 CONFIG_MQTT_PROTOCOL_5 时自动 #include "mqtt5_client.h"
```

## 类型与句柄

```c
typedef struct esp_mqtt_client *esp_mqtt_client_handle_t;
typedef esp_mqtt_event_t *esp_mqtt_event_handle_t;
typedef struct esp_mqtt_client *esp_mqtt5_client_handle_t;   // 同 client handle
```

## scheme 宏（`include/mqtt_client.h`）

```c
#define MQTT_OVER_TCP_SCHEME "mqtt"
#define MQTT_OVER_SSL_SCHEME "mqtts"
#define MQTT_OVER_WS_SCHEME  "ws"
#define MQTT_OVER_WSS_SCHEME "wss"
```

## 核心客户端 API

```c
// 生命周期
esp_mqtt_client_handle_t esp_mqtt_client_init(const esp_mqtt_client_config_t *config);
esp_err_t esp_mqtt_client_set_uri(esp_mqtt_client_handle_t client, const char *uri);
esp_err_t esp_mqtt_client_start(esp_mqtt_client_handle_t client);
esp_err_t esp_mqtt_client_reconnect(esp_mqtt_client_handle_t client);   // 仅在等待重连时
esp_err_t esp_mqtt_client_disconnect(esp_mqtt_client_handle_t client);
esp_err_t esp_mqtt_client_stop(esp_mqtt_client_handle_t client);        // 禁止在事件回调中调用
esp_err_t esp_mqtt_client_destroy(esp_mqtt_client_handle_t client);     // 禁止在事件回调中调用

// 配置 / 状态
esp_err_t esp_mqtt_set_config(esp_mqtt_client_handle_t client, const esp_mqtt_client_config_t *config);
esp_mqtt_client_connection_state_t esp_mqtt_client_get_state(esp_mqtt_client_handle_t client);
int esp_mqtt_client_get_outbox_size(esp_mqtt_client_handle_t client);

// 事件
esp_err_t esp_mqtt_client_register_event(esp_mqtt_client_handle_t client,
                                         esp_mqtt_event_id_t event,
                                         esp_event_handler_t event_handler,
                                         void *event_handler_arg);
esp_err_t esp_mqtt_client_unregister_event(esp_mqtt_client_handle_t client,
                                           esp_mqtt_event_id_t event,
                                           esp_event_handler_t event_handler);
esp_err_t esp_mqtt_dispatch_custom_event(esp_mqtt_client_handle_t client, esp_mqtt_event_t *event);

// 传输
esp_transport_handle_t esp_mqtt_client_get_transport(esp_mqtt_client_handle_t client,
                                                     char *transport_scheme);  // 仅在 MQTT_EVENT_BEFORE_CONNECT
```

## 订阅 / 发布

```c
// _Generic 宏（C）：按 topic_type 分派
#define esp_mqtt_client_subscribe(client_handle, topic_type, qos_or_size)

int esp_mqtt_client_subscribe_single(esp_mqtt_client_handle_t client, const char *topic, int qos);
int esp_mqtt_client_subscribe_multiple(esp_mqtt_client_handle_t client,
                                       const esp_mqtt_topic_t *topic_list, int size);
int esp_mqtt_client_unsubscribe(esp_mqtt_client_handle_t client, const char *topic);

int esp_mqtt_client_publish(esp_mqtt_client_handle_t client, const char *topic,
                            const char *data, int len, int qos, int retain);
int esp_mqtt_client_enqueue(esp_mqtt_client_handle_t client, const char *topic,
                            const char *data, int len, int qos, int retain, bool store);
```

返回值约定：`>=0` 成功（QoS0 publish 恒为 0），`-1` 失败，`-2` outbox 满。

## 枚举

### `esp_mqtt_event_id_t`
```c
MQTT_EVENT_ANY = -1
MQTT_EVENT_ERROR = 0
MQTT_EVENT_CONNECTED
MQTT_EVENT_DISCONNECTED
MQTT_EVENT_SUBSCRIBED
MQTT_EVENT_UNSUBSCRIBED
MQTT_EVENT_PUBLISHED
MQTT_EVENT_DATA
MQTT_EVENT_BEFORE_CONNECT
MQTT_EVENT_DELETED          // 需 CONFIG_MQTT_REPORT_DELETED_MESSAGES
MQTT_USER_EVENT
```

### `esp_mqtt_connect_return_code_t`
```c
MQTT_CONNECTION_ACCEPTED = 0
MQTT_CONNECTION_REFUSE_PROTOCOL
MQTT_CONNECTION_REFUSE_ID_REJECTED
MQTT_CONNECTION_REFUSE_SERVER_UNAVAILABLE
MQTT_CONNECTION_REFUSE_BAD_USERNAME
MQTT_CONNECTION_REFUSE_NOT_AUTHORIZED
```

### `esp_mqtt_error_type_t`
```c
MQTT_ERROR_TYPE_NONE = 0
MQTT_ERROR_TYPE_TCP_TRANSPORT
MQTT_ERROR_TYPE_CONNECTION_REFUSED
MQTT_ERROR_TYPE_SUBSCRIBE_FAILED
#define MQTT_ERROR_TYPE_ESP_TLS MQTT_ERROR_TYPE_TCP_TRANSPORT   // 兼容别名
```

### `esp_mqtt_transport_t`
```c
MQTT_TRANSPORT_UNKNOWN = 0x0
MQTT_TRANSPORT_OVER_TCP
MQTT_TRANSPORT_OVER_SSL
MQTT_TRANSPORT_OVER_WS
MQTT_TRANSPORT_OVER_WSS
```

### `esp_mqtt_protocol_ver_t`
```c
MQTT_PROTOCOL_UNDEFINED = 0
MQTT_PROTOCOL_V_3_1
MQTT_PROTOCOL_V_3_1_1
MQTT_PROTOCOL_V_5
```

### `esp_mqtt_client_connection_state_t`
```c
MQTT_CLIENT_STATE_NOT_INITIALIZED = 0
MQTT_CLIENT_STATE_NOT_STARTED
MQTT_CLIENT_STATE_DISCONNECTED
MQTT_CLIENT_STATE_CONNECTED
MQTT_CLIENT_STATE_WAITING_RECONNECT
```

## 关键结构体（`esp_mqtt_client_config_t` 嵌套）

| 路径 | 类型 | 说明 |
|---|---|---|
| `broker.address.uri` | `const char*` | 完整 URI（优先级最高） |
| `broker.address.hostname` | `const char*` | 主机名 |
| `broker.address.transport` | `esp_mqtt_transport_t` | 传输选择 |
| `broker.address.path` | `const char*` | URI 路径（WS 用） |
| `broker.address.port` | `uint32_t` | 端口 |
| `broker.verification.use_global_ca_store` | `bool` | 全局 CA store |
| `broker.verification.crt_bundle_attach` | `esp_err_t(*)(void*)` | 证书包 attach |
| `broker.verification.certificate` / `.certificate_len` | `const char*` / `size_t` | CA（PEM/DER） |
| `broker.verification.psk_hint_key` | `const psk_key_hint*` | PSK（esp_tls.h） |
| `broker.verification.skip_cert_common_name_check` | `bool` | 跳过 CN 校验 |
| `broker.verification.alpn_protos` | `const char**` | ALPN |
| `broker.verification.common_name` | `const char*` | 期望 CN |
| `broker.verification.ciphersuites_list` | `const int*` | 密码套件（IDF>=5.5） |
| `credentials.username` | `const char*` | 用户名 |
| `credentials.client_id` | `const char*` | client id（默认 ESP32_<chipid>） |
| `credentials.set_null_client_id` | `bool` | NULL client id |
| `credentials.authentication.password` | `const char*` | 密码 |
| `credentials.authentication.certificate` / `.certificate_len` | client 证书（双向） |
| `credentials.authentication.key` / `.key_len` | client 私钥 |
| `credentials.authentication.key_password` / `.key_password_len` | 私钥解密口令 |
| `credentials.authentication.use_secure_element` | `bool` | ATECC608A |
| `credentials.authentication.ds_data` | `void*` | 数字签名外设 |
| `credentials.authentication.use_ecdsa_peripheral` | `bool` | ECDSA 外设 |
| `credentials.authentication.ecdsa_key_efuse_blk` | `uint8_t` | ECDSA efuse 块 |
| `session.last_will.{topic,msg,msg_len,qos,retain}` | LWT |
| `session.disable_clean_session` | `bool` | 默认 clean=true |
| `session.keepalive` | `int` | 秒，默认 120 |
| `session.disable_keepalive` | `bool` | 关闭 keepalive |
| `session.protocol_ver` | `esp_mqtt_protocol_ver_t` | 协议版本 |
| `session.message_retransmit_timeout` | `int` | ms，默认 1000 |
| `network.reconnect_timeout_ms` | `int` | 默认 10000 |
| `network.timeout_ms` | `int` | 默认 10000 |
| `network.refresh_connection_after_ms` | `int` | 刷新连接 |
| `network.disable_auto_reconnect` | `bool` | 关闭自动重连 |
| `network.tcp_keep_alive_cfg` | `esp_transport_keep_alive_t` | TCP keepalive |
| `network.transport` | `esp_transport_handle_t` | 自定义传输 |
| `network.if_name` | `struct ifreq*` | 指定网卡 |
| `task.priority` / `task.stack_size` | `int` | 任务配置 |
| `buffer.size` / `buffer.out_size` | `int` | 收 / 发缓冲，默认 1024 |
| `outbox.limit` | `uint64_t` | outbox 字节上限 |

### `esp_mqtt_event_t` 字段
`event_id`、`client`、`data`、`data_len`、`total_data_len`、`current_data_offset`、`topic`、`topic_len`、`msg_id`、`session_present`、`error_handle`、`retain`、`qos`、`dup`、`protocol_ver`；MQTT5 额外：`reason_code`、`property`。

### `esp_mqtt_error_codes_t` 字段
`esp_tls_last_esp_err`、`esp_tls_stack_err`、`esp_tls_cert_verify_flags`、`error_type`、`connect_return_code`、`esp_transport_sock_errno`（MQTT5 还有 deprecated `disconnect_return_code`）。

### `esp_mqtt_topic_t`
```c
typedef struct topic_t {
    const char *filter;
    int qos;
} esp_mqtt_topic_t;
```

## MQTT 5 API（`include/mqtt5_client.h`，需 `CONFIG_MQTT_PROTOCOL_5`）

```c
esp_err_t esp_mqtt5_client_set_connect_property(esp_mqtt5_client_handle_t client,
        const esp_mqtt5_connection_property_config_t *connect_property);
esp_err_t esp_mqtt5_client_set_publish_property(esp_mqtt5_client_handle_t client,
        const esp_mqtt5_publish_property_config_t *property);
esp_err_t esp_mqtt5_client_set_subscribe_property(esp_mqtt5_client_handle_t client,
        const esp_mqtt5_subscribe_property_config_t *property);
esp_err_t esp_mqtt5_client_set_unsubscribe_property(esp_mqtt5_client_handle_t client,
        const esp_mqtt5_unsubscribe_property_config_t *property);
esp_err_t esp_mqtt5_client_set_disconnect_property(esp_mqtt5_client_handle_t client,
        const esp_mqtt5_disconnect_property_config_t *property);

// user property（内部 malloc，用完必 delete）
esp_err_t esp_mqtt5_client_set_user_property(mqtt5_user_property_handle_t *user_property,
        esp_mqtt5_user_property_item_t item[], uint8_t item_num);
esp_err_t esp_mqtt5_client_get_user_property(mqtt5_user_property_handle_t user_property,
        esp_mqtt5_user_property_item_t *item, uint8_t *item_num);
uint8_t esp_mqtt5_client_get_user_property_count(mqtt5_user_property_handle_t user_property);
void esp_mqtt5_client_delete_user_property(mqtt5_user_property_handle_t user_property);
```

### MQTT5 结构体
- `esp_mqtt5_connection_property_config_t`：`session_expiry_interval`、`maximum_packet_size`、`receive_maximum`、`topic_alias_maximum`、`request_resp_info`、`request_problem_info`、`user_property`、`will_delay_interval`、`message_expiry_interval`、`payload_format_indicator`、`content_type`、`response_topic`、`correlation_data`、`correlation_data_len`、`will_user_property`。
- `esp_mqtt5_publish_property_config_t`：`payload_format_indicator`、`message_expiry_interval`、`topic_alias`、`response_topic`、`correlation_data(_len)`、`content_type`、`user_property`。
- `esp_mqtt5_subscribe_property_config_t`：`subscribe_id`、`no_local_flag`、`retain_as_published_flag`、`retain_handle`、`is_share_subscribe`、`share_name`、`user_property`。
- `esp_mqtt5_unsubscribe_property_config_t`：`is_share_subscribe`、`share_name`、`user_property`。
- `esp_mqtt5_disconnect_property_config_t`：`session_expiry_interval`、`disconnect_reason`、`user_property`。
- `esp_mqtt5_event_property_t`：`payload_format_indicator`、`response_topic(_len)`、`correlation_data(_len)`、`content_type(_len)`、`subscribe_id`、`user_property`、`server`（`esp_mqtt5_server_resp_property_t`）。
- `esp_mqtt5_server_resp_property_t`：`maximum_packet_size`、`receive_maximum`、`topic_alias_maximum`、`max_qos`、`retain_available`、`wildcard_subscribe_available`、`subscribe_identifiers_available`、`shared_subscribe_available`、`response_info(_len)`。
- `esp_mqtt5_user_property_item_t`：`{ const char *key; const char *value; }`。
- `esp_mqtt5_reason_code_t`：见 `mqtt5_client.h`（`MQTT5_SUCCESS`、`MQTT5_NORMAL_DISCONNECT`、`MQTT5_GRANTED_QOS0/1/2`、`MQTT5_DISCONNECT_WITH_WILL`、`MQTT5_NOT_AUTHORIZED`、`MQTT5_SERVER_UNAVAILABLE`、`MQTT5_BAD_USERNAME_OR_PWD` 等共 50+ 项）。

## 支持 feature 宏（`include/mqtt_supported_features.h`，按 IDF 版本启用）

| 宏 | 最低 IDF |
|---|---|
| `MQTT_SUPPORTED_FEATURE_EVENT_LOOP` / `_SKIP_CRT_CMN_NAME_CHECK` | 3.3 |
| `MQTT_SUPPORTED_FEATURE_WS_SUBPROTOCOL` / `_TRANSPORT_ERR_REPORTING` | 4.0 |
| `MQTT_SUPPORTED_FEATURE_PSK_AUTHENTICATION` / `_DER_CERTIFICATES` / `_ALPN` / `_CLIENT_KEY_PASSWORD` | 4.1 |
| `MQTT_SUPPORTED_FEATURE_SECURE_ELEMENT` | 4.2 |
| `MQTT_SUPPORTED_FEATURE_DIGITAL_SIGNATURE` / `_TRANSPORT_SOCK_ERRNO_REPORTING` | 4.3 |
| `MQTT_SUPPORTED_FEATURE_CERTIFICATE_BUNDLE` | 4.4 |
| `MQTT_SUPPORTED_FEATURE_CRT_CMN_NAME` | 5.1 |
| `MQTT_SUPPORTED_FEATURE_ECDSA_PERIPHERAL` | 5.2 |
| `MQTT_SUPPORTED_FEATURE_CIPHERSUITES_LIST` | 5.5 |

## 数字签名（DS）外设辅助 API

> 用于 `recipes/tls_digital_signature.md`。头文件 `esp_secure_cert_read.h` 来自组件 `espressif/esp_secure_cert_mgr`（在 `main/idf_component.yml` 声明依赖）。`ds_data` 字段在 `esp_mqtt_client_config_t.credentials.authentication` 中类型为 `void *`，客户端不 copy 也不 free。

```c
#include "esp_secure_cert_read.h"   // 来自 esp_secure_cert_mgr 组件

/* 仅在 CONFIG_ESP_SECURE_CERT_DS_PERIPHERAL 下可用；读取 subtype 0 的 DS 上下文 */
esp_ds_data_ctx_t *esp_secure_cert_get_ds_ctx(void);          // 成功返回指针，失败返回 NULL
void esp_secure_cert_free_ds_ctx(esp_ds_data_ctx_t *ds_ctx);  // 释放 DS 上下文内存

/* 设备证书：NVS 分区动态分配（需配对 free），cust_flash 返回只读 flash 指针 */
esp_err_t esp_secure_cert_get_device_cert(char **buffer, uint32_t *len);  // ESP_OK / 错误
esp_err_t esp_secure_cert_free_device_cert(char *buffer);
```

`esp_ds_data_ctx_t` 定义（`psa_crypto_driver_esp_rsa_ds_contexts.h`，IDF mbedtls port）：

```c
typedef struct {
    esp_ds_data_t *esp_ds_data;     /* 指向 esp ds data */
    uint8_t efuse_key_id;           /* DS 外设 HMAC 密钥所在 efuse 块 id（如 0,1） */
    uint16_t rsa_length_bits;       /* RSA 私钥长度（位，如 2048） */
} esp_ds_data_ctx_t;
```

典型 MQTT 客户端配置（`.key = NULL`，改用 `.ds_data`）：

```c
esp_ds_data_ctx_t *ds_data = esp_secure_cert_get_ds_ctx();
char *device_cert = NULL; uint32_t len = 0;
esp_secure_cert_get_device_cert(&device_cert, &len);

const esp_mqtt_client_config_t cfg = {
    .broker.address.uri = "mqtts://broker:8884",
    .broker.verification.certificate = (const char *)server_cert_pem_start,
    .credentials.authentication = {
        .certificate = (const char *)device_cert,
        .key = NULL,
        .ds_data = (void *)ds_data,
    },
};
```

> Provisioning 工具：`pip install esp-secure-cert-tool`，用 `configure_esp_secure_cert.py --configure_ds` 生成 `esp_secure_cert.bin` 分区镜像。详见 https://github.com/espressif/esp_secure_cert_mgr/tree/main/tools 。
