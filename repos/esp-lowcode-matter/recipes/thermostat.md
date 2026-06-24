# 温控器产品（单 endpoint 多特性：温度 + 制冷/制热设定点）

> **适用摘要**: 基于 `products/thermostat` 实现温控器（MA-thermostat，device_type_id 769）——在**单个 endpoint** 上同时处理三个 feature_id：温度（`LOW_CODE_FEATURE_ID_TEMPERATURE`）、制冷设定点（`LOW_CODE_FEATURE_ID_COOLING_SETPOINT`）、制热设定点（`LOW_CODE_FEATURE_ID_HEATING_SETPOINT`），均使用带符号 `int16_t`（°C×100）。

## 触发意图

- "温控器"
- "thermostat"
- "制冷/制热设定点"
- "单 endpoint 多 feature"
- "MA-thermostat"
- "cooling setpoint / heating setpoint"

## 前置条件

| 条件 | 要求 |
|---|---|
| 头文件 | `low_code.h`（`low_code_feature_data_t`、`LOW_CODE_FEATURE_ID_TEMPERATURE/COOLING_SETPOINT/HEATING_SETPOINT`、`LOW_CODE_VALUE_TYPE_INTEGER`） |
| 组件依赖 | REQUIRES 含 `low_code system`（如加指示灯/按键再补 `light`、`button`） |
| 数据模型 | `configuration/data_model_wifi.zap`（或 `data_model_thread.zap`）中 endpoint 1 的 endpointType `deviceTypeRef.code = 769`（MA-thermostat），且启用 Thermostat cluster（code 513） |
| 参考产品 | `products/thermostat`（骨架，带占位驱动；含 README、app_main.cpp、app_driver.cpp、app_priv.h、configuration/*） |

## 分步说明

### 1. 数据模型：单 endpoint = MA-thermostat

温控器与多通道插座（`recipes/relay_socket.md`）的模式相反：插座是**多 endpoint 单 feature**，温控器是**单 endpoint 多 feature**。

`products/thermostat/configuration/data_model_wifi.zap` 关键定义：

- endpoint 0 = root node（不可改）
- endpoint 1 → endpointType `deviceTypeRef.code = 769`（`MA-thermostat`），仅一个 endpoint
- 该 endpointType 启用 **Thermostat cluster**（`"code": 513`，`"define": "THERMOSTAT_CLUSTER"`），关键 attribute：

| 属性名 | code | 类型 | 默认值 | 对应 feature_id |
|---|---|---|---|---|
| `LocalTemperature` | 0 | temperature（int16，°C×100） | — | `LOW_CODE_FEATURE_ID_TEMPERATURE` (4004) |
| `OccupiedCoolingSetpoint` | 17 | temperature | 2600 | `LOW_CODE_FEATURE_ID_COOLING_SETPOINT` (4005) |
| `OccupiedHeatingSetpoint` | 18 | temperature | 2000 | `LOW_CODE_FEATURE_ID_HEATING_SETPOINT` (4006) |
| `AbsMinHeatSetpointLimit` | 3 | temperature | 700 | — |
| `AbsMaxHeatSetpointLimit` | 4 | temperature | 3000 | — |
| `AbsMinCoolSetpointLimit` | 5 | temperature | 1600 | — |
| `AbsMaxCoolSetpointLimit` | 6 | temperature | 3200 | — |
| `SystemMode` | 28 | SystemModeEnum | 1 | — |
| `ControlSequenceOfOperation` | 27 | ControlSequenceOfOperationEnum | 4 | — |

`Thermostat.FeatureMap`（code 65532）默认值为 `3`（bit0=Heating + bit1=Cooling 同时启用）。命令 `SetpointRaiseLower`（code 0，incoming）已启用。

> 温度单位是 **°C×100 的带符号 int16**（如 21.50°C → `2150`，零下 5°C → `-500`），所以应用侧必须用 `int16_t` 与 `LOW_CODE_VALUE_TYPE_INTEGER`，**不要**用 `UNSIGNED_INTEGER`。

### 2. `product_info.json`（取自 `products/thermostat`）

`device_type_id` 必须是 `769`：

```json
{
    "config_version": 3,
    "vendor_id": 65521,
    "product_id": 32768,
    "device_type_id": 769,
    "chip": "esp32c6",
    "connection_type": "wifi",
    "product_type": "thermostat",
    "solution_type": "low_code"
}
```

### 3. `feature_update_from_system`：单 endpoint 内按 feature_id 三分发

核心模式——同一个 `endpoint_id == 1` 下用 `if ... else if` 分发三个 feature_id（取自 `products/thermostat/main/app_main.cpp`）：

```cpp
#include <low_code.h>
#include "app_priv.h"

int feature_update_from_system(low_code_feature_data_t *data)
{
    uint16_t endpoint_id = data->details.endpoint_id;
    uint32_t feature_id = data->details.feature_id;

    if (endpoint_id == 1) {
        if (feature_id == LOW_CODE_FEATURE_ID_TEMPERATURE) {            // 温度
            int16_t temperature = *(int16_t *)data->value.value;
            app_driver_set_temperature(temperature);
            printf("%s: Feature update: temperature: %d\n", TAG, temperature);
        } else if (feature_id == LOW_CODE_FEATURE_ID_COOLING_SETPOINT) { // 制冷设定点
            int16_t cooling_setpoint = *(int16_t *)data->value.value;
            app_driver_set_cooling_setpoint(cooling_setpoint);
            printf("%s: Feature update: cooling setpoint: %d\n", TAG, cooling_setpoint);
        } else if (feature_id == LOW_CODE_FEATURE_ID_HEATING_SETPOINT) { // 制热设定点
            int16_t heating_setpoint = *(int16_t *)data->value.value;
            app_driver_set_heating_setpoint(heating_setpoint);
            printf("%s: Feature update: heating setpoint: %d\n", TAG, heating_setpoint);
        }
    }
    return 0;
}
```

注意与多 endpoint 插座的差异：插座按 `endpoint_id == 1 || endpoint_id == 2` 选继电器；温控器固定 `endpoint_id == 1`，**按 `feature_id` 选语义**。

### 4. 驱动回调（占位骨架，取自 `products/thermostat/main/app_driver.cpp`）

仓库提供的是带占位的模板（`printf` + 注释），实际硬件逻辑需自行补全：

```cpp
#include "app_priv.h"

int app_driver_init()
{
    printf("%s: Initializing thermostat driver\n", TAG);
    /* TODO: 加温控硬件初始化（温湿度传感器、继电器/执行器、显示等） */
    return 0;
}

int app_driver_set_temperature(int16_t temperature)
{
    /* temperature 单位是 °C×100，可负 */
    printf("%s: Setting temperature: %d\n", TAG, temperature);
    /* TODO: 更新 LocalTemperature 读数源（如 SHT30，见 recipes/sht30_sensor.md） */
    return 0;
}

int app_driver_set_cooling_setpoint(int16_t cooling_setpoint)
{
    printf("%s: Setting cooling setpoint: %d\n", TAG, cooling_setpoint);
    /* TODO: 控制制冷执行器，或在显示上更新目标制冷温度 */
    return 0;
}

int app_driver_set_heating_setpoint(int16_t heating_setpoint)
{
    printf("%s: Setting heating setpoint: %d\n", TAG, heating_setpoint);
    /* TODO: 控制制热执行器 */
    return 0;
}
```

### 5. 主动上报当前温度

温控器通常还要周期上报 `LocalTemperature`（`feature_id` 同样是 `LOW_CODE_FEATURE_ID_TEMPERATURE`），用 `system_timer` 周期读取传感器后上报（周期定时器见 `recipes/system_timer.md`，SHT30 读法见 `recipes/sht30_sensor.md`）：

```cpp
static void app_driver_report_temperature(float temp)
{
    int16_t temperature = (int16_t)(temp * 100);   /* °C×100，带符号 */
    low_code_feature_data_t update_data = {
        .details = { .endpoint_id = 1, .feature_id = LOW_CODE_FEATURE_ID_TEMPERATURE },
        .value = {
            .type = LOW_CODE_VALUE_TYPE_INTEGER,     /* 温度可负，必须 INTEGER */
            .value_len = sizeof(int16_t),
            .value = (uint8_t *)&temperature,
        },
    };
    low_code_feature_update_to_system(&update_data);
}
```

### 6. 事件处理（仓库内最完整的 switch 参考）

`products/thermostat/main/app_driver.cpp` 的 `app_driver_event_handler()` 是仓库内**覆盖全部 `LOW_CODE_EVENT_*` 用例最完整**的版本——每个 case 都带 `printf`，可直接作为事件处理骨架参考（完整代码见 `recipes/event_handling.md` 与源文件）。

### 7. setup / loop / main（标准骨架）

温控器的 `app_main.cpp` 仍是标准骨架，没有特殊处理：

```cpp
static void setup()
{
    low_code_register_callbacks(feature_update_from_system, event_from_system);
    app_driver_init();
}

static void loop()
{
    low_code_get_feature_update_from_system();
    low_code_get_event_from_system();
}

extern "C" int main()
{
    printf("%s: Starting low code\n", TAG);
    system_setup();   // 必须最先
    setup();
    while (1) {
        system_loop();
        loop();
    }
    return 0;
}
```

## 常见错误

| 错误 | 原因 | 解决方法 |
|---|---|---|
| 三个特性互相串台 | 单 endpoint 内只判了 endpoint 没判 feature_id | 在 `endpoint_id == 1` 内再 `if/else if` 按 `TEMPERATURE/COOLING_SETPOINT/HEATING_SETPOINT` 分发 |
| 温度值解析异常（负温变超大正数） | 用了 `uint16_t` 或 `UNSIGNED_INTEGER` | 一律用 `int16_t` + `LOW_CODE_VALUE_TYPE_INTEGER` |
| 单位错误（直接传浮点或 °C） | 把 `float` 或整数 °C 直接塞进 value | 换算成 `°C×100` 的 `int16_t`（21.5°C→2150） |
| 设定值被 Matter 拒绝 | 超出 zap 中的 AbsMin/AbsMax 限值 | 核对 `AbsMinHeatSetpointLimit`(700)/`AbsMaxHeatSetpointLimit`(3000)/`AbsMinCoolSetpointLimit`(1600)/`AbsMaxCoolSetpointLimit`(3200) |
| App 看不到温控分类 | `product_info.json` 的 `device_type_id` 不是 769 | 改为 `"device_type_id": 769`，并重跑 Upload Configuration |
| 上报后 App 不刷新 | feature_id 与 zap cluster attribute 不对应 | 上报 `LocalTemperature` 用 `LOW_CODE_FEATURE_ID_TEMPERATURE`；设定点下发只接不发 |
| 改了 zap 不生效 | 未重跑 Upload Configuration | 编辑 `data_model_wifi.zap` 后必须重新生成并烧录 `data_model.bin` |

## 参考

- `products/thermostat/main/app_main.cpp`（单 endpoint 三 feature 分发）
- `products/thermostat/main/app_driver.cpp`（占位驱动 + 完整事件 switch）
- `products/thermostat/main/app_priv.h`（驱动函数原型）
- `products/thermostat/configuration/data_model_wifi.zap`（endpoint 1 = MA-thermostat 769，Thermostat cluster 513）
- `products/thermostat/configuration/product_info.json`（`device_type_id: 769`）
- `components/low_code/low_code.h`（`LOW_CODE_FEATURE_ID_TEMPERATURE/COOLING_SETPOINT/HEATING_SETPOINT`、`low_code_feature_data_t`）
- `recipes/feature_update.md`（feature_id 路由与上报通用说明）、`recipes/event_handling.md`（事件 switch）、`recipes/sht30_sensor.md`（温度读取与 `system_timer` 周期上报）、`recipes/relay_socket.md`（对照：多 endpoint 单 feature 的反向模式）
