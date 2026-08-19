# 上报温度传感器集群测量值（TemperatureMeasurement server）

> **适用摘要**: 把一个传感器读数（温度）发布到 Matter `TemperatureMeasurement` 集群（`ZCL_TEMP_MEASUREMENT_CLUSTER_ID = 0x0402`）。核心调用是 `emberAfTemperatureMeasurementClusterSetMeasuredValueCallback(endpoint, int16_t)`，其中 `int16_t` 值为摄氏度 × 100。这是 sensor 类设备类型的规范上报模式（与 actuator-only 的 OnOff → GPIO 模式相对）。

> Version: selected repo/component version; align with the user project and dependency manifest.
> Evidence: `repos/connectedhomeip/resources/`, source/examples in `repos/connectedhomeip/`, and this recipe path `repos/connectedhomeip/recipes/temperature_measurement_cluster.md`.
> Validation: draft metadata added from repository routing; verify APIs, Kconfig symbols, and component dependencies against the selected version.

## 触发意图

- "温度上报"
- "TemperatureMeasurement cluster"
- "发布传感器读数到 Matter"
- "emberAfTemperatureMeasurementClusterSetMeasuredValueCallback"
- "温度传感器示例"

## 前置条件

| 条件 | 要求 |
|---|---|
| 参考示例 | `examples/temperature-measurement-app/esp32/`（最小 sensor 示例）、`examples/all-clusters-app/esp32/`（同时演示温度属性写入） |
| 服务端头文件 | `<app/clusters/temperature-measurement-server/temperature-measurement-server.h>` |
| 生成宏 | `<app/common/gen/cluster-id.h>`（`ZCL_TEMP_MEASUREMENT_CLUSTER_ID`）、`<app/common/gen/attribute-id.h>`（`ZCL_TEMP_MEASURED_VALUE_ATTRIBUTE_ID = 0x0000`） |
| zap 配置 | 目标 endpoint 必须在 `.zap` 中含 `TemperatureMeasurement` server（temperature-measurement-app 自带 `temperature-measurement.zap`） |

## 分步说明

### 1. 选择最小 sensor 示例作为起点

`temperature-measurement-app` 是仓库中最小的 ESP32 sensor 示例，`app_main` 仅做标准初始化后进入空循环（`while(true) vTaskDelay`），集群上报逻辑全部由 zap 生成的 server 代码处理：

```cpp
// examples/temperature-measurement-app/esp32/main/main.cpp
extern "C" void app_main()
{
    nvs_flash_init();                                  // 1. ESP NVS
    CHIPDeviceManager & deviceMgr = CHIPDeviceManager::GetInstance();
    deviceMgr.Init(&EchoCallbacks);                    // 2. CHIP DeviceLayer
    InitServer();                                      // 3. Matter Server

    while (true) { vTaskDelay(50 / portTICK_PERIOD_MS); }  // UI 循环
}
```

该示例的 `DeviceCallbacks::PostAttributeChangeCallback` 目前只打日志（`"Unhandled cluster ID"`），因为它是 sensor（上报方），不需要响应 controller 写入。

### 2. 用服务端回调更新 MeasuredValue

服务端头文件暴露了三个 setter 和三个 getter，全部接受 `int16_t`（摄氏度 × 100，例如 21.00 °C = `2100`）：

```cpp
// src/app/clusters/temperature-measurement-server/temperature-measurement-server.h
EmberAfStatus emberAfTemperatureMeasurementClusterSetMeasuredValueCallback(chip::EndpointId endpoint, int16_t measuredValue);
EmberAfStatus emberAfTemperatureMeasurementClusterSetMinMeasuredValueCallback(chip::EndpointId endpoint, int16_t minMeasuredValue);
EmberAfStatus emberAfTemperatureMeasurementClusterSetMaxMeasuredValueCallback(chip::EndpointId endpoint, int16_t maxMeasuredValue);

EmberAfStatus emberAfTemperatureMeasurementClusterGetMeasuredValue(chip::EndpointId endpoint, int16_t * measuredValue);
EmberAfStatus emberAfTemperatureMeasurementClusterGetMinMeasuredValue(chip::EndpointId endpoint, int16_t * minMeasuredValue);
EmberAfStatus emberAfTemperatureMeasurementClusterGetMaxMeasuredValue(chip::EndpointId endpoint, int16_t * maxMeasuredValue);
```

`all-clusters-app` 在初始化 pretend 设备时这样写入 21 °C：

```cpp
// examples/all-clusters-app/esp32/main/main.cpp :: SetupPretendDevices()
// 写入温度属性：21 °C -> 2100（int16_t，单位 0.01 °C）
emberAfTemperatureMeasurementClusterSetMeasuredValueCallback(1, static_cast<int16_t>(21 * 100));
```

M5Stack 屏幕上 "+" / "-" 按钮调整温度时也走同一函数：

```cpp
// examples/all-clusters-app/esp32/main/main.cpp :: EditAttributeListModel::ItemAction()
if (name == "Temperature") {
    emberAfTemperatureMeasurementClusterSetMeasuredValueCallback(1, static_cast<int16_t>(n * 100));
}
```

### 3. 从真实传感器周期性上报

在应用任务或定时器里读取传感器（例如 I2C/SPI 温度芯片），换算为 `int16_t` 后调用 setter：

```cpp
#include <app/clusters/temperature-measurement-server/temperature-measurement-server.h>

// 读真实传感器（示例伪代码），milliC = 摄氏度 × 100，截断到 int16_t
int16_t PublishTemperature(int16_t milliC)
{
    EmberAfStatus status = emberAfTemperatureMeasurementClusterSetMeasuredValueCallback(1, milliC);
    if (status != EMBER_ZCL_STATUS_SUCCESS) {
        ESP_LOGE(TAG, "set measured value failed: 0x%02x", status);
    }
    return status;
}
```

注意：`MeasuredValue` 范围受 `MinMeasuredValue` / `MaxMeasuredValue` 约束。若设置值超出 `[min, max]`，server 将返回 `EMBER_ZCL_STATUS_INVALID_VALUE`。务必保证 `min < max` 且 `min ≤ measured ≤ max`（zap 默认 min/max 也为可写属性）。

### 4. 配网后用 chip-tool 读取 / 订阅

temperature-measurement-app 支持 `TemperatureMeasurement` 与 `Basic` 集群。配网（默认 BLE，discriminator `3840` / pin `20202021`）后，chip-tool 的 `temperaturemeasurement` 集群命令集为：

```bash
# chip-tool 支持的 temperaturemeasurement 子命令（仅读/订阅，无写）
chip-tool temperaturemeasurement
#   discover / read / report

# 读取 endpoint 1 的 MeasuredValue
chip-tool temperaturemeasurement read measured-value 1

# 订阅周期性上报
chip-tool temperaturemeasurement report measured-value 1
```

## 常见错误

| 错误 | 原因 | 解决方法 |
|---|---|---|
| setter 返回 `EMBER_ZCL_STATUS_INVALID_VALUE` | measured 值超出 `[min, max]` 区间 | 先用 `SetMin/MaxMeasuredValueCallback` 设合理边界（如 0 和 10000） |
| chip-tool `read` 拿不到值 | endpoint 未实现 TemperatureMeasurement server | 在 `.zap` 中为该 endpoint 添加 TemperatureMeasurement server，重新生成 |
| 单位混乱（显示 2100 °C） | 误把整数摄氏度直接当 measured 值 | 记住单位是 0.01 °C：`static_cast<int16_t>(celsius * 100)` |
| `SetMeasuredValueCallback` 未定义 | 未包含服务端头文件 | `#include <app/clusters/temperature-measurement-server/temperature-measurement-server.h>` |
| 负温度读数异常 | 截断到错误类型 | 用 `int16_t`（有符号），不要用 `uint16_t` |

## 参考项目

- `examples/temperature-measurement-app/esp32/` — 最小 ESP32 sensor 示例（`main/main.cpp` 标准初始化、`temperature-measurement.zap`）
- `examples/all-clusters-app/esp32/main/main.cpp` — `SetupPretendDevices()` 与 `EditAttributeListModel::ItemAction()` 演示 `SetMeasuredValueCallback`
- `src/app/clusters/temperature-measurement-server/temperature-measurement-server.h` — 服务端 setter/getter 签名
- `examples/chip-tool/README.md` — `temperaturemeasurement` 集群命令集
