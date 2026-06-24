# GPIO LED 指示灯

> **适用摘要**: 使用 `led_indicator` 组件的 GPIO 后端创建指示灯，定义 `blink_step_t` 灯效表（双闪、三闪、慢闪、快闪），按优先级启停灯效。GPIO 后端只支持开关（LED_DUTY_1_BIT），不支持亮度调节。

## 触发意图

- "LED 指示灯"
- "led_indicator 灯效"
- "按键反馈灯"
- "blink_step 灯效表"
- "状态指示灯"

## 前置条件

| 条件 | 要求 |
|---|---|
| IDF 环境 | ESP-IDF v5.3+ |
| 组件依赖 | `espressif/led_indicator` |
| 参考示例 | `examples/indicator/gpio/main/main.c` |

## 分步说明

### 1. 添加组件依赖

```bash
idf.py add-dependency "espressif/led_indicator"
```

### 2. 定义灯效表（数组下标即优先级，0 最高）

```c
#include "led_indicator.h"
#include "led_indicator_gpio.h"

#define GPIO_LED_PIN        2
#define GPIO_ACTIVE_LEVEL   1       // 高电平点亮；低电平点亮设 0

enum {
    BLINK_DOUBLE = 0,    // 优先级 0（最高）
    BLINK_TRIPLE,        // 优先级 1
    BLINK_SLOW,          // 优先级 2
    BLINK_FAST,          // 优先级 3
    BLINK_MAX,
};

static const blink_step_t double_blink[] = {
    {LED_BLINK_HOLD, LED_STATE_ON,  500},
    {LED_BLINK_HOLD, LED_STATE_OFF, 500},
    {LED_BLINK_HOLD, LED_STATE_ON,  500},
    {LED_BLINK_HOLD, LED_STATE_OFF, 500},
    {LED_BLINK_STOP, 0, 0},
};

static const blink_step_t triple_blink[] = {
    {LED_BLINK_HOLD, LED_STATE_ON,  500},
    {LED_BLINK_HOLD, LED_STATE_OFF, 500},
    {LED_BLINK_HOLD, LED_STATE_ON,  500},
    {LED_BLINK_HOLD, LED_STATE_OFF, 500},
    {LED_BLINK_HOLD, LED_STATE_ON,  500},
    {LED_BLINK_HOLD, LED_STATE_OFF, 500},
    {LED_BLINK_STOP, 0, 0},
};

static const blink_step_t slow_blink[] = {
    {LED_BLINK_HOLD, LED_STATE_ON,  1000},
    {LED_BLINK_HOLD, LED_STATE_OFF, 1000},
    {LED_BLINK_LOOP, 0, 0},         // 循环
};

static const blink_step_t fast_blink[] = {
    {LED_BLINK_HOLD, LED_STATE_ON,  100},
    {LED_BLINK_HOLD, LED_STATE_OFF, 100},
    {LED_BLINK_LOOP, 0, 0},
};

// 注意：下标连续，BLINK_MAX 位置必须 NULL
blink_step_t const *led_mode[] = {
    [BLINK_DOUBLE] = double_blink,
    [BLINK_TRIPLE] = triple_blink,
    [BLINK_SLOW]   = slow_blink,
    [BLINK_FAST]   = fast_blink,
    [BLINK_MAX]    = NULL,
};
```

### 3. 创建指示灯并运行灯效

```c
static led_indicator_handle_t s_led = NULL;
static const char *TAG = "led_gpio";

void app_main(void)
{
    led_indicator_gpio_config_t gpio_cfg = {
        .gpio_num = GPIO_LED_PIN,
        .is_active_level_high = GPIO_ACTIVE_LEVEL,
    };
    const led_indicator_config_t cfg = {
        .blink_lists = led_mode,
        .blink_list_num = BLINK_MAX,
    };
    ESP_ERROR_CHECK(led_indicator_new_gpio_device(&cfg, &gpio_cfg, &s_led));

    // 轮播各灯效
    while (1) {
        for (int i = 0; i < BLINK_MAX; i++) {
            led_indicator_start(s_led, i);
            ESP_LOGI(TAG, "start blink: %d", i);
            vTaskDelay(pdMS_TO_TICKS(4000));
            led_indicator_stop(s_led, i);
            ESP_LOGI(TAG, "stop blink: %d", i);
            vTaskDelay(pdMS_TO_TICKS(1000));
        }
    }
}
```

### 4. 抢占式启停（高优先级临时打断）

```c
// 立即执行某灯效，直到 preempt_stop
led_indicator_preempt_start(s_led, BLINK_DOUBLE);
// ...
led_indicator_preempt_stop(s_led, BLINK_DOUBLE);
```

### 5. 开关控制

```c
led_indicator_set_on_off(s_led, true);   // 亮
led_indicator_set_on_off(s_led, false);  // 灭
```

## blink_step_type_t 动作枚举

| 动作 | 含义 |
|---|---|
| `LED_BLINK_STOP` | 结束序列 |
| `LED_BLINK_HOLD` | 保持��状态（value 为 LED_STATE_ON/OFF/亮度） |
| `LED_BLINK_BREATHE` | 呼吸过渡（需支持亮度的硬件） |
| `LED_BLINK_BRIGHTNESS` | 亮度过渡 |
| `LED_BLINK_RGB` / `LED_BLINK_HSV` | 颜色变化（RGB/Strips 后端） |
| `LED_BLINK_RGB_RING` / `LED_BLINK_HSV_RING` | 色环渐变 |
| `LED_BLINK_LOOP` | 回到序列首循环 |

> GPIO 后端只用 `LED_BLINK_HOLD` / `LED_BLINK_STOP` / `LED_BLINK_LOOP`，亮度/颜色相关动作无效。

## 常见错误

| 错误 | 原因 | 解决方法 |
|---|---|---|
| 灯效错乱 / 数组越界 | `led_mode[]` 下标不连续或末尾未 NULL | 下标连续 0..BLINK_MAX-1，`[BLINK_MAX]=NULL` |
| 灯一直亮/不亮 | `is_active_level_high` 设反 | 高电平点亮设 true，低电平点亮设 false |
| `led_indicator_start` 返回 `ESP_ERR_NOT_FOUND` | `blink_type` 超出 `blink_list_num` | 确认 i 在 0..BLINK_MAX-1 范围 |
| 调 `led_indicator_set_brightness` 无效 | GPIO 后端不支持亮度 | 改用 LEDC/RGB/Strips 后端 |

## 参考

- 组件头文件：`components/led/led_indicator/include/led_indicator.h`、`led_indicator_gpio.h`、`led_types.h`
- 真实示例：`examples/indicator/gpio/main/main.c`
- 示例 README：`examples/indicator/gpio/README.md`
