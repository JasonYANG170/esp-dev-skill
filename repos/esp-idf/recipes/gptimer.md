# GPTimer 通用定时器

> **适用摘要**: 用新 HAL 的 GPTimer（`gptimer_new_timer`）创建高分辨率定时器、配置告警回调，实现周期任务（适配自 gptimer 示例）。

## 触发意图

- "定时器中断"
- "周期性任务"
- "gptimer"
- "每秒触发一次"
- "测量脉宽 / 周期"

## 前置条件

| 条件 | 要求 |
|---|---|
| 组件 | `esp_driver_gptimer` |
| 头文件 | `driver/gptimer.h` |
| 参考 | `examples/peripherals/timer_group/gptimer` |

## 分步说明

### 创建 1MHz（1µs/tick）向上计数定时器

```c
#include "driver/gptimer.h"
#include "freertos/queue.h"

static QueueHandle_t s_queue;

typedef struct { uint64_t event_count; } evt_t;

static bool IRAM_ATTR on_alarm_cb(gptimer_handle_t timer,
                                  const gptimer_alarm_event_data_t *edata,
                                  void *user_data)
{
    BaseType_t hpw = pdFALSE;
    evt_t ele = { .event_count = edata->count_value };
    xQueueSendFromISR(s_queue, &ele, &hpw);
    return (hpw == pdTRUE);
}

void app_main(void)
{
    s_queue = xQueueCreate(10, sizeof(evt_t));

    gptimer_handle_t gptimer = NULL;
    gptimer_config_t timer_config = {
        .clk_src       = GPTIMER_CLK_SRC_DEFAULT,
        .direction     = GPTIMER_COUNT_UP,
        .resolution_hz = 1000000,   /* 1 MHz => 1 tick = 1 µs */
    };
    ESP_ERROR_CHECK(gptimer_new_timer(&timer_config, &gptimer));

    /* 注��回调 */
    gptimer_event_callbacks_t cbs = { .on_alarm = on_alarm_cb };
    ESP_ERROR_CHECK(gptimer_register_event_callbacks(gptimer, &cbs, s_queue));

    /* 周期告警：1s 自动重装 */
    gptimer_alarm_config_t alarm = {
        .reload_count = 0,
        .alarm_count  = 1000000,                       /* 1s */
        .flags.auto_reload_on_alarm = true,
    };
    ESP_ERROR_CHECK(gptimer_set_alarm_action(gptimer, &alarm));

    ESP_ERROR_CHECK(gptimer_enable(gptimer));
    ESP_ERROR_CHECK(gptimer_start(gptimer));

    evt_t ele;
    while (1) {
        if (xQueueReceive(s_queue, &ele, portMAX_DELAY)) {
            ESP_LOGI(TAG, "tick count=%llu", ele.event_count);
        }
    }
}
```

### 读取/设置计数值

```c
uint64_t count;
gptimer_get_raw_count(gptimer, &count);
gptimer_set_raw_count(gptimer, 0);
```

### 关键 API

```c
esp_err_t gptimer_new_timer(const gptimer_config_t *config, gptimer_handle_t *ret_timer);
esp_err_t gptimer_set_alarm_action(gptimer_handle_t timer, const gptimer_alarm_config_t *config);
esp_err_t gptimer_register_event_callbacks(gptimer_handle_t timer,
                                           const gptimer_event_callbacks_t *cbs, void *user_data);
esp_err_t gptimer_enable(gptimer_handle_t timer);
esp_err_t gptimer_disable(gptimer_handle_t timer);
esp_err_t gptimer_start(gptimer_handle_t timer);
esp_err_t gptimer_stop(gptimer_handle_t timer);
esp_err_t gptimer_get_raw_count(gptimer_handle_t timer, uint64_t *value);
esp_err_t gptimer_set_raw_count(gptimer_handle_t timer, uint64_t value);
```

> 回调运行在 ISR 上下文：须 `IRAM_ATTR`，仅用 `...FromISR`，且回调返回 `true` 表示需要 yield。

## 常见错误

| 错误 | 原因 | 解决方法 |
|---|---|---|
| 改回调/告警无效 | 定时器未 disable | 改配置前 `gptimer_disable`，改完 `enable`/`start` |
| 回调内崩溃 | 调了阻塞 API | ISR 内仅用 `...FromISR` + `portYIELD_FROM_ISR` |
| 频率不准 | `resolution_hz` 与 `alarm_count` 不匹配 | 1MHz 时 `alarm_count=1000000` 即 1s |
| 一次触发后停止 | `auto_reload_on_alarm=false` | 周期任务设 `true` |
| 多次注册回调 | 重复调用 | 一个 timer 注册一次；如需更换先 disable |

## 参考

- `examples/peripherals/timer_group/gptimer` — 周期告警、auto-reload、读改计数值
- `examples/peripherals/timer_group/gptimer_capture_hc_sr04` — 捕获模式测距
- ESP-IDF `components/esp_driver_gptimer/include/driver/gptimer.h`
