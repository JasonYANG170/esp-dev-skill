# GPIO 输入输出与中断

> **适用摘要**: 用 `gpio_config` 统一配置 ESP8266 GPIO 输入/输出/上下拉与中断类型，安装 per-pin ISR 服务、通过队列把中断转交任务处理。

> Version: ESP8266 RTOS SDK version used by the project.
> Evidence: `repos/ESP8266_RTOS_SDK/resources/`, source/examples in `repos/ESP8266_RTOS_SDK/`, and this recipe path `repos/ESP8266_RTOS_SDK/recipes/gpio.md`.
> Validation: draft metadata added from repository routing; verify APIs, Kconfig symbols, and component dependencies against the selected version.

## 触发意图

- "GPIO 配置"
- "按���中断"
- "输出高低电平"
- "gpio 中断"

## 前置条件

| 条件 | 要求 |
|---|---|
| 参考示例 | `examples/peripherals/gpio/` |
| 引脚约束 | 仅 GPIO0~GPIO16；禁用 GPIO6~GPIO11（接 flash）；GPIO16 无上拉 |

## 分步说明

### 1. 输出 + 输入中断（改编自示例）

```c
#include "driver/gpio.h"
#include "esp_log.h"
#include "freertos/FreeRTOS.h"
#include "freertos/task.h"
#include "freertos/queue.h"

#define GPIO_OUTPUT_IO_0    15
#define GPIO_OUTPUT_IO_1    16
#define GPIO_OUTPUT_PIN_SEL ((1ULL<<GPIO_OUTPUT_IO_0) | (1ULL<<GPIO_OUTPUT_IO_1))
#define GPIO_INPUT_IO_0     4
#define GPIO_INPUT_IO_1     5
#define GPIO_INPUT_PIN_SEL  ((1ULL<<GPIO_INPUT_IO_0) | (1ULL<<GPIO_INPUT_IO_1))

static xQueueHandle gpio_evt_queue = NULL;

static void gpio_isr_handler(void *arg)
{
    uint32_t gpio_num = (uint32_t) arg;
    xQueueSendFromISR(gpio_evt_queue, &gpio_num, NULL);   // ISR 里只入队
}

static void gpio_task_example(void *arg)
{
    uint32_t io_num;
    for (;;) {
        if (xQueueReceive(gpio_evt_queue, &io_num, portMAX_DELAY)) {
            ESP_LOGI("main", "GPIO[%d] intr, val: %d", io_num, gpio_get_level(io_num));
        }
    }
}

void app_main(void)
{
    // 输出配置
    gpio_config_t io_conf = {
        .intr_type = GPIO_INTR_DISABLE,
        .mode = GPIO_MODE_OUTPUT,
        .pin_bit_mask = GPIO_OUTPUT_PIN_SEL,
        .pull_down_en = 0,
        .pull_up_en = 0,
    };
    gpio_config(&io_conf);

    // 输入配置（上升沿中断 + 上拉）
    io_conf.intr_type = GPIO_INTR_POSEDGE;
    io_conf.mode = GPIO_MODE_INPUT;
    io_conf.pin_bit_mask = GPIO_INPUT_PIN_SEL;
    io_conf.pull_up_en = 1;
    gpio_config(&io_conf);

    // 单独改某脚中断类型
    gpio_set_intr_type(GPIO_INPUT_IO_0, GPIO_INTR_ANYEDGE);

    gpio_evt_queue = xQueueCreate(10, sizeof(uint32_t));
    xTaskCreate(gpio_task_example, "gpio_task", 2048, NULL, 10, NULL);

    // 安装 per-pin ISR 服务（与 gpio_isr_register 二选一）
    gpio_install_isr_service(0);
    gpio_isr_handler_add(GPIO_INPUT_IO_0, gpio_isr_handler, (void *)GPIO_INPUT_IO_0);
    gpio_isr_handler_add(GPIO_INPUT_IO_1, gpio_isr_handler, (void *)GPIO_INPUT_IO_1);

    int cnt = 0;
    while (1) {
        ESP_LOGI("main", "cnt: %d", cnt++);
        vTaskDelay(1000 / portTICK_RATE_MS);
        gpio_set_level(GPIO_OUTPUT_IO_0, cnt % 2);
        gpio_set_level(GPIO_OUTPUT_IO_1, cnt % 2);
    }
}
```

### 2. 关键 API / 类型

| API / 类型 | 作用 |
|---|---|
| `gpio_config(const gpio_config_t *)` | 统一配置（mask/mode/pull/intr） |
| `gpio_config_t` 字段：`pin_bit_mask`、`mode`、`pull_up_en`、`pull_down_en`、`intr_type` | 配置结构 |
| `gpio_num_t` | `GPIO_NUM_0`~`GPIO_NUM_16` |
| `gpio_mode_t` | `GPIO_MODE_DISABLE/INPUT/OUTPUT/OUTPUT_OD` |
| `gpio_int_type_t` | `GPIO_INTR_DISABLE/POSEDGE/NEGEDGE/ANYEDGE/LOW_LEVEL/HIGH_LEVEL` |
| `gpio_set_level(gpio_num, level)` | 输出电平 |
| `gpio_get_level(gpio_num)` | 读输入电平 |
| `gpio_set_direction / set_pull_mode / pullup_en / pulldown_en` | 运行时改方向/上下拉 |
| `gpio_install_isr_service(0)` + `gpio_isr_handler_add(num, cb, arg)` | per-pin ISR（推荐） |
| `gpio_isr_register(fn, arg, 0, NULL)` | 全局单一 ISR（与 isr_service 互斥） |

## 常见错误

| 错误 | 原因 | 解决方法 |
|---|---|---|
| 中断不触发 | 没装 `gpio_install_isr_service` 或没 `gpio_isr_handler_add` | 两者都要调用 |
| 引脚无反应 | 用了 GPIO6~GPIO11 | 换其它脚 |
| GPIO16 上拉无效 | GPIO16 无内部上拉（仅下拉） | 外部上拉电阻 |
| ISR 里做长操作 | 在中断里 printf/malloc | ISR 只入队，处理放任务 |
| `gpio_config` 返回 `ESP_ERR_INVALID_ARG` | `pin_bit_mask` 含不支持位 | 仅置目标脚位 |
| 与 `gpio_isr_register` 冲突 | 同时用了 isr_service 与全局 ISR | 二选一 |

## 参考

- `examples/peripherals/gpio/` — 官方 GPIO 示例（`main/user_main.c`）
- `components/esp8266/include/driver/gpio.h` — 完整原型与枚举
- `docs/en/api-reference/peripherals/gpio.rst`
