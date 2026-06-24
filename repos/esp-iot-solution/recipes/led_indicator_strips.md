# RGB LED 与 WS2812 灯带（HSV/RGB 彩色灯效）

> **适用摘要**: 使用 `led_indicator` 的 RGB 后端（三通道 PWM LED）与 Strips 后端（WS2812 等 RMT/SPI 寻址灯带）做彩色灯效：`LED_BLINK_RGB` / `LED_BLINK_HSV` 设颜色，`LED_BLINK_RGB_RING` / `LED_BLINK_HSV_RING` 做颜色渐变，`SET_IHSV` / `SET_IRGB` / `INSERT_INDEX` 控制灯带单颗或全部灯。颜色宏 `SET_RGB(r,g,b)`、`SET_HSV(h,s,v)` 在 `led_convert.h`。

## 触发意图

- "RGB LED 彩色灯效"
- "WS2812 灯带"
- "led_indicator strips / rgb"
- "LED_BLINK_HSV / LED_BLINK_RGB_RING"
- "颜色渐变 / flowing"
- "SET_IHSV / MAX_INDEX"

## 前置条件

| 条件 | 要求 |
|---|---|
| 目标芯片 | ESP32 全系（RGB 需 3 路 LEDC；Strips 需 RMT 或 SPI） |
| IDF 环境 | ESP-IDF v5.3+；Strips 需 `espressif/led_strip` 组件 |
| 组件依赖 | `espressif/led_indicator` |
| 硬件 | RGB LED（共阳/共阴三引脚）或 WS2812 灯带（单数据线接 GPIO） |
| 参考示例 | `examples/indicator/rgb`、`examples/indicator/ws2812_strips` |

## 分步说明

### 1. 添加组件依赖

```bash
idf.py add-dependency "espressif/led_indicator"
```

### 2. 颜色宏与最大值（led_convert.h）

```c
#include "led_indicator.h"
#include "led_indicator_rgb.h"      // RGB 后端
#include "led_indicator_strips.h"    // Strips 后端
#include "led_convert.h"            // SET_RGB / SET_HSV / SET_IHSV

// MAX_HUE=360  MAX_SATURATION=255  MAX_BRIGHTNESS=255  MAX_INDEX=127
// SET_RGB(r,g,b)         -> 24 位 RGB
// SET_HSV(h,s,v)         -> h 0..360, s/v 0..255
// SET_IRGB(index,r,g,b)  -> index 0..126；127=MAX_INDEX 表示全部灯
// SET_IHSV(index,h,s,v)  -> 同上 HSV
// INSERT_INDEX(index,brightness) -> 给 BRIGHTNESS/BREATHE 加 index
```

### 3. 定义彩色灯效表（HSV / RGB / 渐变）

`LED_BLINK_RGB` / `LED_BLINK_HSV` 立即设色（`hold_time_ms` 表示停留时长）；`LED_BLINK_RGB_RING` / `LED_BLINK_HSV_RING` 从当前色平滑过渡到新色（`hold_time_ms` = 渐变时长）。

```c
enum {
    BLINK_DOUBLE_RED = 0,
    BLINK_BLUE_BREATH,
    BLINK_COLOR_HSV_RING,
    BLINK_COLOR_RGB_RING,
    BLINK_MAX,
};

// 红色双闪：先设红，再 HOLD ON/OFF
static const blink_step_t double_red_blink[] = {
    {LED_BLINK_RGB, SET_RGB(255, 0, 0), 0},   // 设红色（即时）
    {LED_BLINK_HOLD, LED_STATE_ON,  500},
    {LED_BLINK_HOLD, LED_STATE_OFF, 500},
    {LED_BLINK_HOLD, LED_STATE_ON,  500},
    {LED_BLINK_HOLD, LED_STATE_OFF, 500},
    {LED_BLINK_STOP, 0, 0},
};

// 蓝色呼吸：HSV 设色 + BREATHE 改亮度
static const blink_step_t breath_blue_blink[] = {
    {LED_BLINK_HSV, SET_HSV(240, MAX_SATURATION, 0), 0},  // 蓝，亮度 0
    {LED_BLINK_BREATHE, LED_STATE_ON,  1000},             // 渐亮
    {LED_BLINK_BREATHE, LED_STATE_OFF, 1000},             // 渐暗
    {LED_BLINK_LOOP, 0, 0},
};

// HSV 环渐变：红 <-> 蓝
static const blink_step_t color_hsv_ring_blink[] = {
    {LED_BLINK_HSV,      SET_HSV(0,   MAX_SATURATION, MAX_BRIGHTNESS), 0},
    {LED_BLINK_HSV_RING, SET_HSV(240, MAX_SATURATION, 127), 2000},    // 2s 红->蓝
    {LED_BLINK_HSV_RING, SET_HSV(0,   MAX_SATURATION, MAX_BRIGHTNESS), 2000},
    {LED_BLINK_LOOP, 0, 0},
};

// RGB 环渐变：绿 -> 品红 -> 绿
static const blink_step_t color_rgb_ring_blink[] = {
    {LED_BLINK_RGB,      SET_RGB(0, 255, 0),   0},
    {LED_BLINK_RGB_RING, SET_RGB(255, 0, 255), 2000},
    {LED_BLINK_RGB_RING, SET_RGB(0, 255, 0),   2000},
    {LED_BLINK_LOOP, 0, 0},
};

blink_step_t const *led_mode[] = {
    [BLINK_DOUBLE_RED]    = double_red_blink,
    [BLINK_BLUE_BREATH]   = breath_blue_blink,
    [BLINK_COLOR_HSV_RING] = color_hsv_ring_blink,
    [BLINK_COLOR_RGB_RING] = color_rgb_ring_blink,
    [BLINK_MAX]           = NULL,
};
```

### 4. RGB 后端（三引脚 PWM LED）

`led_indicator_rgb_config_t` 三个 GPIO + 三个 LEDC channel + 一个 timer；`timer_inited=false` 让组件自管 timer。

```c
static led_indicator_handle_t s_led = NULL;

void app_main(void)
{
    led_indicator_rgb_config_t rgb_cfg = {
        .is_active_level_high = 1,            // 共阳=1，共阴=0
        .timer_inited  = false,
        .timer_num     = LEDC_TIMER_0,
        .red_gpio_num   = CONFIG_EXAMPLE_GPIO_RED_NUM,
        .green_gpio_num = CONFIG_EXAMPLE_GPIO_GREEN_NUM,
        .blue_gpio_num  = CONFIG_EXAMPLE_GPIO_BLUE_NUM,
        .red_channel    = CONFIG_EXAMPLE_LEDC_RED_CHANNEL,
        .green_channel  = CONFIG_EXAMPLE_LEDC_GREEN_CHANNEL,
        .blue_channel   = CONFIG_EXAMPLE_LEDC_BLUE_CHANNEL,
    };
    const led_indicator_config_t cfg = {
        .blink_lists = led_mode,
        .blink_list_num = BLINK_MAX,
    };
    ESP_ERROR_CHECK(led_indicator_new_rgb_device(&cfg, &rgb_cfg, &s_led));

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

### 5. WS2812 Strips 后端（寻址灯带）

`led_indicator_strips_config_t` 内嵌 `led_strip_config_t`（来自 `led_strip` 组件）+ 驱动选 `LED_STRIP_RMT` 或 `LED_STRIP_SPI`。

```c
#define WS2812_GPIO_NUM    CONFIG_EXAMPLE_WS2812_GPIO_NUM
#define WS2812_STRIPS_NUM  CONFIG_EXAMPLE_WS2812_STRIPS_NUM
#define LED_STRIP_RMT_RES_HZ  (10 * 1000 * 1000)

void app_main(void)
{
    led_strip_config_t strip_config = {
        .strip_gpio_num = WS2812_GPIO_NUM,
        .max_leds       = WS2812_STRIPS_NUM,
        .color_component_format = LED_STRIP_COLOR_COMPONENT_FMT_GRB,  // WS2812 常见 GRB
        .led_model      = LED_MODEL_WS2812,
        .flags.invert_out = false,
    };
    led_strip_rmt_config_t rmt_config = {
        .clk_src       = RMT_CLK_SRC_DEFAULT,
        .resolution_hz = LED_STRIP_RMT_RES_HZ,
        .flags.with_dma = false,                 // ESP32-S3 可开 DMA
    };
    led_indicator_strips_config_t strips_config = {
        .led_strip_cfg     = strip_config,
        .led_strip_driver  = LED_STRIP_RMT,
        .led_strip_rmt_cfg = rmt_config,
    };
    const led_indicator_config_t cfg = {
        .blink_lists = led_mode,
        .blink_list_num = BLINK_MAX,
    };
    ESP_ERROR_CHECK(led_indicator_new_strips_device(&cfg, &strips_config, &s_led));
    // 同上循环 start/stop
}
```

### 6. 灯带 Index 控制（单颗 / 全部 / 流水）

`SET_IHSV(MAX_INDEX, ...)` 操作所有灯；`SET_IHSV(i, ...)` 操作第 i 颗。`INSERT_INDEX(MAX_INDEX, brightness)` 给 BREATHE 加"全部灯"语义。

```c
// 流水灯：全部灯沿色环渐变（仅当 STRIPS_NUM > 1 才有意义）
static const blink_step_t flowing_blink[] = {
    {LED_BLINK_HSV,      SET_IHSV(MAX_INDEX, 0,       MAX_SATURATION, MAX_BRIGHTNESS), 0},
    {LED_BLINK_HSV_RING, SET_IHSV(MAX_INDEX, MAX_HUE, MAX_SATURATION, MAX_BRIGHTNESS), 2000},
    {LED_BLINK_LOOP, 0, 0},
};

// 三颗灯分别红/绿/蓝
static const blink_step_t index_rgb[] = {
    {LED_BLINK_RGB, SET_IRGB(0, 255, 0,   0), 0},
    {LED_BLINK_RGB, SET_IRGB(1, 0,   255, 0), 0},
    {LED_BLINK_RGB, SET_IRGB(2, 0,   0,   255), 0},
    {LED_BLINK_LOOP, 0, 0},
};

// 全部灯一起呼吸
static const blink_step_t all_breath[] = {
    {LED_BLINK_BRIGHTNESS, INSERT_INDEX(MAX_INDEX, LED_STATE_OFF), 0},
    {LED_BLINK_BREATHE,    INSERT_INDEX(MAX_INDEX, LED_STATE_ON),  1000},
    {LED_BLINK_BREATHE,    INSERT_INDEX(MAX_INDEX, LED_STATE_OFF), 1000},
    {LED_BLINK_LOOP, 0, 0},
};
```

### 7. 运行时改色 / Gamma 校正

```c
led_indicator_set_hsv(s_led, SET_HSV(120, 255, 128));   // 直接设 HSV（不经 blink 表）
led_indicator_set_rgb(s_led, SET_RGB(0xFF, 0x80, 0x00));

led_indicator_new_gamma_table(2.3);   // 重建 gamma 表（默认 2.3），改善暗部细节
```

## 支持矩阵（来自 docs/en/display/led_indicator.rst）

| 后端 | 开关 | 亮度 | 呼吸 | 颜色 | 渐变 | Index |
|---|:---:|:---:|:---:|:---:|:---:|:---:|
| GPIO | √ | × | × | × | × | × |
| LEDC | √ | √ | √ | × | × | × |
| RGB | √ | √ | √ | √ | √ | × |
| Strips | √ | √ | √ | √ | √ | √ |

## 常见错误

| 错误 | 原因 | 解决方法 |
|---|---|---|
| 颜色灯效无效 | 用了 GPIO 后端 | 颜色/渐变须 RGB 或 Strips 后端，GPIO 只有开/关 |
| WS2812 颜色错位 | pixel format 不对 | `LED_STRIP_COLOR_COMPONENT_FMT_GRB`（多数 WS2812）或 RGB/RGBW |
| RGB 共阳/共阴反相 | `is_active_level_high` 不对 | 共阳设 `true`，共阴设 `false` |
| 三路 LEDC channel 冲突 | channel 被其它外设占用 | 换空闲 `LEDC_CHANNEL_x`，三路各占一个 |
| Strips 编译报缺 led_strip | 未加 `led_strip` 依赖 | `idf.py add-dependency "espressif/led_strip"`（通常被 led_indicator 自动拉） |
| 流水灯只亮一颗 | `STRIPS_NUM` 配成 1 | Kconfig 把灯数设为实际颗数；流水灯效要 `>1` |
| 渐变不平滑 | `hold_time_ms` 太短或 gamma 未建 | 渐变时长 ≥ 数百 ms；`led_indicator_new_gamma_table(2.3)` |
| `SET_HSV` h>360 | 没用 `MAX_HUE` 常量 | H 范围 0..360，用 `MAX_HUE` 防越界（宏内部会 clamp） |
| Index 127 不全亮 | 用了非 Strips 后端 | Index 仅 Strips 支持；RGB/GPIO 无 index 概念 |

## 参考项目

- 组件头文件：`components/led/led_indicator/include/led_indicator_rgb.h`、`led_indicator_strips.h`、`led_convert.h`、`led_types.h`
- RGB 示例：`examples/indicator/rgb/main/main.c`
- WS2812 Strips 示例：`examples/indicator/ws2812_strips/main/main.c`
- 在线文档：`docs/en/display/led_indicator.rst`（"Controlling Color" 与 "Controlling Index" 小节）
