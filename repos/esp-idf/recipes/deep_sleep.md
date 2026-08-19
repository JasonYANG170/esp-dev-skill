# 深度睡眠与唤醒源

> **适用摘要**: 进入深度睡眠并配置定时器 / GPIO（ext0/ext1）唤醒源，启动后用 `esp_sleep_get_wakeup_causes` 判断唤醒来源（适配自 system/deep_sleep）。

> Version: ESP-IDF version used by the project.
> Evidence: `repos/esp-idf/resources/`, source/examples in `repos/esp-idf/`, and this recipe path `repos/esp-idf/recipes/deep_sleep.md`.
> Validation: draft metadata added from repository routing; verify APIs, Kconfig symbols, and component dependencies against the selected version.

## 触发意图

- "低功耗"
- "深度睡眠"
- "定时唤醒"
- "GPIO 唤醒"
- "省电"

## 前置条件

| 条件 | 要求 |
|---|---|
| 头文件 | `esp_sleep.h`、`driver/gpio.h` |
| 组件 | `esp_driver_gpio`、`esp_hw_support`（默认含） |
| 参考 | `examples/system/deep_sleep` |

## 分步说明

### 定时器唤醒（最常用）

```c
#include "esp_sleep.h"
#include "esp_log.h"

#define WAKEUP_SEC  10

void app_main(void)
{
    /* 判断唤醒来源（可选） */
    uint32_t causes = esp_sleep_get_wakeup_causes();
    if (causes & BIT(ESP_SLEEP_WAKEUP_TIMER)) {
        ESP_LOGI(TAG, "woke up by timer");
    } else if (causes == 0) {
        ESP_LOGI(TAG, "not a deep sleep reset");
    }

    /* 配置唤醒源 */
    ESP_ERROR_CHECK(esp_sleep_enable_timer_wakeup((uint64_t)WAKEUP_SEC * 1000000));

    /* 进入深度睡眠（此函数不返回） */
    esp_deep_sleep_start();
}
```

### GPIO ext0 唤醒（单 RTC GPIO，指定电平）

```c
/* 仅 RTC GPIO 可用于 ext0/ext1（如 GPIO0/2/4/12-15/25-27 等，因芯片而异） */
ESP_ERROR_CHECK(esp_sleep_enable_ext0_wakeup(GPIO_NUM_0, 0));  /* 低电平唤醒 */
```

### GPIO ext1 唤醒（多 RTC GPIO）

```c
/* 多个 RTC GPIO，掩码组合 */
uint64_t mask = BIT64(GPIO_NUM_4) | BIT64(GPIO_NUM_12);
ESP_ERROR_CHECK(esp_sleep_enable_ext1_wakeup(mask, ESP_EXT1_WAKEUP_ANY_HIGH));
```

### 上电外设断电域 GPIO 唤醒（新写法）

```c
/* GPIO 先配置为输入 + 上拉/下拉 */
gpio_config_t io = {
    .pin_bit_mask = BIT64(WAKEUP_PIN),
    .mode = GPIO_MODE_INPUT,
    .pull_up_en = GPIO_PULLUP_ENABLE,
    .intr_type = GPIO_INTR_DISABLE,
};
gpio_config(&io);
ESP_ERROR_CHECK(esp_sleep_enable_gpio_wakeup_on_hp_periph_powerdown(
        BIT64(WAKEUP_PIN), ESP_GPIO_WAKEUP_GPIO_LOW));
```

### 关键 API

```c
esp_err_t esp_sleep_enable_timer_wakeup(uint64_t time_in_us);
esp_err_t esp_sleep_enable_ext0_wakeup(gpio_num_t gpio_num, int level);
esp_err_t esp_sleep_enable_ext1_wakeup(uint64_t io_mask, esp_sleep_ext1_wakeup_mode_t level_mode);
esp_err_t esp_sleep_enable_gpio_wakeup_on_hp_periph_powerdown(uint64_t io_mask,
                                                              esp_sleep_gpio_wake_up_mode_t mode);
void      esp_deep_sleep_start(void);            /* 不返回 */
esp_err_t esp_light_sleep_start(void);           /* 浅睡，会返回 */
uint32_t  esp_sleep_get_wakeup_causes(void);
```

唤醒原因枚举（`esp_sleep_source_t`）：`ESP_SLEEP_WAKEUP_TIMER` / `ESP_SLEEP_WAKEUP_EXT0` / `ESP_SLEEP_WAKEUP_EXT1` / `ESP_SLEEP_WAKEUP_GPIO` / `ESP_SLEEP_WAKEUP_UART` / `ESP_SLEEP_WAKEUP_UNDEFINED`。

> 深度睡眠掉电后 RAM 内容丢失，醒来等价于一次复位，从 `app_main` 重新执行（区别仅是 `esp_sleep_get_wakeup_causes` 能识别来源）。

## 常见错误

| 错误 | 原因 | 解决方法 |
|---|---|---|
| ext0/ext1 不唤醒 | 用了非 RTC GPIO | 仅 RTC GPIO 可用于 ext0/ext1，查 datasheet |
| 醒来变量全丢 | 深睡掉 RAM | 重要状态写 RTC memory（`RTC_SLOW_MEM`）或 NVS |
| 电流仍大 | 外设未关 | 关 Wi-Fi/蓝牙、配置 `esp_sleep_pd_config` 掉电域 |
| `esp_deep_sleep_start` 之后还执行 | 误以为会返回 | 该函数不返回；后续代码不会执行 |
| 多 GPIO 唤醒失效 | ext1 掩码/模式错 | 用 `BIT64(pin)` 组掩码；选 `ANY_HIGH`/`ALL_LOW` |

## 参考

- `examples/system/deep_sleep` — 定时器/ext0/ext1/gpio 多种唤醒源
- `examples/system/deep_sleep/main/gpio_wakeup.c` — 上电外设断电域 GPIO 唤醒
- ESP-IDF `components/esp_hw_support/include/esp_sleep.h`
- ESP-IDF `docs/en/api-guides/deep-sleep-stub.rst` — 深睡 stub
