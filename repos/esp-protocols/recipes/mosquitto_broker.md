# ESP32 板载 Mosquitto Broker

> **适用摘要**: 用 `mosquitto` 组件在 ESP32 上运行一个本地 MQTT broker —— 通过 `mosq_broker_run(&config)` 在调用线程阻塞运行；配置 `host`/`port`/`tls_cfg`（ESP-TLS 服务器配置）/`handle_connect_cb`（basic auth 校验）/`handle_message_cb`（消息回调）。可叠加本地 MQTT 客户端走 loopback 自测，或结合 serverless_mqtt 跨私有网络同步。

## 触发意图

- "ESP32 上跑 MQTT broker"
- "mosquitto 移植到 ESP32"
- "mosq_broker_run 用法"
- "板载 broker basic auth / TLS"
- "ESP32 作 MQTT 服务器"

## 前置条件

| 条件 | 要求 |
|---|---|
| ESP-IDF | v5.x，CMake 构建 |
| 组件依赖 | `espressif/mosquitto`（idf_component.yml 声明） |
| 网络栈 | `CONFIG_LIBC_NEWLIB=y`（broker 依赖 newlib） |
| 栈空间 | 调用任务至少 5 kB 栈；官方建议 ≥ 4 kB，推荐 8 kB+ 留余量 |
| 堆空间 | 启动约 2 kB；每个客户端约 4 kB（含异常断连重连场景额外余量） |
| 参考示例 | `components/mosquitto/examples/broker/`（TCP/TLS/basic auth + 本地客户端 loopback）、`components/mosquitto/examples/serverless_mqtt/`（双 ESP broker 跨 NAT 同步） |

## 分步说明

### 1. 组件依赖与栈配置

`main/idf_component.yml`：
```yaml
dependencies:
  espressif/mosquitto: "*"
  idf:
    version: ">=5.0"
```

`sdkconfig.defaults`：
```
CONFIG_LIBC_NEWLIB=y
```

broker 在调用线程阻塞运行（不创建独立任务），因此 `app_main` 所在任务栈需足够大；如要长生命周期运行，建议单独建任务：

```c
xTaskCreate(broker_task, "mqtt_broker", 1024 * 8, NULL, 5, NULL);   // 8 kB 栈
```

### 2. 最小 TCP broker（监听 0.0.0.0:1883）

```c
#include "nvs_flash.h"
#include "esp_netif.h"
#include "esp_event.h"
#include "esp_log.h"
#include "mosq_broker.h"
#include "protocol_examples_common.h"   // example_connect() 连 Wi-Fi/以太网

void app_main(void)
{
    ESP_ERROR_CHECK(nvs_flash_init());
    ESP_ERROR_CHECK(esp_netif_init());
    ESP_ERROR_CHECK(esp_event_loop_create_default());
    ESP_ERROR_CHECK(example_connect());

    struct mosq_broker_config config = {
        .host = CONFIG_EXAMPLE_BROKER_HOST,   // "0.0.0.0" 监听所有 IPv4
        .port = CONFIG_EXAMPLE_BROKER_PORT,   // 1883
        .tls_cfg = NULL,
        .handle_connect_cb = NULL,
    };

    mosq_broker_run(&config);   // 阻塞直到 mosq_broker_stop() 或异常退出
}
```

> `mosq_broker_run` 返回 int，0 表示��功。默认监听 `0.0.0.0`，可用同一网段的任何 MQTT 客户端连到 ESP32 的 IP:1883。

### 3. Basic Auth（handle_connect_cb 校验用户名/密码）

`handle_connect_cb` 签名为 `int(const char *client_id, const char *username, const char *password, int password_len)` —— 返回 0 接受连接，非 0 拒绝。

```c
#define EXAMPLE_USERNAME "testuser"
#define EXAMPLE_PASSWORD "testpass"

static int example_connect_callback(const char *client_id, const char *username,
                                    const char *password, int password_len)
{
    if (!username || strcmp(username, EXAMPLE_USERNAME) != 0) return 1;   // 拒绝
    if (!password || strcmp(password, EXAMPLE_PASSWORD) != 0) return 1;   // 拒绝
    return 0;   // 接受
}

// ...
struct mosq_broker_config config = {
    .host = "0.0.0.0",
    .port = 1883,
    .tls_cfg = NULL,
    .handle_connect_cb = example_connect_callback,   // 启用 basic auth
};
mosq_broker_run(&config);
```

### 4. 消息回调（handle_message_cb 监听所有发布）

```c
static void handle_message_cb(char *client, char *topic, char *data,
                              int len, int qos, int retain)
{
    ESP_LOGI(TAG, "client=%s topic=%s qos=%d retain=%d len=%d", client, topic, qos, retain, len);
    ESP_LOGI(TAG, "data=%.*s", len, data);
}

// ...
config.handle_message_cb = handle_message_cb;
```

### 5. TLS broker（esp_tls_cfg_server_t）

把服务器证书与私钥以 PEM 嵌入，通过 `esp_tls_cfg_server_t` 配置 `servercert_buf`/`serverkey_buf`：

```c
extern const unsigned char servercert_start[] asm("_binary_servercert_pem_start");
extern const unsigned char servercert_end[]   asm("_binary_servercert_pem_end");
extern const unsigned char serverkey_start[]  asm("_binary_serverkey_pem_start");
extern const unsigned char serverkey_end[]    asm("_binary_serverkey_pem_end");
extern const char cacert_start[] asm("_binary_cacert_pem_start");
extern const char cacert_end[]   asm("_binary_cacert_pem_end");

esp_tls_cfg_server_t tls_cfg = {
    .servercert_buf = servercert_start,
    .servercert_bytes = servercert_end - servercert_start,
    .serverkey_buf  = serverkey_start,
    .serverkey_bytes = serverkey_end - serverkey_start,
};
config.tls_cfg = &tls_cfg;
```

> TLS 配置类似 HTTPS server，详见 `idf.py docs -sp api-reference/protocols/esp_tls.html`。example 里的证书是自签名测试用（CN=127.0.0.1），勿用于生产。

### 6. 本地客户端 loopback 自测

broker 也可在同一芯片上跑一个 `esp_mqtt_client` 连 `127.0.0.1` 做发布/订阅自测：

```c
esp_mqtt_client_config_t mqtt_cfg = {
    .broker.address.hostname = "127.0.0.1",
    .broker.address.transport = MQTT_TRANSPORT_OVER_TCP,   // 或 SSL 对应 TLS broker
    .broker.address.port = config.port,
#if CONFIG_EXAMPLE_BROKER_USE_BASIC_AUTH
    .credentials.username = "testuser",
    .credentials.authentication.password = "testpass",
#endif
};
esp_mqtt_client_handle_t client = esp_mqtt_client_init(&mqtt_cfg);
esp_mqtt_client_register_event(client, ESP_EVENT_ANY_ID, mqtt_event_handler, NULL);
esp_mqtt_client_start(client);
```

> example 在 `MQTT_EVENT_BEFORE_CONNECT` 里 `vTaskDelay(1000)` 等 broker 起来；在 `MQTT_EVENT_CONNECTED` 里订阅 `/topic/qos0`，在 `MQTT_EVENT_SUBSCRIBED` 里 publish 一条 `"data"`。

### 7. 停止 broker

```c
mosq_broker_stop();   // 调用后 mosq_broker_run() 解除阻塞并返回
```

### 内存占用参考（来自 README）

- 程序空间约 60 kB Flash
- 启动堆约 2 kB；每客户端约 4 kB 堆
- 异常断连时旧连接信息保留一段时间才释放，期间重连会再占 4 kB —— 多客户端场景需留足堆余量
- 官方长期测试：5 客户端、每客户端 1 秒发 1 条、订阅全部主题、含 10 秒断重连

## 常见错误

| 错误 | 原因 | 解决方法 |
|---|---|---|
| `mosq_broker_run` 立即返回非 0 | 任务栈不足或 newlib 未启用 | 任务栈 ≥ 5 kB；`CONFIG_LIBC_NEWLIB=y` |
| 客户端连上但马上断开 | basic auth 回调返回非 0 | 校验回调返回 0 才接受；确认客户端传了正确的 username/password |
| TLS 客户端握手失败 | 证书与私钥不匹配或 CN 不符 | 用 `servercert`/`serverkey` 配对；客户端用 `cacert` 验证 |
| 多客户端下 broker 崩溃/OOM | 堆不够（异常断连重连占双倍） | 增大堆或限流；README 建议每客户端预留 4 kB+ |
| 本地客户端连不上 127.0.0.1 | broker 尚未就绪 | 客户端 `MQTT_EVENT_BEFORE_CONNECT` 里 `vTaskDelay` 等待；或先 `mosq_broker_run` 起来后再启动客户端 |
| 外部客户端连不上 ESP32 IP | broker host 只绑了 loopback | `host = "0.0.0.0"` 才监听所有接口 |

## 参考项目

- `components/mosquitto/examples/broker/main/example_broker.c` — TCP/TLS/basic auth + 本地客户端 loopback 完整示例
- `components/mosquitto/examples/broker/main/Kconfig.projbuild` — `CONFIG_EXAMPLE_BROKER_HOST/PORT/RUN_LOCAL_MQTT_CLIENT/USE_BASIC_AUTH/WITH_TLS`
- `components/mosquitto/examples/serverless_mqtt/main/serverless_mqtt.c` — 双 ESP broker 经 ICE/WebRTC 跨 NAT 同步（用 `handle_message_cb` 转发）
- `components/mosquitto/port/include/mosq_broker.h` — `mosq_broker_config`、`mosq_broker_run/stop`、`mosq_connect_cb_t`、`mosq_message_cb_t`
- `components/mosquitto/api.md` — API 文档（结构与回调字段说明）
- `components/mosquitto/README.md` — 内存占用与长期测试说明
