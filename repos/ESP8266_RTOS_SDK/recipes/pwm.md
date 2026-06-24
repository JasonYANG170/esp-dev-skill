# PWM 多通道输出

> **适用摘要**: 用软件 PWM 驱动（`pwm_init`）配置多通道（最多 8 路）的周期、占空比、相位，运行时调整并调用 `pwm_start` 生效，支持停机与通道反相。

## 触发意图

- "PWM 输出"
- "占空比 / 调光"
- "多路 PWM"
- "相位差 PWM"

## 前置条件

| 条件 | 要求 |
|---|---|
| 参考示例 | `examples/peripherals/pwm/` |
| 通道数 | 最多 8 通道；引脚必须是可用 GPIO（避开 GPIO6~GPIO11） |
| 周期单位 | 微秒（us），最短 20us；如 1kHz → period=1000 |

## 分步说明

### 1. 初始化多通道 + 相位（改编自示例）

```c
#include "driver/pwm.h"
#include "esp_log.h"
#include "freertos/FreeRTOS.h"
#include "freertos/task.h"

#define PWM_0_OUT_IO_NUM   12
#define PWM_1_OUT_IO_NUM   13
#define PWM_2_OUT_IO_NUM   14
#define PWM_3_OUT_IO_NUM   15

#define PWM_PERIOD    (1000)   // 1kHz

const uint32_t pin_num[4] = {PWM_0_OUT_IO_NUM, PWM_1_OUT_IO_NUM,
                             PWM_2_OUT_IO_NUM, PWM_3_OUT_IO_NUM};
uint32_t duties[4]  = {500, 500, 500, 500};   // real_duty = duties[x]/PERIOD
float    phase[4]   = {0, 0, 90.0, -90.0};    // 度，范围 (-180, 180]

void app_main()
{
    pwm_init(PWM_PERIOD, duties, 4, pin_num);
    pwm_set_phases(phase);
    pwm_start();

    int16_t count = 0;
    while (1) {
        if (count == 20) {
            pwm_stop(0x3);          // bit0/1 通道停机后输出高，bit2/3 输出低
            ESP_LOGI("pwm_example", "PWM stop");
        } else if (count == 30) {
            pwm_start();
            count = 0;
        }
        count++;
        vTaskDelay(1000 / portTICK_RATE_MS);
    }
}
```

### 2. 运行时调节

```c
// 单通道占空比（生效需 pwm_start）
pwm_set_duty(2, 200);
pwm_start();

// 一次性改全部通道
uint32_t new_duties[4] = {100, 200, 300, 400};
pwm_set_duties(new_duties);
pwm_start();

// 改周期 + 全部占空比
pwm_set_period_duties(2000, new_duties);   // 改为 500Hz
pwm_start();

// 通道反相 / 清除反相（生效需 pwm_start）
pwm_set_channel_invert(0x0F);     // 通道 0~3 反相
pwm_clear_channel_invert(0x01);   // 清除通道 0 反相
pwm_start();
```

### 3. 关键 API

| API | 作用 |
|---|---|
| `pwm_init(period, duties, channel_num, pin_num)` | 初始化（period us，最多 8 通道） |
| `pwm_deinit(void)` | 卸载 |
| `pwm_set_duty(ch, duty)` / `pwm_get_duty(ch, &duty)` | 单通道占空比 |
| `pwm_set_duties(duties)` | 全部通道占空比 |
| `pwm_set_period(period)` / `pwm_set_period_duties(period, duties)` | 周期（us） |
| `pwm_set_phase(ch, phase)` / `pwm_set_phases(phases)` / `pwm_get_phase` | 相位（度） |
| `pwm_set_channel_invert(mask)` / `pwm_clear_channel_invert(mask)` | 通道反相 |
| `pwm_start(void)` | **改任何配置后必须调用才生效** |
| `pwm_stop(stop_level_mask)` | 停机，bit=1 的通道输出高，其余低 |

> `stop_level_mask`：例如初始化 8 通道、mask=`0x0f`，则通道 0~3 停机后输出高，4~7 输出低。

## 常见错误

| 错误 | 原因 | 解决方法 |
|---|---|---|
| 改占空比输出不变 | 没调 `pwm_start()` | 任何 set 后都 `pwm_start()` |
| 占空比溢出 | duty 超过 period | duty ≤ period |
| 通道数超限 | channel_num > 8 | 最多 8 通道 |
| 引脚无波形 | 用了 GPIO6~11 | 换可用 GPIO |
| 相位无效果 | 范围错 | phase ∈ (-180, 180] |
| 频率太低/太高 | period 太大/太小 | 最小 20us；常用 1000us=1kHz |

## 参考

- `examples/peripherals/pwm/` — 官方 PWM 示例（`main/pwm_example_main.c`）
- `components/esp8266/include/driver/pwm.h` — 完整原型
- `docs/en/api-reference/peripherals/pwm.rst`
- `docs/en/api-guides/pwm-and-sniffer-coexists.rst` — PWM 与 sniffer 共存注意
