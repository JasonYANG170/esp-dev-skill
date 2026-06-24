# MQTT 客户端（TCP/SSL/双向认证/PSK/WS/WSS）

> **适用摘要**: 在已联网的 ESP8266 上用 esp-mqtt 组件连接 MQTT broker，支持 6 种传输（`mqtt://`、`mqtts://`、双向认证、PSK、`ws://`、`wss://`），基于事件回调处理连接/订阅/发布/数据；客户端在独立内部任务中运行，应用只通过 `esp_mqtt_client_register_event` 注册回调。

## 触发意图

- "MQTT"
- "连 MQTT broker"
- "订阅/发布主题"
- "MQTT SSL / 双向认证 / PSK / WebSocket"

## 前置条件

| 条件 | 要求 |
|---|---|
| 参考示例 | `examples/protocols/mqtt/{tcp,ssl,ssl_mutual_auth,ssl_psk,ws,wss}` |
| 联网 | 先 `example_connect()`（`common_components/protocol_examples_common`）；底层 `esp_netif_init()` + `esp_event_loop_create_default()` |
| 组件 | `mqtt_client.h`（由 `components/mqtt` 提供，内部编译 `esp-mqtt/mqtt_client.c` 等） |
| 配置 | menuconfig → `Example Configuration` 填 broker URL（`CONFIG_BROKER_URL` / `CONFIG_BROKER_URI`）；SSL/WSS 需 `CONFIG_MQTT_TRANSPORT_SSL=y`、WS 需 `CONFIG_MQTT_TRANSPORT_WEBSOCKET=y`、WSS 需 `CONFIG_MQTT_TRANSPORT_WEBSOCKET_SECURE=y`（均默认开） |
| 栈 | MQTT 内部任务栈默认 `CONFIG_MQTT_TASK_STACK_SIZE=6144`；自定义需开 `CONFIG_MQTT_USE_CUSTOM_CONFIG` |

## 分步说明

> **配置字段说明**：`esp_mqtt_client_config_t` 是嵌套结构体（见头文件 `esp-mqtt/include/mqtt_client.h`）：`broker.address`（含 `uri/hostname/transport/path/port`）、`broker.verification`（含 `use_global_ca_store/certificate/certificate_len/psk_hint_key/skip_cert_common_name_check/common_name/ciphersuites_list/alpn_protos/crt_bundle_attach`）、`credentials`（含 `username/client_id/set_null_client_id` 与 `authentication.{certificate,key,password,...}`）、`session`（含 `keepalive/disable_keepalive/disable_clean_session/protocol_ver/message_retransmit_timeout/last_will`）、`network`、`task`、`buffer`、`outbox`。
>
> ESP8266 RTOS SDK 自带的 6 个示例（`examples/protocols/mqtt/*`）写于较早版本，使用扁平的兼容简写（`.uri`、`.cert_pem`、`.client_cert_pem`、`.client_key_pem`、`.psk_hint_key`）；这些简写在当前头文件中分别对应 `broker.address.uri`、`broker.verification.certificate`、`credentials.authentication.certificate`、`credentials.authentication.key`、`broker.verification.psk_hint_key`。下面代码以当前头文件的嵌套字段为权威写法，并在对应小节标注示例的原始简写。

### 1. app_main 初始化（改编自 tcp 示例 `app_main`）

```c
#include "nvs_flash.h"
#include "esp_netif.h"
#include "esp_event.h"
#include "esp_log.h"
#include "protocol_examples_common.h"
#include "mqtt_client.h"

static const char *TAG = "MQTT_EXAMPLE";

void app_main(void)
{
    ESP_ERROR_CHECK(nvs_flash_init());
    ESP_ERROR_CHECK(esp_netif_init());
    ESP_ERROR_CHECK(esp_event_loop_create_default());
    ESP_ERROR_CHECK(example_connect());          // 连 WiFi（或以太网）

    mqtt_app_start();
}
```

### 2. 事件回调（switch on `event->event_id`）

所有 6 个示例都用同一套事件派发：`esp_event` 把事件投递给 `mqtt_event_handler`，它转调 `mqtt_event_handler_cb(event_data)`，在回调里对 `event->event_id` 做 switch。`esp_mqtt_event_t` 真实字段：`event_id`、`client`、`data`、`data_len`、`total_data_len`、`current_data_offset`、`topic`、`topic_len`、`msg_id`、`session_present`、`error_handle`（`esp_mqtt_error_codes_t *`，含 `error_type`、`esp_tls_last_esp_err`、`esp_tls_stack_err`、`esp_tls_cert_verify_flags`、`connect_return_code`、`esp_transport_sock_errno`）、`retain`、`qos`、`dup`。

```c
static esp_err_t mqtt_event_handler_cb(esp_mqtt_event_handle_t event)
{
    esp_mqtt_client_handle_t client = event->client;
    int msg_id;
    switch (event->event_id) {
        case MQTT_EVENT_CONNECTED:
            ESP_LOGI(TAG, "MQTT_EVENT_CONNECTED");
            // 在 CONNECTED 里发布/订阅（示例真实调用顺序）
            msg_id = esp_mqtt_client_publish(client, "/topic/qos1", "data_3", 0, 1, 0);
            msg_id = esp_mqtt_client_subscribe(client, "/topic/qos0", 0);
            msg_id = esp_mqtt_client_subscribe(client, "/topic/qos1", 1);
            msg_id = esp_mqtt_client_unsubscribe(client, "/topic/qos1");
            break;
        case MQTT_EVENT_DISCONNECTED:
            ESP_LOGI(TAG, "MQTT_EVENT_DISCONNECTED");
            break;
        case MQTT_EVENT_SUBSCRIBED:
            ESP_LOGI(TAG, "MQTT_EVENT_SUBSCRIBED, msg_id=%d", event->msg_id);
            break;
        case MQTT_EVENT_UNSUBSCRIBED:
            ESP_LOGI(TAG, "MQTT_EVENT_UNSUBSCRIBED, msg_id=%d", event->msg_id);
            break;
        case MQTT_EVENT_PUBLISHED:
            ESP_LOGI(TAG, "MQTT_EVENT_PUBLISHED, msg_id=%d", event->msg_id);
            break;
        case MQTT_EVENT_DATA:
            printf("TOPIC=%.*s\r\n", event->topic_len, event->topic);
            printf("DATA=%.*s\r\n", event->data_len, event->data);
            break;
        case MQTT_EVENT_ERROR:
            ESP_LOGI(TAG, "MQTT_EVENT_ERROR");
            if (event->error_handle->error_type == MQTT_ERROR_TYPE_TCP_TRANSPORT) {
                ESP_LOGI(TAG, "esp-tls last err: 0x%x, stack err: 0x%x",
                         event->error_handle->esp_tls_last_esp_err,
                         event->error_handle->esp_tls_stack_err);
            } else if (event->error_handle->error_type == MQTT_ERROR_TYPE_CONNECTION_REFUSED) {
                ESP_LOGI(TAG, "Connection refused: 0x%x",
                         event->error_handle->connect_return_code);
            }
            break;
        default:
            ESP_LOGI(TAG, "Other event id:%d", event->event_id);
            break;
    }
    return ESP_OK;
}

static void mqtt_event_handler(void *handler_args, esp_event_base_t base,
                               int32_t event_id, void *event_data)
{
    mqtt_event_handler_cb(event_data);
}
```

> ssl 示例的 `MQTT_EVENT_ERROR` 分支用兼容宏 `MQTT_ERROR_TYPE_ESP_TLS`（头文件定义为 `MQTT_ERROR_TYPE_TCP_TRANSPORT`），等价。

### 3. TCP 客户端：`init` + `register_event` + `start`

tcp 示例只设 `broker.address.uri`（示例里写作 `.uri = CONFIG_BROKER_URL`）：

```c
static void mqtt_app_start(void)
{
    const esp_mqtt_client_config_t mqtt_cfg = {
        .broker.address.uri = CONFIG_BROKER_URL,          // "mqtt://broker:1883"
    };
    esp_mqtt_client_handle_t client = esp_mqtt_client_init(&mqtt_cfg);
    esp_mqtt_client_register_event(client, ESP_EVENT_ANY_ID, mqtt_event_handler, client);
    esp_mqtt_client_start(client);
}
```

### 4. SSL/TLS（服务端 CA 校验）

ssl 示例嵌入 server 根证书（`mqtt_eclipse_org_pem_start[]`，由 `target_add_binary_data` 生成 `_binary_mqtt_eclipse_org_pem_start/end`），配置用 `broker.verification.certificate`（示例简写 `.cert_pem`）：

```c
extern const uint8_t mqtt_eclipse_org_pem_start[] asm("_binary_mqtt_eclipse_org_pem_start");
extern const uint8_t mqtt_eclipse_org_pem_end[]   asm("_binary_mqtt_eclipse_org_pem_end");

static void mqtt_app_start(void)
{
    const esp_mqtt_client_config_t mqtt_cfg = {
        .broker.address.uri = CONFIG_BROKER_URI,         // "mqtts://broker:8883"
        .broker.verification.certificate = (const char *)mqtt_eclipse_org_pem_start,
    };
    esp_mqtt_client_handle_t client = esp_mqtt_client_init(&mqtt_cfg);
    esp_mqtt_client_register_event(client, ESP_EVENT_ANY_ID, mqtt_event_handler, client);
    esp_mqtt_client_start(client);
}
```

> 也可改用 `broker.verification.use_global_ca_store = true`，或 `broker.verification.crt_bundle_attach`（x509 证书束）、`broker.verification.common_name`、`broker.verification.skip_cert_common_name_check`、`broker.verification.ciphersuites_list`、`broker.verification.alpn_protos`。

### 5. SSL 双向认证（client cert + key）

ssl_mutual_auth 示例把 `client.crt`/`client.key` 放进 `main/`，用 `credentials.authentication.certificate` + `credentials.authentication.key`（示例简写 `.client_cert_pem` / `.client_key_pem`）。该示例用旧式 `event_handle` 直接回调而非 `register_event`，下面给出与当前头文件一致的等价写法：

```c
extern const uint8_t client_cert_pem_start[] asm("_binary_client_crt_start");
extern const uint8_t client_cert_pem_end[]   asm("_binary_client_crt_end");
extern const uint8_t client_key_pem_start[]  asm("_binary_client_key_start");
extern const uint8_t client_key_pem_end[]    asm("_binary_client_key_end");

static void mqtt_app_start(void)
{
    const esp_mqtt_client_config_t mqtt_cfg = {
        .broker.address.uri = "mqtts://test.mosquitto.org:8884",
        .credentials.authentication.certificate = (const char *)client_cert_pem_start,
        .credentials.authentication.key         = (const char *)client_key_pem_start,
    };
    esp_mqtt_client_handle_t client = esp_mqtt_client_init(&mqtt_cfg);
    esp_mqtt_client_register_event(client, ESP_EVENT_ANY_ID, mqtt_event_handler, client);
    esp_mqtt_client_start(client);
}
```

> 若私钥带口令，可加 `credentials.authentication.key_password` + `key_password_len`；ESP8266 不支持 `use_secure_element`/`ds_data`/`use_ecdsa_peripheral`（这些是 ESP32/ESP32-S3/C6 字段）。证书也可用 DER：此时须同时给 `certificate_len`/`key_len`。

### 6. TLS-PSK（预共享密钥）

ssl_psk 示例定义 `psk_hint_key_t`（来自 `esp_tls.h`，字段 `key` / `key_size` / `hint`），通过 `broker.verification.psk_hint_key` 传入（示例简写 `.psk_hint_key`）：

```c
#include "esp_tls.h"

static const uint8_t s_key[] = { 0xBA, 0xD1, 0x23 };     // 与 broker 的 psk_file 一致
static const psk_hint_key_t psk_hint_key = {
    .key = s_key,
    .key_size = sizeof(s_key),
    .hint = "hint",
};

static void mqtt_app_start(void)
{
    const esp_mqtt_client_config_t mqtt_cfg = {
        .broker.address.uri = "mqtts://192.168.0.2",     // 支持 PSK 的 broker
        .broker.verification.psk_hint_key = &psk_hint_key,
    };
    esp_mqtt_client_handle_t client = esp_mqtt_client_init(&mqtt_cfg);
    esp_mqtt_client_register_event(client, ESP_EVENT_ANY_ID, mqtt_event_handler, client);
    esp_mqtt_client_start(client);
}
```

> 头文件注明：PSK 仅在“没有其他方式校验 broker”时启用（即未设 certificate / use_global_ca_store / crt_bundle_attach 时才生效）。

### 7. WebSocket（`ws://`）

ws 示例只设 `broker.address.uri`，路径写在 URI 里（示例简写 `.uri = CONFIG_BROKER_URI`，如 `ws://host:80/mqtt`）：

```c
const esp_mqtt_client_config_t mqtt_cfg = {
    .broker.address.uri = "ws://broker:80/mqtt",         // path 写在 URI
};
esp_mqtt_client_handle_t client = esp_mqtt_client_init(&mqtt_cfg);
esp_mqtt_client_register_event(client, ESP_EVENT_ANY_ID, mqtt_event_handler, client);
esp_mqtt_client_start(client);
```

### 8. WebSocket Secure（`wss://` + CA）

wss 示例在 ws 基础上加 server CA（`broker.verification.certificate`，示例简写 `.cert_pem`）：

```c
extern const uint8_t mqtt_eclipse_org_pem_start[] asm("_binary_mqtt_eclipse_org_pem_start");

const esp_mqtt_client_config_t mqtt_cfg = {
    .broker.address.uri = "wss://broker:443/mqtt",
    .broker.verification.certificate = (const char *)mqtt_eclipse_org_pem_start,
};
```

### 9. 传输选择速查

scheme 由 URI 前缀决定，对应头文件 `esp_mqtt_transport_t`（`MQTT_TRANSPORT_OVER_TCP/SSL/WS/WSS`）：

| scheme | 传输 | 示例 URI | 关键配置字段 |
|---|---|---|---|
| `mqtt://` | TCP 明文 | `mqtt://broker:1883` | `broker.address.uri` |
| `mqtts://` | TLS（服务端证书校验） | `mqtts://broker:8883` | `broker.address.uri` + `broker.verification.certificate` |
| `mqtts://` 双向 | TLS 双向认证 | `mqtts://test.mosquitto.org:8884` | + `credentials.authentication.certificate` / `.key` |
| `mqtts://` PSK | TLS-PSK | `mqtts://192.168.0.2` | + `broker.verification.psk_hint_key` |
| `ws://` | WebSocket | `ws://broker:80/mqtt` | `broker.address.uri`（path 在 URI） |
| `wss://` | WebSocket Secure | `wss://broker:443/mqtt` | `broker.address.uri` + `broker.verification.certificate` |

### 10. 关键 API（签名取自 `mqtt_client.h`）

| API | 签名 | 作用 |
|---|---|---|
| `esp_mqtt_client_init` | `esp_mqtt_client_handle_t esp_mqtt_client_init(const esp_mqtt_client_config_t *config)` | 创建客户端（返回 NULL 出错） |
| `esp_mqtt_client_register_event` | `esp_err_t esp_mqtt_client_register_event(client, esp_mqtt_event_id_t event, esp_event_handler_t cb, void *arg)` | 注册事件（`ESP_EVENT_ANY_ID` 收全部） |
| `esp_mqtt_client_start` | `esp_err_t esp_mqtt_client_start(client)` | 启动内部任务、开始建连 |
| `esp_mqtt_client_publish` | `int esp_mqtt_client_publish(client, topic, data, len, qos, retain)` | 发布（QoS0 的 msg_id 恒为 0；-1 失败、-2 outbox 满） |
| `esp_mqtt_client_subscribe_single` | `int esp_mqtt_client_subscribe_single(client, topic, qos)` | 订阅单个主题 |
| `esp_mqtt_client_subscribe` | `_Generic` 宏，`char*` → `subscribe_single`，`esp_mqtt_topic_t*` → `subscribe_multiple` | 便捷订阅 |
| `esp_mqtt_client_unsubscribe` | `int esp_mqtt_client_unsubscribe(client, topic)` | 取消订阅 |
| `esp_mqtt_client_enqueue` | `int esp_mqtt_client_enqueue(client, topic, data, len, qos, retain, bool store)` | 入队（非阻塞 publish） |
| `esp_mqtt_client_stop` | `esp_err_t esp_mqtt_client_stop(client)` | 停止任务（不能在事件回调里调用） |
| `esp_mqtt_client_destroy` | `esp_err_t esp_mqtt_client_destroy(client)` | 销毁句柄（不能在事件回调里调用） |
| `esp_mqtt_client_reconnect` | `esp_err_t esp_mqtt_client_reconnect(client)` | 强制重连 |
| `esp_mqtt_client_disconnect` | `esp_err_t esp_mqtt_client_disconnect(client)` | 强制断开 |
| `esp_mqtt_client_set_uri` | `esp_err_t esp_mqtt_client_set_uri(client, const char *uri)` | 覆盖 init 时的 URI |
| `esp_mqtt_set_config` | `esp_err_t esp_mqtt_set_config(client, const esp_mqtt_client_config_t *config)` | 运行期改配置 |

### 11. 事件 id 速查（`esp_mqtt_event_id_t`）

| 事件 | 含义 |
|---|---|
| `MQTT_EVENT_BEFORE_CONNECT` | 已初始化、即将建连（可在此用 `esp_mqtt_client_get_transport` 调传输层） |
| `MQTT_EVENT_CONNECTED` | CONNACK 成功（带 `session_present`） |
| `MQTT_EVENT_DISCONNECTED` | 断开 |
| `MQTT_EVENT_SUBSCRIBED` | 订阅 ACK（带 `msg_id`） |
| `MQTT_EVENT_UNSUBSCRIBED` | 退订 ACK（带 `msg_id`） |
| `MQTT_EVENT_PUBLISHED` | 发布 ACK（QoS≥1，带 `msg_id`） |
| `MQTT_EVENT_DATA` | 收到消息：`topic/topic_len`、`data/data_len`、`total_data_len`、`current_data_offset`（长消息会拆多次） |
| `MQTT_EVENT_ERROR` | 传输层错误，查 `error_handle->error_type` |
| `MQTT_EVENT_DELETED` | outbox 消息过期删除（需 `MQTT_REPORT_DELETED_MESSAGES`） |

## 常见错误

| 错误 | 原因 | 解决方法 |
|---|---|---|
| 一直 `MQTT_EVENT_ERROR` | broker URI 错 / 没拿到 IP | 先确认 WiFi 已 `IP_EVENT_STA_GOT_IP` 再 `start` |
| TLS 握手失败 | CA 不匹配 / 过期 | 重新下载根证书并嵌入 `broker.verification.certificate` |
| 双向认证失败 | client cert/key 不对或 `_binary_*_start` 符号名写错 | 符号名与文件名对应（如 `client.crt` → `_binary_client_crt_start`） |
| WSS 连不上 | 漏 server CA | WSS 必须传 `broker.verification.certificate` |
| 连上但收不到消息 | 没 subscribe / topic 写错 | 在 `MQTT_EVENT_CONNECTED` 里 subscribe |
| WS 路径 404 | URI 没带 path | ws/wss 的 path 写在 URI（如 `ws://host:80/mqtt`） |
| PSK 握手失败 | hint/key 与 broker 不一致；或同时设了 CA | 两端 `psk_hint_key` 完全一致；PSK 仅在未设证书时生效 |
| 字段名不存在（`.cert_pem` 等） | 用了旧扁平字段而头文件是嵌套版 | 改用 `broker.verification.certificate` / `credentials.authentication.key` 等 |
| `stop`/`destroy` 死锁 | 在事件回调里调用 | 这两个 API 不能在 MQTT 事件回调里调用（见头文件 Notes） |
| 栈溢出 | MQTT 任务栈太小 | 开 `CONFIG_MQTT_USE_CUSTOM_CONFIG`，调大 `CONFIG_MQTT_TASK_STACK_SIZE`（默认 6144） |

## 参考项目

- `examples/protocols/mqtt/tcp/main/app_main.c` — TCP（事件派发：`register_event` + `ESP_EVENT_ANY_ID`）
- `examples/protocols/mqtt/ssl/main/app_main.c` — SSL（`cert_pem`，`MQTT_EVENT_ERROR` 分支读 `error_handle`）
- `examples/protocols/mqtt/ssl_mutual_auth/main/app_main.c` — 双向认证（`client_cert_pem` / `client_key_pem`，旧式 `event_handle`）
- `examples/protocols/mqtt/ssl_psk/main/app_main.c` — PSK（`psk_hint_key_t`，`psk_file: hint:BAD123`）
- `examples/protocols/mqtt/ws/main/app_main.c` — WebSocket
- `examples/protocols/mqtt/wss/main/app_main.c` — WSS（带 server CA）
- `components/mqtt/Kconfig` — `CONFIG_MQTT_TRANSPORT_SSL/WEBSOCKET/WEBSOCKET_SECURE`、`CONFIG_MQTT_TASK_STACK_SIZE`、`CONFIG_MQTT_USE_CUSTOM_CONFIG`、`CONFIG_MQTT_BUFFER_SIZE`、`CONFIG_MQTT_PROTOCOL_311`
- esp-mqtt 头文件（`components/mqtt/esp-mqtt/include/mqtt_client.h`，本仓库以 git submodule 引入）：`esp_mqtt_client_config_t` 嵌套结构、`esp_mqtt_event_id_t`、`esp_mqtt_error_codes_t`、`psk_hint_key_t`（`esp_tls.h`）
