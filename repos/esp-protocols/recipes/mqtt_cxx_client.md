# esp_mqtt_cxx C++ MQTT 客户端

> **适用摘要**: 用 `esp_mqtt_cxx` 组件的 `idf::mqtt::Client` 封装类实现 MQTT 客户端 —— 继承 Client 重写 `on_connected`/`on_data` 等成员事件回调，通过 `BrokerConfiguration`/`ClientCredentials`/`Configuration` 三件套配置 broker、安全（明文/PEM/DER/PSK/Insecure/GlobalCAStore）与连接参数，使用 `subscribe`/`publish` 与 `Filter` 主题过滤器。支持 MQTT 3.11 与 TLS。

> Version: selected repo/component version; align with the user project and dependency manifest.
> Evidence: `repos/esp-protocols/resources/`, source/examples in `repos/esp-protocols/`, and this recipe path `repos/esp-protocols/recipes/mqtt_cxx_client.md`.
> Validation: draft metadata added from repository routing; verify APIs, Kconfig symbols, and component dependencies against the selected version.

## 触发意图

- "esp_mqtt_cxx C++ MQTT"
- "idf::mqtt::Client 用法"
- "MQTT C++ 客户端封装"
- "BrokerConfiguration / ClientCredentials / Configuration"
- "MQTT C++ TLS / LastWill / Filter"

## 前置条件

| 条件 | 要求 |
|---|---|
| ESP-IDF | v5.x，CMake 构建；目标芯片 ESP32 / ESP32-S2 / ESP32-S3 / ESP32-C2 / ESP32-C3 |
| 组件依赖 | `espressif/esp_mqtt_cxx`（idf_component.yml 声明） |
| menuconfig | `CONFIG_COMPILER_CXX_EXCEPTIONS=y`（类内部用异常做错误处理，否则头文件会 `#error`） |
| 网络 | 已连 Wi-Fi 或以太网（example 用 `example_connect()`） |
| 参考示例 | `components/esp_mqtt_cxx/examples/tcp/`（明文）、`components/esp_mqtt_cxx/examples/ssl/`（TLS） |

## 分步说明

### 1. 组件依赖与异常启用

`main/idf_component.yml`：
```yaml
dependencies:
  espressif/esp_mqtt_cxx: "*"
  idf:
    version: ">=5.0"
```

`sdkconfig.defaults`（必须启用 C++ 异常，否则 `esp_mqtt.hpp` 编译报 `#error MQTT class can only be used when __cpp_exceptions is enabled`）：
```
CONFIG_COMPILER_CXX_EXCEPTIONS=y
CONFIG_COMPILER_CXX_EXCEPTIONS_EMG_POOL_SIZE=1024
```

### 2. 继承 Client，重写 on_connected / on_data（纯虚）

`idf::mqtt::Client` 是基类，必须继承并提供 `on_connected` 与 `on_data`（两者为纯虚）。其余事件回调（`on_error`/`on_disconnected`/`on_subscribed`/`on_unsubscribed`/`on_published`/`on_before_connect`）有默认空实现，按需重写。

```cpp
#include "esp_mqtt.hpp"
#include "esp_mqtt_client_config.hpp"

namespace mqtt = idf::mqtt;

class MyClient final : public mqtt::Client {
public:
    using mqtt::Client::Client;   // 复用基类构造（BrokerConfiguration, ClientCredentials, Configuration）

private:
    void on_connected(esp_mqtt_event_handle_t const event) override
    {
        using mqtt::QoS;
        // 连接成功后订阅：Filter::get() 返回主题过滤字符串
        subscribe(messages.get());                       // 默认 QoS::AtLeastOnce
        subscribe(sent_load.get(), QoS::AtMostOnce);    // 显式 QoS
    }

    void on_data(esp_mqtt_event_handle_t const event) override
    {
        // 用 Filter::match() 判断事件主题是否命中过滤器（支持 + 与 # 通配符）
        if (messages.match(event->topic, event->topic_len)) {
            // 处理消息...
        }
    }

    mqtt::Filter messages{"$SYS/broker/messages/received"};
    mqtt::Filter sent_load{"$SYS/broker/load/+/sent"};   // 含 + 单层通配
};
```

### 3. 明文 TCP 连接（BrokerConfiguration + Insecure）

```cpp
#include "nvs_flash.h"
#include "esp_netif.h"
#include "esp_event.h"
#include "protocol_examples_common.h"

extern "C" void app_main(void)
{
    ESP_ERROR_CHECK(nvs_flash_init());
    ESP_ERROR_CHECK(esp_netif_init());
    ESP_ERROR_CHECK(esp_event_loop_create_default());
    ESP_ERROR_CHECK(example_connect());   // 连 Wi-Fi/以太网

    mqtt::BrokerConfiguration broker{
        .address = { mqtt::URI{ std::string{CONFIG_BROKER_URL} } },   // 用 URI 设 broker 地址
        .security = mqtt::Insecure{}                                   // 明文，不验证
    };
    mqtt::ClientCredentials credentials{};   // 默认无用户名/密码
    mqtt::Configuration config{};

    MyClient client{broker, credentials, config};
    client.start();   // 启动底层 esp_mqtt 客户端；必须在派生类构造完成后调用

    while (true) { vTaskDelay(pdMS_TO_TICKS(500)); }
}
```

### 4. TLS 连接（BrokerConfiguration + PEM 证书）

将 broker 的 CA 证书以 PEM 嵌入固件（`EMBED_TXTFILES` 或 `target_add_binary_data`），通过 `CryptographicInformation{PEM{...}}` 传入 `security`：

```cpp
extern const char broker_pem_start[] asm("_binary_mqtt_eclipseprojects_io_pem_start");
extern const char broker_pem_end[]   asm("_binary_mqtt_eclipseprojects_io_pem_end");

mqtt::BrokerConfiguration broker{
    .address  = { mqtt::URI{ std::string{CONFIG_BROKER_URI} } },     // 如 mqtts://mqtt.eclipseprojects.io:8883
    .security = mqtt::CryptographicInformation{ mqtt::PEM{broker_pem_start} }
};
mqtt::ClientCredentials credentials{};
mqtt::Configuration config{};

MyClient client{broker, credentials, config};
client.start();
```

> `BrokerAuthentication` 是 `std::variant<Insecure, GlobalCAStore, CryptographicInformation, PSK>`，按场景任选其一。`CryptographicInformation` 本身是 `std::variant<PEM, DER>`（DER 多一个 `len` 字段）。

### 5. 订阅 / 发布 / QoS / Retain

```cpp
// 订阅：返回 std::optional<MessageID>，失败为 std::nullopt
auto id = subscribe("sensor/temp", mqtt::QoS::ExactlyOnce);

// 发布：用 Message<Container>（StringMessage = Message<std::string>）
mqtt::StringMessage msg{ .data = "23.5", .qos = mqtt::QoS::AtLeastOnce, .retain = mqtt::Retain::Retained };
auto pub_id = publish("sensor/temp", msg);

// 也支持迭代器对：publish(topic, first, last, qos, retain)
char buf[4] = {'2','4','.','1'};
publish("sensor/temp", buf, buf + sizeof(buf), mqtt::QoS::AtMostOnce, mqtt::Retain::NotRetained);
```

枚举（来自 `esp_mqtt.hpp`，勿自行编号）：
```cpp
enum class QoS { AtMostOnce = 0, AtLeastOnce = 1, ExactlyOnce = 2 };
enum class Retain : bool { NotRetained = false, Retained = true };
```

### 6. 可选：LastWill / 用户名密码 / 自定义 client_id

通过 `ClientCredentials` 设凭据，通过 `Session.last_will` 设遗嘱。字段定义见 `esp_mqtt_client_config.hpp`：

```cpp
mqtt::ClientCredentials credentials{
    .username = std::string{"user"},
    .authentication = mqtt::Password{ std::string{"pass"} },   // 或 ClientCertificate / SecureElement
    .alpn_protos = { "x-esp-mqtt" },
    .client_id = std::string{"my-device-01"}
};

mqtt::Configuration config{
    .task = { .task_prio = 5, .task_stack = 6144 },
    .session = {
        .last_will = { .lwt_topic = "status/device01", .lwt_msg = "offline",
                       .lwt_qos = 1, .lwt_retain = 1, .lwt_msg_len = 7 },
        .disable_clean_session = 0,
        .keepalive = 120,
        .disable_keepalive = false,
        .protocol_ver = MQTT_PROTOCOL_V_3_1_1   // esp_mqtt_protocol_ver_t
    },
    .connection = {
        .transport = MQTT_TRANSPORT_OVER_SSL,
        .reconnect_timeout_ms = 10000,
        .network_timeout_ms = 10000,
        .refresh_connection_after_ms = 0,
        .disable_auto_reconnect = false
    },
    .user_context = nullptr,
    .buffer_size = 1024,
    .out_buffer_size = 0
};
```

### 7. 直接传 esp_mqtt_client_config_t（备选构造）

如果已有一份现成的 `esp_mqtt_client_config_t`（C 结构体），可直接构造 Client，跳过三件套拆解：

```cpp
esp_mqtt_client_config_t cfg = {};
cfg.broker.address.uri = "mqtt://broker.local";
cfg.credentials.client_id = "dev1";

MyClient client{cfg};   // Client(const esp_mqtt_client_config_t &config)
client.start();
```

## 常见错误

| 错误 | 原因 | 解决方法 |
|---|---|---|
| `#error MQTT class can only be used when __cpp_exceptions is enabled` | 未启用 C++ 异常 | `CONFIG_COMPILER_CXX_EXCEPTIONS=y` |
| 连接成功后不触发 `on_connected` | 忘记 `client.start()` | 派生类构造完成后必须显式 `start()` |
| 在构造函数体内 `subscribe` 失败 | 事件尚未分发 | `subscribe` 放在 `on_connected` 内（连接建立后订阅） |
| TLS 连接握手失败 | PEM/DER 未正确嵌入或 broker URI scheme 错 | 用 `mqtts://` + 正确的 `CryptographicInformation{PEM{...}}` |
| `Filter` 构造抛 `std::domain_error` | 主题过滤器非法（如多余 `/`、空段） | 校验过滤器格式，`+` 只能独占一层、`#` 只能放末尾 |
| 链接报 `undefined reference to esp_mqtt_client_*` | 未声明组件依赖 | `idf_component.yml` 加 `espressif/esp_mqtt_cxx`（会自动 select esp_mqtt） |

## 参考项目

- `components/esp_mqtt_cxx/examples/tcp/main/mqtt_tcp_example.cpp` — 明文 TCP 订阅 `$SYS/broker/...` 示例
- `components/esp_mqtt_cxx/examples/ssl/main/mqtt_ssl_example.cpp` — TLS（PEM 证书）示例
- `components/esp_mqtt_cxx/examples/tcp/sdkconfig.defaults` — `CONFIG_COMPILER_CXX_EXCEPTIONS=y` 配置
- `components/esp_mqtt_cxx/include/esp_mqtt.hpp` — `Client`、`Filter`、`Message`、`QoS`、`Retain` 定义
- `components/esp_mqtt_cxx/include/esp_mqtt_client_config.hpp` — `BrokerConfiguration`、`ClientCredentials`、`Configuration`、`LastWill`、`Session`、`Connection`、`Task` 等
- `docs/esp_mqtt_cxx/en/index.rst` — 组件文档首页（配置类清单）
