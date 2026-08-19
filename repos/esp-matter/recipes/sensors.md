# 传感器设备（温度 / 湿度 / 占用 / 接触）

> **适用摘要**: 在一个 Matter 节点上创建多个传感器 endpoint（温度、湿度、占用），从传感器驱动异步拿到数据后，用 `chip::DeviceLayer::SystemLayer().ScheduleLambda(...)` 切到 Matter 线程，再 `attribute::update()` 上报。

> Version: selected repo/component version; align with the user project and dependency manifest.
> Evidence: `repos/esp-matter/resources/`, source/examples in `repos/esp-matter/`, and this recipe path `repos/esp-matter/recipes/sensors.md`.
> Validation: draft metadata added from repository routing; verify APIs, Kconfig symbols, and component dependencies against the selected version.

## 触发意图

- "做一个 Matter 温湿度传感器"
- "Matter 传感器怎么上报数值"
- "occupancy sensor"
- "temperature_sensor / humidity_sensor 怎么建"
- "ScheduleLambda"

## 前置条件

| 条件 | 要求 |
|---|---|
| 参考工程 | `examples/sensors/`（SHTC3 温湿度 + PIR 占用） |
| 头文件 | `esp_matter_data_model.h`（`attribute::get/get_val/update`） |

## 分步说明

### 1. 建多个传感器 endpoint

来自 `examples/sensors/main/app_main.cpp` 的模式：温度在 endpoint 1、湿度在 2、占用在 3。

```cpp
using namespace esp_matter::endpoint;
using namespace chip::app::Clusters;

// 温度传感器
temperature_sensor::config_t temp_cfg;
endpoint_t *temp_ep = temperature_sensor::create(node, &temp_cfg, ENDPOINT_FLAG_NONE, nullptr);
uint16_t temp_ep_id = endpoint::get_id(temp_ep);

// 湿度传感器
humidity_sensor::config_t hum_cfg;
endpoint_t *hum_ep = humidity_sensor::create(node, &hum_cfg, ENDPOINT_FLAG_NONE, nullptr);
uint16_t hum_ep_id = endpoint::get_id(hum_ep);

// 占用传感器
occupancy_sensor::config_t occ_cfg;
endpoint_t *occ_ep = occupancy_sensor::create(node, &occ_cfg, ENDPOINT_FLAG_NONE, nullptr);
uint16_t occ_ep_id = endpoint::get_id(occ_ep);
```

> 接触式传感器用 `contact_sensor::create()`（BooleanState cluster）；光照传感器用 `light_sensor::create()`。

### 2. 传感器数值单位（Matter 规范）

| cluster | 属性 | 单位 / 缩放 |
|---|---|---|
| TemperatureMeasurement | MeasuredValue | 0.01 ℃（`int16_t`，25.45℃ = 2545） |
| RelativeHumidityMeasurement | MeasuredValue | 0.01 %（`uint16_t`，50.00% = 5000） |
| OccupancySensing | Occupancy | bitmap bit0（`uint8_t`，0/1） |

### 3. 异步上报：ScheduleLambda → attribute::update

这是关键模式。传感器驱动在自己的 task 里拿到数据，**不能**直接调 `attribute::update()`，必须切到 Matter 线程。来自 `examples/sensors/main/app_main.cpp`：

```cpp
// 温度上报（驱动 task 调用）
static void temp_sensor_notification(uint16_t endpoint_id, float temp, void *user_data)
{
    chip::DeviceLayer::SystemLayer().ScheduleLambda([endpoint_id, temp]() {
        attribute_t *attr = attribute::get(endpoint_id, TemperatureMeasurement::Id,
                                           TemperatureMeasurement::Attributes::MeasuredValue::Id);
        esp_matter_attr_val_t val;
        attribute::get_val(attr, &val);
        val.val.i16 = static_cast<int16_t>(temp * 100);   // 0.01℃
        attribute::update(endpoint_id, TemperatureMeasurement::Id,
                          TemperatureMeasurement::Attributes::MeasuredValue::Id, &val);
    });
}

// 湿度上报
static void humidity_sensor_notification(uint16_t endpoint_id, float humidity, void *user_data)
{
    chip::DeviceLayer::SystemLayer().ScheduleLambda([endpoint_id, humidity]() {
        attribute_t *attr = attribute::get(endpoint_id, RelativeHumidityMeasurement::Id,
                                           RelativeHumidityMeasurement::Attributes::MeasuredValue::Id);
        esp_matter_attr_val_t val;
        attribute::get_val(attr, &val);
        val.val.u16 = static_cast<uint16_t>(humidity * 100);  // 0.01%
        attribute::update(endpoint_id, RelativeHumidityMeasurement::Id,
                          RelativeHumidityMeasurement::Attributes::MeasuredValue::Id, &val);
    });
}

// 占用上报（bitmap）
static void occupancy_sensor_notification(uint16_t endpoint_id, bool occupancy, void *user_data)
{
    chip::DeviceLayer::SystemLayer().ScheduleLambda([endpoint_id, occupancy]() {
        attribute_t *attr = attribute::get(endpoint_id, OccupancySensing::Id,
                                           OccupancySensing::Attributes::Occupancy::Id);
        esp_matter_attr_val_t val;
        attribute::get_val(attr, &val);
        val.val.u8 = occupancy ? 1 : 0;
        attribute::update(endpoint_id, OccupancySensing::Id,
                          OccupancySensing::Attributes::Occupancy::Id, &val);
    });
}
```

### 4. 注册驱动回调

把上面的 notification 函数注册给底层传感器驱动（如 SHTC3 I2C 驱动、PIR GPIO 中断），驱动在拿到新数据时调用它们。

### 5. `app_attribute_update_cb` 一般直接返回 OK

传感器属性通常是只读的 server 属性，不需要在 PRE_UPDATE 里驱动硬件：

```cpp
static esp_err_t app_attribute_update_cb(attribute::callback_type_t type, uint16_t endpoint_id,
                                         uint32_t cluster_id, uint32_t attribute_id,
                                         esp_matter_attr_val_t *val, void *priv_data) {
    return ESP_OK;   // 传感器无需联动硬件
}
```

## 常见错误

| 错误 | 原因 | 解决方法 |
|---|---|---|
| 上报时崩溃 / 死锁 | 驱动 task 直接调 `attribute::update` | 用 `ScheduleLambda` 切到 Matter 线程 |
| 温度读出来偏 100 倍 | 没乘 100 | Matter 温度是 0.01℃ |
| 占用读不到 | 传了 bool 而非 bitmap | `val.val.u8 = occupied ? 1 : 0` |
| `MeasuredValue` 上限保护 | `min/max_measured_value` 没设合理 | device type 默认会建 min/max 属性，超出范围会被框架拒 |
| 多传感器 endpoint 混 | endpoint_id 用错 | 每个 endpoint 记下自己的 id |

## 参考

- `D:/esp-skill/espressif-repos/esp-matter/examples/sensors/main/app_main.cpp`（temp/humidity/occupancy notification 范式）
- `D:/esp-skill/espressif-repos/esp-matter/examples/sensors/README.md`（SHTC3 + PIR 接线）
- `D:/esp-skill/espressif-repos/esp-matter/components/esp_matter/data_model/legacy/esp_matter_endpoint_impl.h`（`temperature_sensor` / `humidity_sensor` / `occupancy_sensor` / `contact_sensor` / `light_sensor`）
- `D:/esp-skill/espressif-repos/esp-matter/components/esp_matter/data_model/esp_matter_attribute_utils.h`（`update` / `get` / `get_val`）
