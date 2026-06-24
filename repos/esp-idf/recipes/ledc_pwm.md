# LEDC PWM 输出与调光

> **适用摘要**: 用 LEDC 外设输出 PWM、改变频率与占空比，含基础初始化与运行时调光（适配自 ledc_basic）。

## 触发意图

- "输出 PWM"
- "LEDC 调光"
- "驱动 LED / 舵机"
- "ledc_set_duty"
- "改变占空比"

## 前置条件

| 条件 | 要求 |
|---|---|
| 组件 | `esp_driver_ledc` |
| 头文件 | `driver/ledc.h` |
| 参考 | `examples/peripherals/ledc/ledc_basic`、`ledc_fade` |

## 分步说明

### 配置定时器与通道（4 kHz, 13-bit 占空比）

```c
#include "driver/ledc.h"

#define LEDC_TIMER          LEDC_TIMER_0
#define LEDC_MODE           LEDC_LOW_SPEED_MODE
#define LEDC_CHANNEL        LEDC_CHANNEL_0
#define LEDC_DUTY_RES       LEDC_TIMER_13_BIT   /* 0..8191 */
#define LEDC_FREQUENCY      4000                /* Hz */
#define LEDC_OUTPUT_IO      4

static void example_ledc_init(void)
{
    ledc_timer_config_t ledc_timer = {
        .speed_mode      = LEDC_MODE,
        .duty_resolution = LEDC_DUTY_RES,
        .timer_num       = LEDC_TIMER,
        .freq_hz         = LEDC_FREQUENCY,
        .clk_cfg         = LEDC_AUTO_CLK,
    };
    ESP_ERROR_CHECK(ledc_timer_config(&ledc_timer));

    ledc_channel_config_t ledc_channel = {
        .speed_mode = LEDC_MODE,
        .channel    = LEDC_CHANNEL,
        .timer_sel  = LEDC_TIMER,
        .gpio_num   = LEDC_OUTPUT_IO,
        .duty       = 0,
        .hpoint     = 0,
    };
    ESP_ERROR_CHECK(ledc_channel_config(&ledc_channel));
}
```

### 设置占空比（必须 update）

```c
/* duty 范围 0 .. (2^LEDC_DUTY_RES - 1) */
ESP_ERROR_CHECK(ledc_set_duty(LEDC_MODE, LEDC_CHANNEL, 4096));  /* 约 50% */
ESP_ERROR_CHECK(ledc_update_duty(LEDC_MODE, LEDC_CHANNEL));     /* 生效 */
```

### 硬件渐变（fade，需开 `CONFIG_LEDC_CTRL_FUNC_IN_RDMA=y` 或对应 Kconfig）

```c
ledc_set_fade_with_time(LEDC_MODE, LEDC_CHANNEL, 8000, 1000); /* 1s 渐到 8000 */
ledc_fade_start(LEDC_MODE, LEDC_CHANNEL, LEDC_FADE_WAIT_DONE);
```

### 关键 API

```c
esp_err_t ledc_timer_config(const ledc_timer_config_t *timer_conf);
esp_err_t ledc_channel_config(const ledc_channel_config_t *ledc_conf);
esp_err_t ledc_set_duty(ledc_mode_t speed_mode, ledc_channel_t channel, uint32_t duty);
esp_err_t ledc_update_duty(ledc_mode_t speed_mode, ledc_channel_t channel);
esp_err_t ledc_set_freq(ledc_mode_t speed_mode, ledc_timer_t timer_num, uint32_t freq_hz);
esp_err_t ledc_set_fade_with_time(ledc_mode_t speed_mode, ledc_channel_t channel,
                                  uint32_t target_duty, int desired_fade_time_ms);
uint32_t  ledc_get_duty(ledc_mode_t speed_mode, ledc_channel_t channel);
```

`speed_mode`：`LEDC_LOW_SPEED_MODE`（全芯片可用）/ `LEDC_HIGH_SPEED_MODE`（仅 esp32）。

> 占空比分辨率与频率成反比：分辨率越高，可达最低频率越高但步进越细，需按 datasheet 取舍。

## 常见错误

| 错误 | 原因 | 解决方法 |
|---|---|---|
| 占空比不变化 | 未 `ledc_update_duty` | `ledc_set_duty` 后必须调 `ledc_update_duty` |
| 100% 占空比达不到 | 用了最大分辨率 | 降低 1 位分辨率（如 13-bit 设 12-bit） |
| 频率偏差大 | `clk_cfg` 选择不当 | 用 `LEDC_AUTO_CLK` 让驱动自选时钟 |
| 高速模式报错 | 非 esp32 用了 HIGH_SPEED | 改 `LEDC_LOW_SPEED_MODE` |
| 引脚无波形 | GPIO 被占用 / mode 不对 | 复位引脚 `gpio_reset_pin`，避开 Flash 引脚 |

## 参考

- `examples/peripherals/ledc/ledc_basic` — 基础 PWM 输出与调光
- `examples/peripherals/ledc/ledc_fade` — 硬件渐变
- ESP-IDF `components/esp_driver_ledc/include/driver/ledc.h`
