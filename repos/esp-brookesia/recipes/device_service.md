# Device 服务：能力查询与外设控制

> **适用摘要**: 使用 `brookesia_service_device` 查询板级能力（capabilities）、读取板信息、控制显示背光与音频播放器（音量/静音）、查询存储文件系统与电池、订阅状态变化事件。Device 服务依赖 HAL，且常与 NVS 服务一起 bind（用于 `ResetData` 持久化）。

## 触发意图

- "查询板子支持哪些外设"
- "控制屏幕背光亮度/开关"
- "设置/读取音量、静音"
- "查询电池状态"
- "查询存储文件系统容量"

## 前置条件

| 条件 | 要求 |
|---|---|
| 依赖 | `espressif/brookesia_service_device`、`espressif/brookesia_service_nvs`、HAL |
| 目标 | 板级工程 `idf.py gen-bmgr-config -b <board>` |
| 参考示例 | `examples/service/device` |

## 分步说明

### 1. 初始化 HAL 与服务

```cpp
using DeviceHelper = service::helper::Device;
using NVSHelper    = service::helper::NVS;

hal::init_all_devices();

auto &service_manager = service::ServiceManager::get_instance();
service_manager.start();

auto nvs_binding    = service_manager.bind(NVSHelper::get_name().data());
auto device_binding = service_manager.bind(DeviceHelper::get_name().data());
```

### 2. 查询能力（capabilities）

```cpp
// 返回 Object: map<interface_name, instance_names>
auto cap = DeviceHelper::call_function_sync<boost::json::object>(DeviceHelper::FunctionId::GetCapabilities);
DeviceHelper::Capabilities capabilities;
BROOKESIA_DESCRIBE_FROM_JSON(cap.value(), capabilities);

// 判断某接口是否支持
auto has = [&](std::string_view name) {
    return capabilities.find(std::string(name)) != capabilities.end();
};
if (has(hal::DisplayBacklightIface::NAME))    { /* 支持背光控制 */ }
if (has(hal::AudioCodecPlayerIface::NAME))    { /* 支持音频播放器 */ }
if (has(hal::StorageFsIface::NAME))           { /* 支持存储查询 */ }
if (has(hal::PowerBatteryIface::NAME))        { /* 支持电池 */ }
```

### 3. 板信息查询

```cpp
auto bi = DeviceHelper::call_function_sync<boost::json::object>(DeviceHelper::FunctionId::GetBoardInfo);
DeviceHelper::BoardInfo info;
BROOKESIA_DESCRIBE_FROM_JSON(bi.value(), info);
```

### 4. 显示背光控制（带事件监听）

```cpp
using BrightnessMonitor = DeviceHelper::EventMonitor<DeviceHelper::EventId::DisplayBacklightBrightnessChanged>;
using OnOffMonitor      = DeviceHelper::EventMonitor<DeviceHelper::EventId::DisplayBacklightOnOffChanged>;

BrightnessMonitor bm; bm.start();
OnOffMonitor      om; om.start();

// 读取当前
auto cur  = DeviceHelper::call_function_sync<double>(DeviceHelper::FunctionId::GetDisplayBacklightBrightness);
auto onff = DeviceHelper::call_function_sync<bool>(DeviceHelper::FunctionId::GetDisplayBacklightOnOff);

// 设置亮度（Number）与开关（Boolean）
double next = (cur.value() < 100.0) ? (cur.value() + 1.0) : 99.0;
DeviceHelper::call_function_sync(DeviceHelper::FunctionId::SetDisplayBacklightBrightness, next);
bm.wait_for(std::vector<service::EventItem>{ next }, 2000);

DeviceHelper::call_function_sync(DeviceHelper::FunctionId::SetDisplayBacklightOnOff, false);
om.wait_for(std::vector<service::EventItem>{ false }, 2000);
```

### 5. 音量/静音控制

```cpp
using VolumeMonitor = DeviceHelper::EventMonitor<DeviceHelper::EventId::AudioPlayerVolumeChanged>;
using MuteMonitor   = DeviceHelper::EventMonitor<DeviceHelper::EventId::AudioPlayerMuteChanged>;

VolumeMonitor vm; vm.start();
MuteMonitor   mm; mm.start();

auto vol  = DeviceHelper::call_function_sync<double>(DeviceHelper::FunctionId::GetAudioPlayerVolume);
auto mute = DeviceHelper::call_function_sync<bool>(DeviceHelper::FunctionId::GetAudioPlayerMute);

double next_vol = (vol.value() < 100.0) ? (vol.value() + 1.0) : 99.0;
DeviceHelper::call_function_sync(DeviceHelper::FunctionId::SetAudioPlayerVolume, next_vol);
vm.wait_for(std::vector<service::EventItem>{ next_vol }, 2000);

DeviceHelper::call_function_sync(DeviceHelper::FunctionId::SetAudioPlayerMute, true);
mm.wait_for(std::vector<service::EventItem>{ true }, 2000);
```

### 6. 存储文件系统查询

```cpp
// 返回 Array
auto fs = DeviceHelper::call_function_sync<boost::json::array>(DeviceHelper::FunctionId::GetStorageFileSystems);
for (const auto &item : fs.value()) {
    std::string mount = item.as_object().at("mount_point").as_string().c_str();
    // 容量：mount_point(String) -> Object
    auto cap = DeviceHelper::call_function_sync<boost::json::object>(
        DeviceHelper::FunctionId::GetStorageFileSystemCapacity, mount);
    BROOKESIA_LOGI("FS %1% capacity: %2%", mount, cap.value());
}
```

### 7. 电池查询与控制

```cpp
// 信息/状态/充电配置
auto info  = DeviceHelper::call_function_sync<boost::json::object>(DeviceHelper::FunctionId::GetPowerBatteryInfo);
auto state = DeviceHelper::call_function_sync<boost::json::object>(DeviceHelper::FunctionId::GetPowerBatteryState);
auto ccfg  = DeviceHelper::call_function_sync<boost::json::object>(DeviceHelper::FunctionId::GetPowerBatteryChargeConfig);

// 监听状态变化（schema: State(Object)）
auto batt_conn = DeviceHelper::subscribe_event(
    DeviceHelper::EventId::PowerBatteryStateChanged,
    [](const std::string &, const boost::json::object & state) {
        hal::PowerBatteryIface::State bs;
        BROOKESIA_DESCRIBE_FROM_JSON(state, bs);
    });

// 充电使能（Number 之前先用 info.has_ability(...) 判断能力）
DeviceHelper::call_function_sync(DeviceHelper::FunctionId::SetPowerBatteryChargingEnabled, true);
```

### 8. 恢复出厂（清空 NVS 持久化）

```cpp
DeviceHelper::call_function_sync(DeviceHelper::FunctionId::ResetData);
```

## 常见错误

| 错误 | 原因 | 解决方法 |
|---|---|---|
| 某接口 Get/Set 失败 | 该板不支持对应接口 | 先查 capabilities 再调用，不支持则跳过 |
| `ResetData` 后状态未变 | 未 bind NVS 服务 | `ResetData` 依赖 NVS，需先 bind NVSHelper |
| 电池控制失败 | 板子无 `ChargeConfig`/`ChargerControl` 能力 | 用 `PowerBatteryInfo::has_ability(...)` 判断后再操作 |
| 事件监听不到 | monitor 未 start 或 item 类型不符 | 背光是 Number、开关是 Boolean、电池是 Object |
| HAL 接口直接拿不到 | 没初始化 HAL | `hal::init_all_devices()` 先执行 |

## 参考

- `examples/service/device/main/main.cpp` — capabilities/board_info/display/audio/storage/battery/reset 全套
- `docs/en/service/device.rst` — Device 服务接口契约
- `docs/en/hal/interface/device.rst` — HAL 设备接口
