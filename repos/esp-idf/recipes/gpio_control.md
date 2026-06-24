# GPIO 输入输出与中断

> **适用摘要**: 配置 GPIO 为输入/输出，读取/设置电平，安装 ISR 服务并处理下降沿等中断。

## 触发意图

- "配置 GPIO"
- "GPIO 中断"
- "按键检测"
- "点亮 LED"
- "gpio_isr_handler_add"

## 前置条件

| 条件 | 要求 |
|---|---|
| 组件 | `esp_driver_gpio` |
| 头文件 | `driver/gpio.h` |
| 参考 | `examples/peripherals/gpio/generic_gpio`、`examples/get-started/blink` |

## 分步说明

### 基本输出（点灯）

```c
#include "driver/gpio.h"

#define LED_GPIO  2

void app_main(void)
{
    gpio_reset_pin(LED_GPIO);                       /* 复位为默认 GPIO 功能 */
    gpio_set_direction(LED_GPIO, GPIO_MODE_OUTPUT);

    while (1) {
        gpio_set_level(LED_GPIO, 1);
        vTaskDelay(pdMS_TO_TICKS(500));
        gpio_set_level(LED_GPIO, 0);
        vTaskDelay(pdMS_TO_TICKS(500));
    }
}
```

### 输入（带上拉）

```c
gpio_config_t io_conf = {
    .pin_bit_mask = (1ULL << BUTTON_GPIO),
    .mode = GPIO_MODE_INPUT,
    .pull_up_en = GPIO_PULLUP_ENABLE,
    .pull_down_en = GPIO_PULLDOWN_DISABLE,
    .intr_type = GPIO_INTR_DISABLE,
};
gpio_config(&io_conf);

int level = gpio_get_level(BUTTON_GPIO);
```

### 中断（边沿触发 + ISR 服务）

```c
#include "freertos/queue.h"

static QueueHandle_t gpio_evt_queue = NULL;

static void IRAM_ATTR gpio_isr_handler(void *arg)
{
    uint32_t gpio_num = (uint32_t)arg;
    BaseType_t hpw = pdFALSE;
    xQueueSendFromISR(gpio_evt_queue, &gpio_num, &hpw);
    portYIELD_FROM_ISR(hpw);
}

static void gpio_task(void *arg)
{
    uint32_t io_num;
    while (1) {
        if (xQueueReceive(gpio_evt_queue, &io_num, portMAX_DELAY)) {
            ESP_LOGI(TAG, "GPIO[%" PRIu32 "] intr, val=%d", io_num, gpio_get_level(io_num));
        }
    }
}

void app_main(void)
{
    gpio_config_t io_conf = {
        .pin_bit_mask = (1ULL << BUTTON_GPIO),
        .mode = GPIO_MODE_INPUT,
        .pull_up_en = GPIO_PULLUP_ENABLE,
        .intr_type = GPIO_INTR_NEGEDGE,            /* 下降沿 */
    };
    gpio_config(&io_conf);

    gpio_evt_queue = xQueueCreate(10, sizeof(uint32_t));
    xTaskCreate(gpio_task, "gpio_task", 2048, NULL, 10, NULL);

    gpio_install_isr_service(0);                   /* 安装 GPIO ISR 服务（全局一次） */
    gpio_isr_handler_add(BUTTON_GPIO, gpio_isr_handler, (void *)BUTTON_GPIO);
}
```

### 关键 API

```c
esp_err_t gpio_config(const gpio_config_t *pGPIOConfig);
esp_err_t gpio_reset_pin(gpio_num_t gpio_num);
esp_err_t gpio_set_direction(gpio_num_t gpio_num, gpio_mode_t mode);
esp_err_t gpio_set_pull_mode(gpio_num_t gpio_num, gpio_pull_mode_t pull);
int       gpio_get_level(gpio_num_t gpio_num);
esp_err_t gpio_set_level(gpio_num_t gpio_num, uint32_t level);
esp_err_t gpio_set_intr_type(gpio_num_t gpio_num, gpio_int_type_t intr_type);
esp_err_t gpio_install_isr_service(int intr_alloc_flags);
esp_err_t gpio_isr_handler_add(gpio_num_t gpio_num, gpio_isr_t isr_handler, void *args);
esp_err_t gpio_isr_handler_remove(gpio_num_t gpio_num);
```

`gpio_int_type_t` 取值：`GPIO_INTR_DISABLE` / `GPIO_INTR_POSEDGE`（上升沿） / `GPIO_INTR_NEGEDGE`（下降沿） / `GPIO_INTR_ANYEDGE`（双边沿） / `GPIO_INTR_LOW_LEVEL` / `GPIO_INTR_HIGH_LEVEL`。

## 常见错误

| 错误 | 原因 | 解决方法 |
|---|---|---|
| `gpio_isr_handler_add` 返回 `ESP_ERR_INVALID_STATE` | 未安装 ISR 服务 | 先 `gpio_install_isr_service(0)` |
| 中断不触发 | 引脚未设 intr_type 或未上拉 | 配置 `intr_type`，按键加 `GPIO_PULLUP_ENABLE` |
| ISR 内崩溃 | 用了阻塞 API 或非 IRAM 函数 | ISR 标 `IRAM_ATTR`，仅用 `...FromISR` |
| 引脚无反应 | 用了 Flash/strapping 引脚 | 避开 GPIO6-11（Flash）、strapping 引脚，查 datasheet |
| `gpio_set_level` 无效 | 模式不是 OUTPUT | 先 `gpio_set_direction(GPIO_MODE_OUTPUT)` |

## 参考

- `examples/peripherals/gpio/generic_gpio` — 通用 GPIO 收发与中断
- `examples/get-started/blink` — GPIO/LED Strip 输出
- ESP-IDF `components/esp_driver_gpio/include/driver/gpio.h`
