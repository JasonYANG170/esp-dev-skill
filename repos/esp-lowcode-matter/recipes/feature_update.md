# 特性（Feature）的接收与上报

> **适用摘要**: 实现 `feature_update_from_system()` 接收系统下发的特性更新，并用 `low_code_feature_update_to_system()` 把设备状态（按键/传感器）主动上报，支持 `feature_id` 与 matter 低层标识两种路由方式。

> Version: selected repo/component version; align with the user project and dependency manifest.
> Evidence: `repos/esp-lowcode-matter/resources/`, source/examples in `repos/esp-lowcode-matter/`, and this recipe path `repos/esp-lowcode-matter/recipes/feature_update.md`.
> Validation: draft metadata added from repository routing; verify APIs, Kconfig symbols, and component dependencies against the selected version.

## 触发意图

- "处理 matter 下发的开关/亮度"
- "上报传感器读数到 matter"
- "low_code_feature_update_to_system"
- "feature_id 怎么填"
- "endpoint cluster attribute"

## 前置条件

| 条件 | 要求 |
|---|---|
| 头文件 | `low_code.h`（`low_code_feature_data_t`、`low_code_feature_id_t`、`low_code_feature_value_type_t`） |
| 参考产品 | `products/socket`（下发 POWER）、`products/light_rgbcw_ws2812`（多 feature）、`products/temperature_sensor`（上报） |

## 分步说明

### 1. 接收系统下发：按 endpoint + feature_id 分发

取自 `products/light_rgbcw_ws2812/main/app_main.cpp`（多特性灯）：

```cpp
int feature_update_from_system(low_code_feature_data_t *data)
{
    uint16_t endpoint_id = data->details.endpoint_id;
    uint32_t feature_id  = data->details.feature_id;

    if (endpoint_id == 1) {
        if (feature_id == LOW_CODE_FEATURE_ID_POWER) {
            bool power_value = *(bool *)data->value.value;
            app_driver_set_light_state(power_value);
        } else if (feature_id == LOW_CODE_FEATURE_ID_BRIGHTNESS) {
            uint8_t brightness = *(uint8_t *)data->value.value;
            app_driver_set_light_brightness(brightness);
        } else if (feature_id == LOW_CODE_FEATURE_ID_COLOR_TEMPERATURE) {
            uint16_t color_temp = *(uint16_t *)data->value.value;
            app_driver_set_light_temperature(color_temp);
        } else if (feature_id == LOW_CODE_FEATURE_ID_HUE) {
            uint8_t hue = *(uint8_t *)data->value.value;
            app_driver_set_light_hue(hue);
        } else if (feature_id == LOW_CODE_FEATURE_ID_SATURATION) {
            uint8_t saturation = *(uint8_t *)data->value.value;
            app_driver_set_light_saturation(saturation);
        }
    }
    return 0;
}
```

插座（单通道，取自 `products/socket`）：

```cpp
int feature_update_from_system(low_code_feature_data_t *data)
{
    uint16_t endpoint_id = data->details.endpoint_id;
    uint32_t feature_id  = data->details.feature_id;
    if (endpoint_id == 1 && feature_id == LOW_CODE_FEATURE_ID_POWER) {
        bool power_value = *(bool *)data->value.value;
        return app_driver_set_socket_state(power_value);
    }
    return 0;
}
```

双通道插座（多 endpoint，取自 `products/socket_2_channel`）：

```cpp
if (endpoint_id == 1 || endpoint_id == 2) {
    if (feature_id == LOW_CODE_FEATURE_ID_POWER) {
        bool power_value = *(bool *)data->value.value;
        return app_driver_set_socket_state(endpoint_id, power_value);
    }
}
```

### 2. 上报特性：用 `feature_id`

取自 `products/template/README.md`（按键后上报电源状态）：

```cpp
bool on_off_state = true;
low_code_feature_data_t feature = {
    .details = {
        .endpoint_id = 1,
        .feature_id = LOW_CODE_FEATURE_ID_POWER
    },
    .value = {
        .type = LOW_CODE_VALUE_TYPE_BOOLEAN,
        .value_len = sizeof(bool),
        .value = (uint8_t*)&on_off_state,
    },
};
low_code_feature_update_to_system(&feature);
```

### 3. 上报特性：无 `feature_id` 映射时用 matter 低层标识

```cpp
bool on_off_state = true;
low_code_feature_data_t feature = {
    .details = {
        .endpoint_id = 1,
        .feature_id = LOW_CODE_FEATURE_ID_UNHANDLED,   /* 未映射 */
        .low_level = {
            .matter = {
                .cluster_id = 0x0006,     /* OnOff cluster */
                .attribute_id = 0x0000    /* OnOff attribute */
            }
        }
    },
    .value = {
        .type = LOW_CODE_VALUE_TYPE_BOOLEAN,
        .value_len = sizeof(bool),
        .value = (uint8_t*)&on_off_state
    },
    .priv_data = NULL
};
low_code_feature_update_to_system(&feature);
```

### 4. 数值类型（`low_code_feature_value_type_t`）

| 枚举 | 用途 |
|---|---|
| `LOW_CODE_VALUE_TYPE_BOOLEAN` | 通断等布尔 |
| `LOW_CODE_VALUE_TYPE_INTEGER` | 带符号整数（如温度 °C*100） |
| `LOW_CODE_VALUE_TYPE_UNSIGNED_INTEGER` | 无符号（如占用 0/1） |
| `LOW_CODE_VALUE_TYPE_FLOAT` | 浮点 |
| `LOW_CODE_VALUE_TYPE_STRING` / `_OCTET_STRING` / `_ARRAY` / `_CUSTOM` | 字符串/字节串/数组/自定义 |

温度上报示例（取自 `products/temperature_sensor`）：温度用 `INTEGER`，`int16_t temperature = temp * 100;`。

### 5. 常用 `low_code_feature_id_t`

见 `SKILL.md` 速查表。常用：`POWER=1001`、`BRIGHTNESS=1002`、`COLOR_TEMPERATURE=1003`、`HUE=1004`、`SATURATION=1005`、`TEMPERATURE=4004`、`COOLING_SETPOINT=4005`、`HEATING_SETPOINT=4006`、`TEMPERATURE_SENSOR_VALUE=5001`、`OCCUPANCY_SENSOR_VALUE=6001`。

## 常见错误

| 错误 | 原因 | 解决方法 |
|---|---|---|
| 下发到错设备 | 没判断 endpoint_id | 先判 endpoint 再判 feature |
| 数值解析错 | type 与数据类型不符 | 温度用 INTEGER，布尔用 BOOLEAN；不要混 |
| 系统侧收不到上报 | feature_id 与 zap 不一致或未填 matter 标识 | 核对 zap；UNHANDLED 必须填 cluster/attribute |
| value_len 错 | 长度与实际不符 | 用 `sizeof(实际类型)` |
| 指针失效 | 上报局部变量出作用域 | 在调用 `low_code_feature_update_to_system` 之前变量仍在作用域 |

## 参考

- `products/template/README.md`（feature_id 与 matter 低层两种方式）
- `products/light_rgbcw_ws2812/main/app_main.cpp`（多 feature 分发）
- `products/socket/main/app_main.cpp`、`products/socket_2_channel/main/app_main.cpp`
- `products/temperature_sensor/main/app_driver.cpp`（温度上报）
- `components/low_code/low_code.h`
