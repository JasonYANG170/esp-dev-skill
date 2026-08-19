# 继电器组件与智能插座（单/双通道）

> **适用摘要**: 用 `relay_driver_init` / `relay_driver_set_power` 控制继电器，实现单通道（`products/socket`）与多 endpoint 双通道（`products/socket_2_channel`）智能插座。

> Version: selected repo/component version; align with the user project and dependency manifest.
> Evidence: `repos/esp-lowcode-matter/resources/`, source/examples in `repos/esp-lowcode-matter/`, and this recipe path `repos/esp-lowcode-matter/recipes/relay_socket.md`.
> Validation: draft metadata added from repository routing; verify APIs, Kconfig symbols, and component dependencies against the selected version.

## 触发意图

- "智能插座"
- "继电器控制"
- "relay_driver"
- "多通道插座"
- "通断控制"

## 前置条件

| 条件 | 要求 |
|---|---|
| 头文件 | `relay_driver.h` |
| 组件依赖 | REQUIRES 含 `relay`（插座通常还含 `button`、`light` 作指示） |
| 参考产品 | `products/socket`（单通道）、`products/socket_2_channel`（双通道） |

## 分步说明

### 1. API（`relay_driver.h`）

```c
void relay_driver_init(int gpio_num);
void relay_driver_set_power(int gpio_num, bool power);   /* true=合, false=断 */
```

### 2. 单通道插座（取自 `products/socket/main/app_driver.cpp`）

GPIO2 继电器、GPIO9 按键、GPIO8 WS2812 指示。

```cpp
#include <relay_driver.h>
#include <button_driver.h>
#include <light_driver.h>
#include <low_code.h>

#define BUTTON_GPIO_NUM    ((gpio_num_t)9)
#define RELAY_GPIO_NUM     ((gpio_num_t)2)
#define INDICATOR_GPIO_NUM ((gpio_num_t)8)

static bool socket_state = false;

int app_driver_set_socket_state(bool state)
{
    socket_state = state;
    relay_driver_set_power(RELAY_GPIO_NUM, state);
    light_driver_set_power(state);   /* 指示灯跟随 */
    return 0;
}

int app_driver_init()
{
    relay_driver_init(RELAY_GPIO_NUM);

    /* 按键：单击切换 + 上报 */
    button_config_t btn_cfg = { .gpio_num = BUTTON_GPIO_NUM, .pullup_en = 1, .active_level = 0 };
    button_handle_t btn = button_driver_create(&btn_cfg);
    button_driver_register_cb(btn, BUTTON_SINGLE_CLICK, [](void *a, void *d){
        socket_state = !socket_state;
        app_driver_set_socket_state(socket_state);
        low_code_feature_data_t upd = {
            .details = { .endpoint_id = 1, .feature_id = LOW_CODE_FEATURE_ID_POWER },
            .value = { .type = LOW_CODE_VALUE_TYPE_BOOLEAN, .value_len = sizeof(bool), .value = (uint8_t*)&socket_state },
        };
        low_code_feature_update_to_system(&upd);
    }, NULL);
    /* 长按复位（见 recipes/event_handling.md） */

    /* 指示灯 */
    light_driver_config_t cfg = {
        .device_type = LIGHT_DEVICE_TYPE_WS2812,
        .channel_comb = LIGHT_CHANNEL_COMB_3CH_RGB,
        .io_conf = { .ws2812_io = { .ctrl_io = INDICATOR_GPIO_NUM } },
        .min_brightness = 0, .max_brightness = 100,
    };
    light_driver_init(&cfg);
    light_driver_set_power(socket_state);
    return 0;
}
```

`feature_update_from_system`（接收 App 下发开关）：

```cpp
int feature_update_from_system(low_code_feature_data_t *data)
{
    if (data->details.endpoint_id == 1 && data->details.feature_id == LOW_CODE_FEATURE_ID_POWER) {
        bool power_value = *(bool *)data->value.value;
        return app_driver_set_socket_state(power_value);
    }
    return 0;
}
```

### 3. 双通道插座（多 endpoint，取自 `products/socket_2_channel/main/app_main.cpp`）

每个通道一个 endpoint（1 和 2），`feature_update_from_system` 按 endpoint 分发到对应继电器：

```cpp
int feature_update_from_system(low_code_feature_data_t *data)
{
    uint16_t endpoint_id = data->details.endpoint_id;
    uint32_t feature_id  = data->details.feature_id;

    if (endpoint_id == 1 || endpoint_id == 2) {
        if (feature_id == LOW_CODE_FEATURE_ID_POWER) {
            bool power_value = *(bool *)data->value.value;
            return app_driver_set_socket_state(endpoint_id, power_value);
        }
    }
    return 0;
}
```

`app_driver_set_socket_state(endpoint_id, state)` 内部按 endpoint 选不同 GPIO 的继电器。对应 `data_model_wifi.zap` 需为每个 endpoint 配置 OnOff cluster。

## 常见错误

| 错误 | 原因 | 解决方法 |
|---|---|---|
| 继电器不动 | 未 `relay_driver_init(gpio)` | 初始化时传入继电器 GPIO |
| 双通道控制错位 | 没按 endpoint 选 GPIO | `set_socket_state(endpoint_id,...)` 内按 endpoint 映射 GPIO |
| App 下发无响应 | feature_update 未判 endpoint/feature | 加 `endpoint_id==N && feature_id==POWER` 判断 |
| 状态不同步 | 按键切换后没上报 | 按键回调里调 `low_code_feature_update_to_system` |

## 参考

- `components/relay/relay_driver.h`
- `products/socket/main/app_driver.cpp`、`products/socket/main/app_main.cpp`
- `products/socket_2_channel/main/app_main.cpp`
- `recipes/button_driver.md`、`recipes/event_handling.md`
