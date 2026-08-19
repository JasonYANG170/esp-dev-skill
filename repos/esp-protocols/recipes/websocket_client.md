# WebSocket 客户端

> **适用摘要**: 用 `esp_websocket_client` 建立 ws/wss 连接，处理连接/数据/错误事件，收发文本、二进制与分片帧，配置 TLS（证书包/双向认证）。

> Version: selected repo/component version; align with the user project and dependency manifest.
> Evidence: `repos/esp-protocols/resources/`, source/examples in `repos/esp-protocols/`, and this recipe path `repos/esp-protocols/recipes/websocket_client.md`.
> Validation: draft metadata added from repository routing; verify APIs, Kconfig symbols, and component dependencies against the selected version.

## 触发意图

- "WebSocket 客户端"
- "esp_websocket_client"
- "ws/wss 连接"
- "发送 WebSocket 文本/二进制帧"
- "WebSocket 分片发送"

## 前置条件

| 条件 | 要求 |
|---|---|
| 网络 | 已连 Wi-Fi 或以太网 |
| 组件依赖 | `espressif/esp_websocket_client`（>=1.7.0） |
| 参考示例 | `components/esp_websocket_client/examples/target/main/websocket_example.c` |

## 分步说明

### 1. 初始化网络与事件循环

```c
#include "nvs_flash.h"
#include "esp_netif.h"
#include "esp_event.h"

ESP_ERROR_CHECK(nvs_flash_init());
ESP_ERROR_CHECK(esp_netif_init());
ESP_ERROR_CHECK(esp_event_loop_create_default());
// example_connect();   // 用 protocol_examples_common 连 Wi-Fi/以太网
```

### 2. 配置并创建客户端、注册事件

```c
#include "esp_websocket_client.h"
#include "esp_crt_bundle.h"
#include "esp_log.h"

static const char *TAG = "websocket";

static void websocket_event_handler(void *args, esp_event_base_t base,
                                    int32_t id, void *data)
{
    esp_websocket_event_data_t *evt = (esp_websocket_event_data_t *)data;
    switch (id) {
    case WEBSOCKET_EVENT_CONNECTED:
        ESP_LOGI(TAG, "CONNECTED");
        break;
    case WEBSOCKET_EVENT_DISCONNECTED:
        ESP_LOGW(TAG, "DISCONNECTED");
        break;
    case WEBSOCKET_EVENT_DATA:
        ESP_LOGI(TAG, "DATA op=%d len=%d", evt->op_code, evt->data_len);
        if (evt->op_code == 0x1)   // text
            ESP_LOGI(TAG, "text: %.*s", evt->data_len, (char *)evt->data_ptr);
        break;
    case WEBSOCKET_EVENT_ERROR:
        ESP_LOGE(TAG, "ERROR handshake=%d type=%d",
                 evt->error_handle.esp_ws_handshake_status_code,
                 evt->error_handle.error_type);
        break;
    }
}

static void websocket_app_start(void)
{
    esp_websocket_client_config_t cfg = {
        .uri = "ws://echo.websocket.events",   // 或 wss://...
        .buffer_size = 1024,
    };

#if 0   // wss：用证书包验证服务器
    cfg.uri = "wss://echo.websocket.org";
    cfg.crt_bundle_attach = esp_crt_bundle_attach;
#endif

    esp_websocket_client_handle_t client = esp_websocket_client_init(&cfg);
    esp_websocket_register_events(client, WEBSOCKET_EVENT_ANY, websocket_event_handler, (void *)client);
    esp_websocket_client_start(client);
```

### 3. 发送文本 / 二进制 / 分片帧

```c
    char data[32];
    for (int i = 0; i < 5; i++) {
        if (esp_websocket_client_is_connected(client)) {
            int len = sprintf(data, "hello %04d", i);
            esp_websocket_client_send_text(client, data, len, portMAX_DELAY);
        }
        vTaskDelay(1000 / portTICK_PERIOD_MS);
    }

    // 分片发送文本：partial → cont → fin
    memset(data, 'a', sizeof(data));
    esp_websocket_client_send_text_partial(client, data, sizeof(data), portMAX_DELAY);
    memset(data, 'b', sizeof(data));
    esp_websocket_client_send_cont_msg(client, data, sizeof(data), portMAX_DELAY);
    esp_websocket_client_send_fin(client, portMAX_DELAY);

    // 分片发送二进制：bin_partial → cont → fin
    char bin[5] = {0};
    esp_websocket_client_send_bin_partial(client, bin, sizeof(bin), portMAX_DELAY);
    esp_websocket_client_send_cont_msg(client, bin, sizeof(bin), portMAX_DELAY);
    esp_websocket_client_send_fin(client, portMAX_DELAY);
```

### 4. 安全关闭与销毁（在事件回调外）

```c
    // 必须在事件处理函数之外调用 close/stop/destroy
    esp_websocket_client_close(client, portMAX_DELAY);   // 干净关闭（发 CLOSE 帧并等服务端回显）
    esp_websocket_unregister_events(client, WEBSOCKET_EVENT_ANY, websocket_event_handler);
    esp_websocket_client_destroy(client);
```

### 5. 双向认证（wss + 客户端证书）

```c
    extern const char cacert_start[] asm("_binary_ca_cert_pem_start");
    extern const char cert_start[]   asm("_binary_client_cert_pem_start");
    extern const char cert_end[]     asm("_binary_client_cert_pem_end");
    extern const char key_start[]    asm("_binary_client_key_pem_start");
    extern const char key_end[]      asm("_binary_client_key_pem_end");

    cfg.cert_pem = cacert_start;
    cfg.client_cert = cert_start;
    cfg.client_cert_len = cert_end - cert_start;
    cfg.client_key = key_start;
    cfg.client_key_len = key_end - key_start;
```

## 常见错误

| 错误 | 原因 | 解决方法 |
|---|---|---|
| `stop/destroy/close` 调用卡死 | 在事件处理函数内调用 | 仅在回调外（定时器/任务）调用 |
| wss 连接失败 | 未配置服务器证书验证 | 设 `crt_bundle_attach = esp_crt_bundle_attach` 或 `cert_pem` |
| 发送 > buffer_size 的数据被拒 | 单帧超 buffer | 用 `send_*_partial` + `send_cont_msg` + `send_fin` 分片 |
| 自动重连过于频繁 | 默认重连间隔短 | `cfg.reconnect_timeout_ms` 调大，或 `disable_auto_reconnect=true` 自行管理 |
| ping/pong 超时断开 | 网络抖动 | 调大 `pingpong_timeout_sec` 或 `disable_pingpong_discon=true` |
| `esp_ws_handshake_status_code` 非 101 | 握手失败 | 检查 URI/path/subprotocol/headers 是否符合服务器要求 |

## 参考

- `components/esp_websocket_client/examples/target/main/websocket_example.c` — 完整 ws/wss/双向认证/分片示例
- `components/esp_websocket_client/include/esp_websocket_client.h` — 配置结构体与全部 API
- `components/esp_websocket_client/examples/target/main/Kconfig.projbuild` — `CONFIG_WEBSOCKET_URI` 等
