# UART 通信

> **适用摘要**: 配置 UART 端口、安装驱动、阻塞/事件驱动收发，含引脚设置。

## 触发意图

- "配置 UART"
- "串口收发"
- "UART 中断接收"
- "uart_read_bytes"
- "GPS/蓝牙模块串口"

## 前置条件

| ��件 | 要求 |
|---|---|
| 组件 | `esp_driver_uart` |
| 头文件 | `driver/uart.h` |
| 参考 | `examples/peripherals/uart/uart_echo`、`uart_async_rxtxtasks` |

## 分步说明

### 初始化并阻塞收发（115200, 8N1）

```c
#include "driver/uart.h"

#define ECHO_UART_PORT   UART_NUM_1
#define ECHO_UART_BAUD   115200
#define ECHO_TX_PIN      17
#define ECHO_RX_PIN      18
#define BUF_SIZE         1024

void app_main(void)
{
    uart_config_t cfg = {
        .baud_rate  = ECHO_UART_BAUD,
        .data_bits  = UART_DATA_8_BITS,
        .parity     = UART_PARITY_DISABLE,
        .stop_bits  = UART_STOP_BITS_1,
        .flow_ctrl  = UART_HW_FLOWCTRL_DISABLE,
        .source_clk = UART_SCLK_DEFAULT,
    };
    /* 安装驱动 */
    ESP_ERROR_CHECK(uart_driver_install(ECHO_UART_PORT, BUF_SIZE, BUF_SIZE, 0, NULL, 0));
    ESP_ERROR_CHECK(uart_param_config(ECHO_UART_PORT, &cfg));
    ESP_ERROR_CHECK(uart_set_pin(ECHO_UART_PORT, ECHO_TX_PIN, ECHO_RX_PIN,
                                 UART_PIN_NO_CHANGE, UART_PIN_NO_CHANGE));

    uint8_t *data = (uint8_t *)malloc(BUF_SIZE);
    while (1) {
        int len = uart_read_bytes(ECHO_UART_PORT, data, BUF_SIZE, pdMS_TO_TICKS(100));
        if (len > 0) {
            uart_write_bytes(ECHO_UART_PORT, (const char *)data, len);
        }
    }
}
```

### 事件驱动接收（推荐用于带队列的并发）

```c
#include "freertos/queue.h"

static QueueHandle_t uart_queue;

static void uart_event_task(void *pvParameters)
{
    uart_event_t event;
    uint8_t *data = (uint8_t *)malloc(BUF_SIZE);
    while (1) {
        if (xQueueReceive(uart_queue, (void *)&event, portMAX_DELAY)) {
            if (event.type == UART_DATA) {
                int len = uart_read_bytes(ECHO_UART_PORT, data, event.size, portMAX_DELAY);
                uart_write_bytes(ECHO_UART_PORT, (const char *)data, len);
            } else if (event.type == UART_FIFO_OVF) {
                uart_flush_input(ECHO_UART_PORT);
            }
        }
    }
}

/* 安装时传入队列 */
uart_driver_install(ECHO_UART_PORT, BUF_SIZE * 2, BUF_SIZE * 2, 20, &uart_queue, 0);
xTaskCreate(uart_event_task, "uart_event", 3072, NULL, 12, NULL);
```

### 关键 API

```c
esp_err_t uart_driver_install(uart_port_t uart_num, int rx_buffer_size,
                              int tx_buffer_size, int queue_size,
                              QueueHandle_t *uart_queue, int intr_alloc_flags);
esp_err_t uart_param_config(uart_port_t uart_num, const uart_config_t *uart_config);
esp_err_t uart_set_pin(uart_port_t uart_num, int tx_io_num, int rx_io_num,
                       int rts_io_num, int cts_io_num);
int       uart_read_bytes(uart_port_t uart_num, uint8_t *data, uint32_t length, uint32_t ticks_to_wait);
int       uart_write_bytes(uart_port_t uart_num, const void *src, size_t size);
esp_err_t uart_wait_tx_done(uart_port_t uart_num, uint32_t ticks_to_wait);
esp_err_t uart_flush_input(uart_port_t uart_num);
esp_err_t uart_set_baudrate(uart_port_t uart_num, uint32_t baudrate);
```

端口枚举：`UART_NUM_0` / `UART_NUM_1` / `UART_NUM_2`（具体数量随芯片）。
> `UART_NUM_0` 常作日志/控制台；应用串口建议用 `UART_NUM_1`/`UART_NUM_2`。

## 常见错误

| 错误 | 原因 | 解决方法 |
|---|---|---|
| `uart_read_bytes` 始终返回 0 | 未 `uart_driver_install` | 先 install 再 config/pin |
| 接收乱码 | 波特率/线序不一致 | 两端波特率、8N1 一致；确认 TX↔RX 交叉 |
| 收到不完整包 | 阻塞超时太短 | 增大 `ticks_to_wait` 或改事件驱动 |
| 数据丢 | FIFO 溢出 | 加大 RX buffer、提高任务优先级或用事件模式 flush |
| 引脚不工作 | 用了被占用引脚 | 查芯片可用 GPIO，避开 Flash/USB 引脚 |

## 参考

- `examples/peripherals/uart/uart_echo` — 回环收发
- `examples/peripherals/uart/uart_async_rxtxtasks` — 异步收发任务
- ESP-IDF `components/esp_driver_uart/include/driver/uart.h`
