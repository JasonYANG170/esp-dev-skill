# 按键组件（button_driver）

> **适用摘要**: 用 `button_driver_create` 创建按键，`button_driver_register_cb` 注册单击/长按回调，实现单击切换设备状态、长按触发工厂复位。

> Version: selected repo/component version; align with the user project and dependency manifest.
> Evidence: `repos/esp-lowcode-matter/resources/`, source/examples in `repos/esp-lowcode-matter/`, and this recipe path `repos/esp-lowcode-matter/recipes/button_driver.md`.
> Validation: draft metadata added from repository routing; verify APIs, Kconfig symbols, and component dependencies against the selected version.

## 触发意图

- "加一个按键"
- "按键单击/长按"
- "button_driver"
- "按键触发工厂复位"
- "LP GPIO / HP GPIO 按键"

## 前置条件

| 条件 | 要求 |
|---|---|
| 头文件 | `button_driver.h` |
| 组件依赖 | `main/CMakeLists.txt` REQUIRES 含 `button` |
| Kconfig | `CONFIG_BUTTON_DRIVER_USE_HP_GPIO=y`（默认）或 `_LP_GPIO`（见 `components/button/Kconfig`） |
| 参考产品 | `products/socket`、`products/temperature_sensor`、`products/occupancy_sensor` |

## 分步说明

### 1. 事件与配置结构（`button_driver.h`）

```c
typedef enum {
    BUTTON_PRESS_DOWN = 0,
    BUTTON_PRESS_UP,
    BUTTON_SINGLE_CLICK,
    BUTTON_LONG_PRESS_START,
    BUTTON_LONG_PRESS_UP,
    BUTTON_EVENT_MAX,
} button_event_t;

typedef struct {
    uint16_t long_press_time;   /* 0=默认 BUTTON_LONG_PRESS_TIME_MS */
    uint16_t short_press_time;  /* 0=默认 BUTTON_SHORT_PRESS_TIME_MS */
    int      gpio_num;
    uint8_t  pullup_en:1;
    uint8_t  pulldown_en:1;
    uint8_t  active_level:1;    /* 按下时的电平 */
} button_config_t;

typedef void (*button_cb_t)(void *button_handle, void *usr_data);
typedef void *button_handle_t;
```

### 2. API

```c
button_handle_t button_driver_create(const button_config_t *config);
int button_driver_delete(button_handle_t btn_handle);
int button_driver_register_cb(button_handle_t btn, button_event_t event, button_cb_t cb, void *usr_data);
int button_driver_unregister_cb(button_handle_t btn, button_event_t event);
```

### 3. 完整示例（取自 `products/socket/main/app_driver.cpp`）

GPIO9、上拉、按下为低电平；单击切换插座状态并上报 `POWER`，长按抬起触发工厂复位。

```cpp
#include <button_driver.h>
#include <low_code.h>

#define BUTTON_GPIO_NUM ((gpio_num_t)9)

static bool socket_state = false;

static void app_driver_toggle_socket_state_button_callback(void *arg, void *data)
{
    socket_state = !socket_state;
    app_driver_set_socket_state(socket_state);   /* 驱动层 */

    /* 上报特性到系统 */
    low_code_feature_data_t update_data = {
        .details = { .endpoint_id = 1, .feature_id = LOW_CODE_FEATURE_ID_POWER },
        .value = {
            .type = LOW_CODE_VALUE_TYPE_BOOLEAN,
            .value_len = sizeof(bool),
            .value = (uint8_t*)&socket_state,
        },
    };
    low_code_feature_update_to_system(&update_data);
}

static void app_driver_trigger_factory_reset_button_callback(void *arg, void *data)
{
    low_code_event_t event = { .event_type = LOW_CODE_EVENT_FACTORY_RESET };
    low_code_event_to_system(&event);
}

/* 在 app_driver_init 中： */
button_config_t btn_cfg = {
    .gpio_num = BUTTON_GPIO_NUM,
    .pullup_en = 1,
    .active_level = 0,
};
button_handle_t btn_handle = button_driver_create(&btn_cfg);
if (!btn_handle) {
    printf("%s: Failed to create the button\n", TAG);
    return -1;
}

button_driver_register_cb(btn_handle, BUTTON_SINGLE_CLICK,   app_driver_toggle_socket_state_button_callback, NULL);
button_driver_register_cb(btn_handle, BUTTON_LONG_PRESS_UP,  app_driver_trigger_factory_reset_button_callback, NULL);
```

### 4. Kconfig 选项（`components/button/Kconfig`）

| 选项 | 默认 | 说明 |
|---|---|---|
| `BUTTON_DRIVER_USE_HP_GPIO` | y | 用 HP GPIO 作按键输入 |
| `HP_BUTTON_LOOP_INTERVAL` | 50 | HP GPIO 轮询间隔(ms) |
| `BUTTON_DRIVER_USE_LP_GPIO` | n | 用 LP GPIO 作按键输入 |
| `MAX_BUTTON_NUM` | 4 | 最大按键数 |

在 product 的 `sdkconfig.defaults` 设置，例如 `products/socket/sdkconfig.defaults`：`CONFIG_BUTTON_DRIVER_USE_HP_GPIO=y`。

## 常见错误

| 错误 | 原因 | 解决方法 |
|---|---|---|
| 按键无反应 | 未 enable 软件中断 / Kconfig 选错 | HP GPIO 需软件中断机制；确认 Kconfig �� HP/LP 与硬件一致 |
| `button_driver_create` 返回 NULL | 超过 `MAX_BUTTON_NUM` 或参数错 | 调大 Kconfig；检查 gpio_num/active_level |
| 长按误触发 | 用了 `PRESS_DOWN` 而非长按事件 | 复位用 `BUTTON_LONG_PRESS_UP` |
| 上拉没生效 | `pullup_en` 字段写法错 | 用位域赋值 `.pullup_en = 1` |
| 回调里崩溃 | 在回调里做阻塞/动态内存 | 回调内只改状态并上报，重活放主循环 |

## 参考

- `components/button/button_driver.h`、`components/button/Kconfig`
- `products/socket/main/app_driver.cpp`
- `products/temperature_sensor/main/app_driver.cpp`
- `recipes/event_handling.md`（工厂复位上报）
