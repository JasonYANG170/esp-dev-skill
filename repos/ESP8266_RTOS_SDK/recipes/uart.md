# UART 收发与事件队列

> **适用摘要**: 用 UART 驱动（`uart_param_config` + `uart_driver_install`）配置串口，通过事件队列处理接收/溢出/错误，用 `uart_write_bytes` / `uart_read_bytes` 收发数据。

## 触发意图

- "串口收发"
- "UART 驱动"
- "uart 事件"
- "串口接收"

## 前置条件

| 条件 | 要求 |
|---|---|
| 参考示例 | `examples/peripherals/uart_events/`、`examples/peripherals/uart_echo/`、`examples/peripherals/uart_select/` |
| 引脚 | UART0 默认 GPIO1(TX)/GPIO3(RX)；UART1 默认 GPIO2(TX)/GPIO8(RX)，可用 `uart_enable_swap()` 交换 |
| 可用端口 | 仅 `UART_NUM_0` 与 `UART_NUM_1`（`UART_NUM_MAX`） |

## 分步说明

### 1. 配置 + 安装驱动（改编自 uart_events 示例）

```c
#include "driver/uart.h"
#include "esp_log.h"
#include "freertos/FreeRTOS.h"
#include "freertos/task.h"
#include "freertos/queue.h"

#define EX_UART_NUM UART_NUM_0
#define BUF_SIZE    (1024)
#define RD_BUF_SIZE (BUF_SIZE)

static QueueHandle_t uart0_queue;

void app_main()
{
    uart_config_t uart_config = {
        .baud_rate  = 74880,
        .data_bits  = UART_DATA_8_BITS,
        .parity     = UART_PARITY_DISABLE,
        .stop_bits  = UART_STOP_BITS_1,
        .flow_ctrl  = UART_HW_FLOWCTRL_DISABLE,
    };
    uart_param_config(EX_UART_NUM, &uart_config);

    // 安装驱动并拿到事件队列
    uart_driver_install(EX_UART_NUM, BUF_SIZE * 2, BUF_SIZE * 2, 100, &uart0_queue, 0);

    xTaskCreate(uart_event_task, "uart_event_task", 2048, NULL, 12, NULL);
}
```

### 2. 事件处理任务

```c
static void uart_event_task(void *pvParameters)
{
    uart_event_t event;
    uint8_t *dtmp = (uint8_t *) malloc(RD_BUF_SIZE);

    for (;;) {
        if (xQueueReceive(uart0_queue, (void *)&event, portMAX_DELAY)) {
            switch (event.type) {
                case UART_DATA:                                     // 收到数据
                    uart_read_bytes(EX_UART_NUM, dtmp, event.size, portMAX_DELAY);
                    uart_write_bytes(EX_UART_NUM, (const char *)dtmp, event.size); // 回显
                    break;
                case UART_FIFO_OVF:                                 // 硬件 FIFO 溢出
                    uart_flush_input(EX_UART_NUM);
                    xQueueReset(uart0_queue);
                    break;
                case UART_BUFFER_FULL:                              // 环形缓冲满
                    uart_flush_input(EX_UART_NUM);
                    xQueueReset(uart0_queue);
                    break;
                case UART_PARITY_ERR:
                case UART_FRAME_ERR:
                default:
                    break;
            }
        }
    }
    free(dtmp);
    vTaskDelete(NULL);
}
```

### 3. 关键 API / 类型

| API / 类型 | 作用 |
|---|---|
| `uart_param_config(uart_num, &uart_config)` | 设波特率/数据位/校验/停止位/流控 |
| `uart_driver_install(uart_num, rx_buf, tx_buf, queue_size, &queue, 0)` | 安装驱动，`tx_buf=0` 则发送阻塞到发完 |
| `uart_write_bytes(uart_num, src, size)` | 阻塞发送（写入 TX 环形/直接 FIFO） |
| `uart_tx_chars(uart_num, buf, len)` | 不等待发送（仅 TX 缓冲未启用时用） |
| `uart_read_bytes(uart_num, buf, len, ticks)` | 读取 |
| `uart_flush(uart_num)` / `uart_flush_input(uart_num)` | 丢弃 RX 缓冲 |
| `uart_wait_tx_done(uart_num, ticks)` | 等最后一字节发出 |
| `uart_set_baudrate / set_word_length / set_stop_bits / set_parity` | 运行时改参数 |
| `uart_set_rx_timeout(uart_num, tout_thresh)` | RX 超时阈值（symbol 周期，0 禁用，max 126） |
| `uart_enable_swap()` / `uart_disable_swap()` | UART0 引脚交换（MTCK/MTDO） |
| `uart_event_type_t` | `UART_DATA / BUFFER_FULL / FIFO_OVF / FRAME_ERR / PARITY_ERR` |
| `uart_port_t` | `UART_NUM_0` / `UART_NUM_1` |

### 4. FIFO 容量提示

`UART_FIFO_LEN = 128`（硬件 FIFO）；驱动 RX 环形缓冲 `rx_buffer_size` 必须 > `UART_FIFO_LEN`；`tx_buffer_size` 为 0 或 > `UART_FIFO_LEN`。

## 常见错误

| 错误 | 原因 | 解决方法 |
|---|---|---|
| 收不到数据 | 没装驱动 / 引脚未连 | 先 `uart_driver_install` 再 read |
| 频繁 `UART_FIFO_OVF` | 处理太慢 / 无流控 | 增大 RX 缓冲、加流控、加快读取 |
| 乱码 | 波特率不一致 | 两端统一（boot ROM 日志 74880） |
| `uart_write_bytes` 阻塞太久 | `tx_buffer_size=0` 且数据多 | 设非 0 TX 缓冲，或分批发 |
| UART1 无输出 | 引脚与 flash 冲突 | UART1 仅 GPIO2 可用，注意 boot 约束 |
| 数据丢失 | 读慢于写 | 用事件队列 + 足够大环形缓冲 |

## 参考

- `examples/peripherals/uart_events/` — 事件驱动收发（`main/uart_events_example_main.c`）
- `examples/peripherals/uart_echo/` — 简单回显
- `examples/peripherals/uart_select/` — select 风格
- `components/esp8266/include/driver/uart.h` — 完整原型
