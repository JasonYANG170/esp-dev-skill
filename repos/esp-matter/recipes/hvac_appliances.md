# 暖通 / 家电设备类型（Thermostat / Refrigerator / Room AC）

> **适用摘要**: 用 esp-matter 标准 device type 创建暖通与白电 endpoint —— Thermostat（带 heating/cooling feature conformance）、Refrigerator + TemperatureControlledCabinet（父子 endpoint）、Room Air Conditioner（OnOff + Thermostat 组合）。涉及 cluster 级 feature flag 设置、父子 endpoint 关联、以及需要 delegate 的 cluster（TemperatureControl）。

## 触发意图

- "做 Matter 空调 / 冰箱 / 温控器"
- "thermostat cluster heating / cooling feature"
- "RefrigeratorAndTCCMode / ModeBase / OperationalState delegate"
- "room_air_conditioner / refrigerator endpoint"
- "set_parent_endpoint"
- "pump_configuration_and_control 构造参数"

## 前置条件

| 条件 | 要求 |
|---|---|
| 参考工程 | `examples/refrigerator/`、`examples/room_air_conditioner/`、`examples/all_device_types_app/` |
| 头文件 | `components/esp_matter/data_model/legacy/esp_matter_endpoint_impl.h`（`refrigerator` / `temperature_controlled_cabinet` / `room_air_conditioner` / `pump` / `thermostat`）、`.../esp_matter_cluster_impl.h`（`thermostat` / `pump_configuration_and_control` feature） |
| 文档 | `docs/en/developing.rst`（Thermostat heating feature、Pump ctor）、`docs/en/app_guide.rst`（delegate cluster 列表） |

## 分步说明

### 1. Room Air Conditioner（OnOff + Thermostat）

`room_air_conditioner::config_t` 同时含 OnOff 和 Thermostat 子结构（来自 `esp_matter_endpoint_impl.h`）：

```cpp
namespace room_air_conditioner {
typedef struct config : app_base_config {
    cluster::on_off::config_t on_off;
    cluster::thermostat::config_t thermostat;
} config_t;
}
```

`examples/room_air_conditioner/main/app_main.cpp` 只设了 `on_off`，Thermostat 用默认（system_mode=1=Auto, control_sequence_of_operation=4=Heating and Cooling）：

```cpp
room_air_conditioner::config_t rac_config;
rac_config.on_off.on_off = DEFAULT_POWER;
endpoint_t *endpoint = room_air_conditioner::create(node, &rac_config, ENDPOINT_FLAG_NONE,
                                                    room_air_conditioner_handle);
room_air_conditioner_endpoint_id = endpoint::get_id(endpoint);
```

如果要给 RAC 的 thermostat 启用具体 feature（如仅制热），参考步骤 3 在 `create()` 前修改 `rac_config.thermostat.feature_flags`。

### 2. Thermostat cluster 的 heating/cooling feature conformance

Thermostat cluster 对 Heating / Cooling 是 **O.a+ conformance**（至少要有一个）。`config_t` 把可选 feature 拆到子结构里（来自 `esp_matter_cluster_impl.h`）：

```cpp
namespace thermostat {
typedef struct config {
    nullable<int16_t> local_temperature;
    uint8_t control_sequence_of_operation;     // 默认 4 = Cooling and Heating
    uint8_t system_mode;                        // 默认 1 = Auto
    void *delegate;
    struct {
        feature::heating::config_t heating;
        feature::cooling::config_t cooling;
        feature::auto_mode::config_t auto_mode;
        feature::occupancy::config_t occupancy;
        feature::setback::config_t setback;
        feature::local_temperature_not_exposed::config_t local_temperature_not_exposed;
        feature::matter_schedule_configuration::config_t matter_schedule_configuration;
    } features;
    uint32_t feature_flags;
} config_t;
}
```

启用 Heating feature 的标准写法（来自 `docs/en/developing.rst`）：

```cpp
thermostat::config_t thermostat_config;
thermostat_config.features.heating.occupied_heating_setpoint = 2200;  // 1/100 °C = 22.00°C
thermostat_config.feature_flags = thermostat::feature::heating::get_id();
cluster::thermostat::create(endpoint, &(config->thermostat_config), CLUSTER_FLAG_SERVER);
```

> `feature::heating::get_id()` 返回该 feature 在 FeatureMap 中的 bit 位。Cooling 同理用 `feature::cooling::get_id()`，可按位或叠加。

### 3. Refrigerator + TemperatureControlledCabinet（父子 endpoint）

Refrigerator 的 cabinet 是独立的 device type endpoint，需要挂在 refrigerator 下面。来自 `examples/refrigerator/main/app_main.cpp`：

```cpp
// 父：Refrigerator（默认只带 descriptor，Identify/Groups/Scenes/RefrigeratorMode/RefrigeratorAlarm 都是可选）
refrigerator::config_t refrigerator_config;
endpoint_t *endpoint = refrigerator::create(node, &refrigerator_config, ENDPOINT_FLAG_NONE, NULL);
refrigerator_endpoint_id = endpoint::get_id(endpoint);

// 子：TemperatureControlledCabinet（默认只带 descriptor + temperature_control，
//     TemperatureMeasurement / RefrigeratorAndTCCMode 可选）
temperature_controlled_cabinet::config_t cabinet_config;
endpoint_t *endpoint1 = temperature_controlled_cabinet::create(node, &cabinet_config,
                                                                ENDPOINT_FLAG_NONE, NULL);
temp_ctrl_endpoint_id = endpoint::get_id(endpoint1);

// 关键：把 cabinet 挂到 refrigerator 下
esp_err_t err = set_parent_endpoint(endpoint1, endpoint);
```

`set_parent_endpoint()` 来自 `components/esp_matter/data_model/esp_matter_data_model.h`：

```cpp
esp_err_t set_parent_endpoint(endpoint_t *endpoint, endpoint_t *parent_endpoint);
```

### 4. TemperatureControl cluster 需要 delegate

Refrigerator 示例为 TemperatureControl 的 TemperatureLevel feature 注册了 delegate（`app/clusters/temperature-control-server/TemperatureControlCluster.h`）：

```cpp
#include <app/clusters/temperature-control-server/TemperatureControlCluster.h>
#include <static-supported-temperature-levels.h>

static chip::app::Clusters::TemperatureControl::AppSupportedTemperatureLevelsDelegate
    sAppSupportedTemperatureLevelsDelegate;

// 在 esp_matter::start() 之前
chip::app::Clusters::TemperatureControlCluster::SetDelegate(&sAppSupportedTemperatureLevelsDelegate);
```

`ESP_MATTER_TEMPERATURE_CONTROL_CLUSTER_ENDPOINT_COUNT`（默认 1）控制有多少 endpoint 启用 TemperatureLevel feature。要为多个 endpoint 启用，需改 `connectedhomeip/connectedhomeip/examples/all-clusters-app/all-clusters-common/src/static-supported-temperature-levels.cpp`（见 `examples/refrigerator/README.md`）。

### 5. Pump（构造参数：max_pressure / max_speed / max_flow）

Pump device type 的 `config_t` 构造接受三个 nullable 上限值（来自 `esp_matter_endpoint_impl.h`）：

```cpp
namespace pump {
typedef struct config : app_base_config {
    cluster::on_off::config_t on_off;
    cluster::pump_configuration_and_control::config_t pump_configuration_and_control;
    explicit config(
        nullable<int16_t> max_pressure = nullable<int16_t>(),
        nullable<uint16_t> max_speed   = nullable<uint16_t>(),
        nullable<uint16_t> max_flow    = nullable<uint16_t>()
    );
} config_t;
}
```

使用（来自 `docs/en/developing.rst`，注意原文档示例中 feature_flags 写法）：

```cpp
// endpoint 层：直接传构造参数
pump::config_t pump_config(/*max_pressure*/ 1, /*max_speed*/ 10, /*max_flow*/ 20);

// cluster 层（如单独创建 pump_configuration_and_control cluster）
pump_configuration_and_control::config_t pcac_config(1, 10, 20);
pcac_config.feature_flags = pump_configuration_and_control::feature::constant_pressure::get_id();
cluster_t *cluster = pump_configuration_and_control::create(endpoint, &pcac_config, CLUSTER_FLAG_SERVER);
```

> 三个上限值一旦在 `config_t` 构造时设定就不能改；不传则为 null。

### 6. 其它带 delegate 的家电 cluster（ModeBase / OperationalState 等）

`docs/en/app_guide.rst` 列出了所有需要应用实现 delegate 的 cluster，家电相关：Thermostat、RefrigeratorAndTCCMode、ModeBase（Mode Select）、OperationalState、OperationalStateOven、OperationalStateRVC、MicrowaveOvenControl、LaundryWasherMode、RVC Clean/Run Mode、DishwasherMode/Alarm、EnergyEvseMode、OvenMode、TemperatureControl 等。

每个 delegate 的参考实现头文件在 `app_guide.rst` 的表格中给出（如 `<app/clusters/mode-select-server/mode-select-delegate.h>`）。典型用法：

```cpp
// 1. 继承对应的 Delegate 接口类（或用现成的 AppXxxDelegate）
// 2. 实现所需虚函数（如 ModeBase 的 AppendMode / OnModeChange）
// 3. 在 esp_matter::start() 前调用 SetDelegate(instance) 注册
```

具体 delegate 接口见 connectedhomeip 的对应 `*-delegate.h` 与 `app_guide.rst` 的"Reference Implementation"列。

### 7. `app_attribute_update_cb` 联动硬件

RAC 示例的回调与灯具同构（`priv_data` 是 driver handle，按 endpoint/cluster/attribute 分发，仅处理 `PRE_UPDATE`）：

```cpp
static esp_err_t app_attribute_update_cb(attribute::callback_type_t type, uint16_t endpoint_id,
                                         uint32_t cluster_id, uint32_t attribute_id,
                                         esp_matter_attr_val_t *val, void *priv_data) {
    esp_err_t err = ESP_OK;
    if (type == PRE_UPDATE) {
        app_driver_handle_t driver_handle = (app_driver_handle_t)priv_data;
        err = app_driver_attribute_update(driver_handle, endpoint_id, cluster_id, attribute_id, val);
    }
    return err;
}
```

启动后用 `app_driver_room_air_conditioner_set_defaults(endpoint_id)` 把数据库当前值推给硬件。

## 常见错误

| 错误 | 原因 | 解决方法 |
|---|---|---|
| Thermostat 创建失败 / 不符合 spec | 没设 heating 或 cooling feature | 设 `feature_flags = feature::heating::get_id()`（或 cooling，至少其一） |
| Refrigerator 的 cabinet 客户端发现不到 | 没设父子关系 | 调 `set_parent_endpoint(cabinet, refrigerator)` |
| TemperatureControl 的 SupportedTemperatureLevels 为空 | 没注册 delegate | 调 `TemperatureControlCluster::SetDelegate(&delegate)` |
| Mode Select / OperationalState 不工作 | 没实现 delegate | 按 `app_guide.rst` 表格找参考头文件并实现 |
| Pump 上限无法运行时修改 | `config_t` 构造锁定 | 构造时一次确定 max_pressure/speed/flow |
| 空调开机没响应 | `priv_data` 不是 driver handle | `create()` 第 4 参传 driver handle |
| 多 endpoint TemperatureLevel 失效 | `ESP_MATTER_TEMPERATURE_CONTROL_CLUSTER_ENDPOINT_COUNT` 太小 | 调大并扩展 `static-supported-temperature-levels.cpp` |

## 参考

- `D:/esp-skill/espressif-repos/esp-matter/examples/refrigerator/main/app_main.cpp` — refrigerator + cabinet + `set_parent_endpoint` + TemperatureControl delegate
- `D:/esp-skill/espressif-repos/esp-matter/examples/refrigerator/README.md` — TemperatureLevel feature / endpoint count 配置
- `D:/esp-skill/espressif-repos/esp-matter/examples/room_air_conditioner/main/app_main.cpp` — `room_air_conditioner::create` + OnOff 默认值 + 启动后 set_defaults
- `D:/esp-skill/espressif-repos/esp-matter/examples/room_air_conditioner/README.md`
- `D:/esp-skill/espressif-repos/esp-matter/examples/all_device_types_app/` — 所有 esp-matter device type 综合测试 App
- `D:/esp-skill/espressif-repos/esp-matter/components/esp_matter/data_model/legacy/esp_matter_endpoint_impl.h` — `refrigerator` / `temperature_controlled_cabinet` / `room_air_conditioner` / `pump` / `thermostat` namespace
- `D:/esp-skill/espressif-repos/esp-matter/components/esp_matter/data_model/legacy/esp_matter_cluster_impl.h` — `thermostat::config_t` (features.heating/cooling/...) / `pump_configuration_and_control::config_t` (feature::constant_pressure)
- `D:/esp-skill/espressif-repos/esp-matter/components/esp_matter/data_model/esp_matter_data_model.h` — `set_parent_endpoint(endpoint_t *, endpoint_t *)`
- `D:/esp-skill/espressif-repos/esp-matter/docs/en/developing.rst` — Thermostat heating feature / Pump ctor 示例
- `D:/esp-skill/espressif-repos/esp-matter/docs/en/app_guide.rst` — 需要 delegate 的 cluster 列表与参考实现头文件
- connectedhomeip `app/clusters/temperature-control-server/`、`mode-select-server/`、`operational-state-server/` 等 delegate 头文件
