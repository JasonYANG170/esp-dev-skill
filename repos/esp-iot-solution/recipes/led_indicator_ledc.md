# LEDC PWM LED 指示灯

> **适用摘要**: 使用 `led_indicator` 组件的 LEDC 后端创建支持亮度调节的单色 LED 指示灯，实现呼吸灯、亮度过渡（25%/75%）等灯效。LEDC 后端支持 `LED_BLINK_BREATHE` / `LED_BLINK_BRIGHTNESS` 动作。

## 触发意图

- "PWM LED 调光"
- "呼吸灯"
- "led_indicator ledc"
- "LED 亮度渐变"

## 前置条件

| 条件 | 要求 |
|---|---|
| IDF 环境 | ESP-IDF v5.3+ |
| 组件依赖 | `espressif/led_indicator` |
| 参考示例 | `examples/indicator/ledc/main/main.c` |

## 分步说明

### 1. 添加组件依赖

```bash
idf.py add-dependency "espressif/led_indicator"
```

### 2. 定义含呼吸/亮度动作的灯效表

```c
#include "led_indicator.h"
#include "led_indicator_ledc.h"

#define GPIO_LED_PIN        CONFIG_EXAMPLE_GPIO_NUM
#define GPIO_ACTIVE_LEVEL   CONFIG_EXAMPLE_GPIO_ACTIVE_LEVEL
#define LEDC_LED_CHANNEL    CONFIG_EXAMPLE_LEDC_CHANNEL

enum {
    BLINK_DOUBLE = 0,
    BLINK_TRIPLE,
    BLINK_BRIGHT_75_PERCENT,
    BLINK_BRIGHT_25_PERCENT,
    BLINK_BREATHE_SLOW,
    BLINK_BREATHE_FAST,
    BLINK_MAX,
};

static const blink_step_t bright_75_percent[] = {
    {LED_BLINK_BRIGHTNESS, LED_STATE_75_PERCENT, 0},   // 渐变到 75%
    {LED_BLINK_STOP, 0, 0},
};

static const blink_step_t bright_25_percent[] = {
    {LED_BLINK_BRIGHTNESS, LED_STATE_25_PERCENT, 0},
    {LED_BLINK_STOP, 0, 0},
};

static const blink_step_t breath_slow[] = {
    {LED_BLINK_HOLD, LED_STATE_OFF, 0},
    {LED_BLINK_BREATHE, LED_STATE_ON,  1000},   // 1s 渐亮
    {LED_BLINK_BREATHE, LED_STATE_OFF, 1000},   // 1s 渐暗
    {LED_BLINK_LOOP, 0, 0},
};

static const blink_step_t breath_fast[] = {
    {LED_BLINK_HOLD, LED_STATE_OFF, 0},
    {LED_BLINK_BREATHE, LED_STATE_ON,  500},
    {LED_BLINK_BREATHE, LED_STATE_OFF, 500},
    {LED_BLINK_LOOP, 0, 0},
};

// double_blink / triple_blink 同 GPIO recipe，此处省略
extern const blink_step_t double_blink[];
extern const blink_step_t triple_blink[];

blink_step_t const *led_mode[] = {
    [BLINK_DOUBLE]              = double_blink,
    [BLINK_TRIPLE]              = triple_blink,
    [BLINK_BRIGHT_75_PERCENT]   = bright_75_percent,
    [BLINK_BRIGHT_25_PERCENT]   = bright_25_percent,
    [BLINK_BREATHE_SLOW]        = breath_slow,
    [BLINK_BREATHE_FAST]        = breath_fast,
    [BLINK_MAX]                 = NULL,
};
```

### 3. 创建 LEDC 指示灯

```c
static led_indicator_handle_t s_led = NULL;
static const char *TAG = "led_ledc";

void app_main(void)
{
    led_indicator_ledc_config_t ledc_cfg = {
        .is_active_level_high = GPIO_ACTIVE_LEVEL,
        .timer_inited = false,            // 由本组件初始化 timer
        .timer_num = LEDC_TIMER_0,
        .gpio_num = GPIO_LED_PIN,
        .channel = LEDC_LED_CHANNEL,
    };
    const led_indicator_config_t cfg = {
        .blink_lists = led_mode,
        .blink_list_num = BLINK_MAX,
    };
    ESP_ERROR_CHECK(led_indicator_new_ledc_device(&cfg, &ledc_cfg, &s_led));

    while (1) {
        for (int i = 0; i < BLINK_MAX; i++) {
            led_indicator_start(s_led, i);
            vTaskDelay(pdMS_TO_TICKS(4000));
            led_indicator_stop(s_led, i);
            vTaskDelay(pdMS_TO_TICKS(1000));
        }
    }
}
```

### 4. 运行时调亮度

```c
led_indicator_set_brightness(s_led, 128);   // 0~255
uint8_t cur = led_indicator_get_brightness(s_led);
```

### 5. 亮度状态枚举（led_types.h）

| 宏 | 值 | 含义 |
|---|---|---|
| `LED_STATE_OFF` | 0 | 灭 |
| `LED_STATE_25_PERCENT` | 64 | 25% |
| `LED_STATE_50_PERCENT` | 128 | 50% |
| `LED_STATE_75_PERCENT` | 191 | 75% |
| `LED_STATE_ON` | 255 | 最亮 |

## 常见错误

| 错误 | 原因 | 解决方法 |
|---|---|---|
| 呼吸灯不渐变 | 用了 GPIO 后端 | 呼吸/亮度动作需 LEDC/RGB/Strips 后端 |
| LEDC 通道冲突 | `channel` 被其他外设占用 | 换空闲的 `LEDC_CHANNEL_x` |
| `timer_inited=true` 但未初始化 | 自管 timer 时未先初始化 | 让组件管 timer 设 `false`，或自行初始化后置 `true` |
| 亮度过渡跳变 | `LED_BLINK_BRIGHTNESS` 的 `hold_time_ms=0` 是即变，非平滑 | 设非 0 值实现渐变时长 |

## 参考

- 组件头文件：`components/led/led_indicator/include/led_indicator_ledc.h`、`led_types.h`
- 真实示例：`examples/indicator/ledc/main/main.c`
- 示例 README：`examples/indicator/ledc/README.md`
