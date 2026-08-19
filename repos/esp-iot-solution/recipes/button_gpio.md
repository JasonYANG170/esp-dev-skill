# GPIO 按键

> **适用摘要**: 使用 `button` 组件创建 GPIO 按键、注册各类事件回调（按下、释放、单击、双击、长按、多击），并动态修改长短按阈值。button 组件 v3.x 采用 `iot_button_new_gpio_device` 工厂函数 + `button_handle_t` 句柄模式。

> Version: selected repo/component version; align with the user project and dependency manifest.
> Evidence: `repos/esp-iot-solution/resources/`, source/examples in `repos/esp-iot-solution/`, and this recipe path `repos/esp-iot-solution/recipes/button_gpio.md`.
> Validation: draft metadata added from repository routing; verify APIs, Kconfig symbols, and component dependencies against the selected version.

## 触发意图

- "创建按键"
- "按键单击/双击/长按"
- "iot_button 事件回调"
- "BOOT 按键"

## 前置条件

| 条件 | 要求 |
|---|---|
| IDF 环境 | ESP-IDF v5.3+（master）或 v4.4+（release/v2.0） |
| 组件依赖 | `espressif/button` |
| 参考示例 | `examples/get-started/button_power_save/main/main.c` |

## 分步说明

### 1. 添加组件依赖

```bash
idf.py add-dependency "espressif/button"
```

### 2. 创建 GPIO 按键并注册回调

```c
#include "iot_button.h"
#include "button_gpio.h"

#define BOOT_BUTTON_NUM     0       // 多数开发板 BOOT 在 GPIO0；C3/C2/H2/C6 为 9
#define BUTTON_ACTIVE_LEVEL 0       // BOOT 按下为低

static const char *TAG = "btn";

static void button_event_cb(void *arg, void *data)
{
    // arg 即 button_handle_t
    iot_button_print_event((button_handle_t)arg);
}

void button_init(uint32_t gpio_num)
{
    button_config_t btn_cfg = {0};     // 0 表示用 Kconfig 默认长短按时间
    button_gpio_config_t gpio_cfg = {
        .gpio_num = gpio_num,
        .active_level = BUTTON_ACTIVE_LEVEL,
        .enable_power_save = false,    // 低功耗场景见 button_power_save recipe
        .disable_pull = false,
    };

    button_handle_t btn = NULL;
    esp_err_t ret = iot_button_new_gpio_device(&btn_cfg, &gpio_cfg, &btn);
    assert(ret == ESP_OK);

    // 注意第三个参数 event_args，普通事件传 NULL
    iot_button_register_cb(btn, BUTTON_PRESS_DOWN,      NULL, button_event_cb, NULL);
    iot_button_register_cb(btn, BUTTON_PRESS_UP,        NULL, button_event_cb, NULL);
    iot_button_register_cb(btn, BUTTON_SINGLE_CLICK,    NULL, button_event_cb, NULL);
    iot_button_register_cb(btn, BUTTON_DOUBLE_CLICK,    NULL, button_event_cb, NULL);
    iot_button_register_cb(btn, BUTTON_LONG_PRESS_START,NULL, button_event_cb, NULL);
    iot_button_register_cb(btn, BUTTON_LONG_PRESS_UP,   NULL, button_event_cb, NULL);
}
```

### 3. 注册多击（N 次点击）事件

```c
button_event_args_t triple_args = { .multiple_clicks.clicks = 3 };
iot_button_register_cb(btn, BUTTON_MULTIPLE_CLICK, &triple_args, button_event_cb, NULL);
// 同一事件可注册多个不同 clicks 的回调
button_event_args_t five_args = { .multiple_clicks.clicks = 5 };
iot_button_register_cb(btn, BUTTON_MULTIPLE_CLICK, &five_args, button_event_cb, NULL);
```

### 4. 自定义长按触发时间

```c
// 全局通过 menuconfig 改：BUTTON_LONG_PRESS_TIME_MS（默认 1500）、BUTTON_SHORT_PRESS_TIME_MS（默认 180）
// 或运行时动态修改单个按键：
uint16_t lp = 3000;   // 3s 触发长按
iot_button_set_param(btn, BUTTON_LONG_PRESS_TIME_MS, &lp);
```

### 5. 轮询模式（可选，与回调可并存）

```c
button_event_t ev = iot_button_get_event(btn);
const char *s = iot_button_get_event_str(ev);   // 取事件字符串
// 或取按下持续时间（ms）
uint32_t ms = iot_button_get_pressed_time(btn);
uint8_t  repeat = iot_button_get_repeat(btn);   // 双击返回 2，三击返回 3
```

### 6. 生命周期管理

```c
iot_button_unregister_cb(btn, BUTTON_SINGLE_CLICK, NULL);
iot_button_delete(btn);
```

## 事件枚举速查（button_event_t）

| 事件 | 触发条件 |
|---|---|
| `BUTTON_PRESS_DOWN` | 按下 |
| `BUTTON_PRESS_UP` | 释放 |
| `BUTTON_PRESS_REPEAT` / `BUTTON_PRESS_REPEAT_DONE` | 重复按下 / 重复结束 |
| `BUTTON_SINGLE_CLICK` | 单击 |
| `BUTTON_DOUBLE_CLICK` | 双击 |
| `BUTTON_MULTIPLE_CLICK` | N 次点击（需配 `multiple_clicks.clicks`） |
| `BUTTON_LONG_PRESS_START` | 长按达到阈值瞬间 |
| `BUTTON_LONG_PRESS_HOLD` | 长按持续期间周期触发 |
| `BUTTON_LONG_PRESS_UP` | 长按后释放 |
| `BUTTON_PRESS_END` | 当前检测周期结束 |

## 常见错误

| 错误 | 原因 | 解决方法 |
|---|---|---|
| 编译报 `gpio_num` 不在 `button_config_t` 中 | 用了旧 API 风格 | 用 `iot_button_new_gpio_device(&btn_cfg, &gpio_cfg, &btn)`，GPIO 配置放 `button_gpio_config_t` |
| 回调不触发 | `active_level` 设反 | BOOT 按键按下为低，`active_level = 0` |
| 回调里调 `vTaskDelay` 卡死 | 回调跑在 button 扫描定时器上下文 | 回调里只做轻量操作；耗时任务用 `xTaskCreate` |
| `iot_button_register_cb` 参数顺序错 | 误把回调放第三参 | 签名是 `(handle, event, event_args*, cb, usr_data)` |
| 多击不触发 | `multiple_clicks.clicks` 未设或 < 2 | 显式设 `event_args.multiple_clicks.clicks` |

## 参考

- 组件头文件：`components/button/include/iot_button.h`、`button_gpio.h`、`button_types.h`
- 组件 Kconfig：`components/button/Kconfig`
- 真实示例：`examples/get-started/button_power_save/main/main.c`
