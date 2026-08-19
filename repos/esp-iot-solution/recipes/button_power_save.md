# 按键低功耗与 Light Sleep

> **适用摘要**: 使用 `button` 组件的 power-save 模式，配合 ESP-IDF PM 框架与 light sleep，实现按键唤醒。创建设备时 `enable_power_save=true`，并注册 `iot_button_register_power_save_cb`，在所有按键空闲时进入 light sleep。

> Evidence: `repos/esp-iot-solution/resources/`, source/examples in `repos/esp-iot-solution/`, and this recipe path `repos/esp-iot-solution/recipes/button_power_save.md`.
> Validation: draft metadata added from repository routing; verify APIs, Kconfig symbols, and component dependencies against the selected version.

## 触发意图

- "按键低功耗"
- "按键唤醒 light sleep"
- "button power save"
- "降低按键待机功耗"

## 前置条件

| 条件 | 要求 |
|---|---|
| IDF 环境 | ESP-IDF v5.3+ |
| 组件依赖 | `espressif/button` |
| menuconfig | 启用 `CONFIG_FREERTOS_USE_TICKLESS_IDLE`、`CONFIG_PM_ENABLE` |
| 参考示例 | `examples/get-started/button_power_save/` |

## 分步说明

### 1. 添加依赖并启用 PM

```bash
idf.py add-dependency "espressif/button"
# menuconfig:
#   Component config → FreeRTOS → Tickless Idle → Enable
#   Component config → Power Management → Support for power management
```

### 2. 初始化 PM 框架

```c
#include "esp_pm.h"
#include "esp_sleep.h"

void power_save_init(void)
{
    esp_pm_config_t pm_config = {
        .max_freq_mhz = 160,
        .min_freq_mhz = 80,
#if CONFIG_FREERTOS_USE_TICKLESS_IDLE
        .light_sleep_enable = true
#endif
    };
    ESP_ERROR_CHECK(esp_pm_configure(&pm_config));
}
```

### 3. 创建带 power-save 的按键

```c
#include "iot_button.h"
#include "button_gpio.h"

#define BOOT_BUTTON_NUM      0
#define BUTTON_ACTIVE_LEVEL  0

static void button_event_cb(void *arg, void *data)
{
    iot_button_print_event((button_handle_t)arg);
    esp_sleep_wakeup_cause_t cause = esp_sleep_get_wakeup_cause();
    if (cause != ESP_SLEEP_WAKEUP_UNDEFINED) {
        ESP_LOGI(TAG, "Wake up from light sleep, reason %d", cause);
    }
}

void button_init(uint32_t gpio_num)
{
    button_config_t btn_cfg = {0};
    button_gpio_config_t gpio_cfg = {
        .gpio_num = gpio_num,
        .active_level = BUTTON_ACTIVE_LEVEL,
        .enable_power_save = true,        // 关键：注册 GPIO 唤醒源
    };

    button_handle_t btn = NULL;
    ESP_ERROR_CHECK(iot_button_new_gpio_device(&btn_cfg, &gpio_cfg, &btn));

    iot_button_register_cb(btn, BUTTON_PRESS_DOWN,       NULL, button_event_cb, NULL);
    iot_button_register_cb(btn, BUTTON_SINGLE_CLICK,     NULL, button_event_cb, NULL);
    iot_button_register_cb(btn, BUTTON_LONG_PRESS_START, NULL, button_event_cb, NULL);

    // 注册低功耗回调：当所有按键空闲时被调用
    button_power_save_config_t ps_cfg = {
        .enter_power_save_cb = button_enter_power_save,
    };
    ESP_ERROR_CHECK(iot_button_register_power_save_cb(&ps_cfg));
}

// 进入睡眠的实际动作
void button_enter_power_save(void *usr_data)
{
    ESP_LOGI(TAG, "Can enter power save now");
    esp_light_sleep_start();
}
```

### 4. app_main 串联

```c
void app_main(void)
{
    power_save_init();
    button_init(BOOT_BUTTON_NUM);
    // button 组件后台跑自己的扫描定时器；app_main 退出后系统进 tickless / light sleep
}
```

### 5. 预期日志

```
I pm: Frequency switching config: CPU_MAX: 160, APB_MAX: 80, APB_MIN: 80, Light sleep: ENABLED
I button: IoT Button Version: 3.2.0
I button_power_save: Button event BUTTON_PRESS_DOWN
I button_power_save: Wake up from light sleep, reason 4
```

## 常见错误

| 错误 | 原因 | 解决方法 |
|---|---|---|
| 创建设备返回 `ESP_ERR_INVALID_STATE` | GPIO 不支持作为唤醒源 | 查芯片 datasheet，选用支持 light sleep 唤醒的 GPIO |
| 唤醒后丢事件 | tickless 未启用 / `BUTTON_PERIOD_TIME_MS` 过大 | menuconfig 启用 tickless；调小扫描周期 |
| `iot_button_register_power_save_cb` 返回 `ESP_ERR_INVALID_STATE` | 尚未创建任何按键 | 先 `iot_button_new_gpio_device` 再注册 power-save 回调 |
| 功耗未下降 | `light_sleep_enable` 未启用或外部电路漏电 | 确认 `pm_config.light_sleep_enable=true`；排查外设电源 |

## 参考

- 组件头文件：`components/button/include/iot_button.h`（`button_power_save_config_t`、`iot_button_register_power_save_cb`）
- 真实示例：`examples/get-started/button_power_save/main/main.c`
- 示例 README：`examples/get-started/button_power_save/README.md`
