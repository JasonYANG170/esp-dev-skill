# ESP-NOW 入门：广播发送与回调接收

> **适用摘要**: 在 ESP-IDF 工程中集成 ESP-NOW 组件，完成 storage/Wi-Fi/espnow 初始化，实现广播发送用户数据并通过回调接收（参考 `examples/get-started`）。

## 触发意图

- "ESP-NOW 收发数据"
- "espnow 怎么初始化"
- "广播发送"
- "入门示例"
- "UART 透传到 ESP-NOW"

## 前置条件

| 条件 | 要求 |
|---|---|
| ESP-IDF | >= v4.4，已 `idf.py set-target <chip>` |
| 依赖 | `idf.py add-dependency "espressif/esp-now=*"` |
| 参考示例 | `examples/get-started/main/app_main.c` |

## 分步说明

### 1. 初始化 storage 与 Wi-Fi（STA + PS_NONE）

```c
#include "esp_wifi.h"
#include "espnow.h"
#include "espnow_storage.h"
#include "espnow_utils.h"

static void app_wifi_init(void)
{
    esp_event_loop_create_default();
    wifi_init_config_t cfg = WIFI_INIT_CONFIG_DEFAULT();
    ESP_ERROR_CHECK(esp_wifi_init(&cfg));
    ESP_ERROR_CHECK(esp_wifi_set_mode(WIFI_MODE_STA));
    ESP_ERROR_CHECK(esp_wifi_set_storage(WIFI_STORAGE_RAM));
    ESP_ERROR_CHECK(esp_wifi_set_ps(WIFI_PS_NONE));
    ESP_ERROR_CHECK(esp_wifi_start());
}
```

### 2. 注册接收回调

```c
static esp_err_t app_recv_cb(uint8_t *src_addr, void *data,
                             size_t size, wifi_pkt_rx_ctrl_t *rx_ctrl)
{
    ESP_PARAM_CHECK(src_addr);
    ESP_PARAM_CHECK(data);
    ESP_PARAM_CHECK(size);
    ESP_PARAM_CHECK(rx_ctrl);

    ESP_LOGI(TAG, "espnow_recv [" MACSTR "][ch %d][rssi %d][%u]: %.*s",
             MAC2STR(src_addr), rx_ctrl->channel, rx_ctrl->rssi,
             size, size, (char *)data);
    return ESP_OK;
}
```

### 3. 初始化 ESP-NOW 并开启接收

```c
void app_main(void)
{
    espnow_storage_init();
    app_wifi_init();

    espnow_config_t espnow_config = ESPNOW_INIT_CONFIG_DEFAULT();
    espnow_init(&espnow_config);

    espnow_set_config_for_data_type(ESPNOW_DATA_TYPE_DATA, true, app_recv_cb);
}
```

### 4. 广播发送（带重传与广播标志）

```c
espnow_frame_head_t frame_head = {
    .retransmit_count = 10,   // 默认 ESPNOW_FRAME_CONFIG_DEFAULT 也是 10
    .broadcast        = true,
};

uint8_t *data = ESP_CALLOC(1, ESPNOW_DATA_LEN);  // size 不得超过 ESPNOW_DATA_LEN
size_t size = ...;  // 例如从 UART 读取

esp_err_t ret = espnow_send(ESPNOW_DATA_TYPE_DATA, ESPNOW_ADDR_BROADCAST,
                            data, size, &frame_head, portMAX_DELAY);
ESP_ERROR_CONTINUE(ret != ESP_OK, "<%s> espnow_send", esp_err_to_name(ret));
```

> 说明：`frame_config` 传 `NULL` 时使用 `ESPNOW_FRAME_CONFIG_DEFAULT()`（broadcast=true, retransmit_count=10）。`ESPNOW_ADDR_BROADCAST` 是组件预定义的广播地址。

### 5. 完整 UART → ESP-NOW 透传任务（参考 get-started 示例）

```c
static void app_uart_read_task(void *arg)
{
    uint8_t *data = ESP_CALLOC(1, ESPNOW_DATA_LEN);
    espnow_frame_head_t frame_head = { .retransmit_count = 10, .broadcast = true };

    for (;;) {
        size_t size = uart_read_bytes(UART_PORT_NUM, data, ESPNOW_DATA_LEN, pdMS_TO_TICKS(10));
        ESP_ERROR_CONTINUE(size <= 0, "");

        esp_err_t ret = espnow_send(ESPNOW_DATA_TYPE_DATA, ESPNOW_ADDR_BROADCAST,
                                    data, size, &frame_head, portMAX_DELAY);
        ESP_ERROR_CONTINUE(ret != ESP_OK, "<%s> espnow_send", esp_err_to_name(ret));
        memset(data, 0, ESPNOW_DATA_LEN);
    }
    ESP_FREE(data);
    vTaskDelete(NULL);
}
```

## 常见错误

| 错误 | 原因 | 解决方法 |
|---|---|---|
| 收不到数据 | 未调用 `espnow_set_config_for_data_type(type, true, cb)` | 显式 enable 并注册回调 |
| `espnow_init` 返回 ESP_FAIL | Wi-Fi 未 `esp_wifi_start` | 先 `app_wifi_init()` 再 `espnow_init` |
| 发送超时 | `wait_ticks` 过小或队列满 | 用 `portMAX_DELAY` 或调大 `qsize`（默认 32） |
| 数据被截断 | `size > ESPNOW_DATA_LEN(230)` | 分包发送，单包 ≤ 230 字节 |
| 两台设备收不到彼此 | 不在同一 Wi-Fi 信道或未 set-target | 确认两端 chip、信道一致；STA 模式默认信道 |

## 参考

- `examples/get-started/main/app_main.c` — UART 与 ESP-NOW 透传完整示例
- `resources/api_reference.md` — `espnow_init` / `espnow_send` / `espnow_set_config_for_data_type` 签名
