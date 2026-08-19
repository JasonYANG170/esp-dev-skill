# Wi-Fi 服务：扫描、连接与 SoftAP 配网

> **适用摘要**: 使用 `brookesia_service_wifi` 完成 AP 扫描、STA 连接/断开、SoftAP 配网、状态/历史查询与事件订阅。Wi-Fi 服务为纯芯片工程，无需 HAL 设备初始化。

> Version: selected repo/component version; align with the user project and dependency manifest.
> Evidence: `repos/esp-brookesia/resources/`, source/examples in `repos/esp-brookesia/`, and this recipe path `repos/esp-brookesia/recipes/wifi_service.md`.
> Validation: draft metadata added from repository routing; verify APIs, Kconfig symbols, and component dependencies against the selected version.

## 触发意图

- "用 Brookesia 连 Wi-Fi"
- "Wi-Fi 扫描 AP"
- "SoftAP 配网"
- "订阅 Wi-Fi 连接事件"
- "查询已连接的 AP"

## 前置条件

| 条件 | 要求 |
|---|---|
| 依赖 | `espressif/brookesia_service_wifi`（可选 `brookesia_service_nvs` 存凭据）|
| 目标 | `idf.py set-target esp32s3`（或其它带 Wi-Fi 的 ESP 芯片）|
| 参考示例 | `examples/service/wifi` |

## 分步说明

### 1. 启动 ServiceManager 与绑定（含 NVS 存凭据）

```cpp
auto &service_manager = service::ServiceManager::get_instance();
service_manager.init();
service_manager.start();
BROOKESIA_CHECK_FALSE_EXIT(WifiHelper::is_available(), "Wifi service is not available");
auto binding = service_manager.bind(WifiHelper::get_name().data());
BROOKESIA_CHECK_FALSE_EXIT(binding.is_valid(), "Failed to bind Wifi service");
```

### 2. 订阅事件

```cpp
std::vector<service::EventRegistry::SignalConnection> conns;

// 通用事件 schema: Event(String), IsUnexpected(Boolean)
conns.push_back(WifiHelper::subscribe_event(
    WifiHelper::EventId::GeneralEventHappened,
    [](const std::string &, const std::string & event, bool is_unexpected) {
        BROOKESIA_LOGI("WiFi event: %1% (unexpected=%2%)", event, is_unexpected);
    }));

// 扫描结果 schema: ApInfos(Array)
conns.push_back(WifiHelper::subscribe_event(
    WifiHelper::EventId::ScanApInfosUpdated,
    [](const std::string &, const boost::json::array & ap_infos) {
        std::vector<WifiHelper::ScanApInfo> aps;
        BROOKESIA_DESCRIBE_FROM_JSON(ap_infos, aps);
        BROOKESIA_LOGI("Scanned %1% APs", aps.size());
    }));
```

### 3. Start 并等待 Started

```cpp
using GeneralMonitor = WifiHelper::EventMonitor<WifiHelper::EventId::GeneralEventHappened>;
GeneralMonitor monitor;
monitor.start();

auto start_result = WifiHelper::call_function_sync(
    WifiHelper::FunctionId::TriggerGeneralAction,
    BROOKESIA_DESCRIBE_TO_STR(WifiHelper::GeneralAction::Start));
BROOKESIA_CHECK_FALSE_EXIT(start_result, "Failed to start WiFi");

bool got = monitor.wait_for(std::vector<service::EventItem>{
    BROOKESIA_DESCRIBE_TO_STR(WifiHelper::GeneralEvent::Started), false
}, 2000);
BROOKESIA_CHECK_FALSE_EXIT(got, "Wait Started timeout");
monitor.stop();
monitor.clear();
```

### 4. 扫描 AP

```cpp
// 设置扫描参数：Object（由 ScanParams 序列化）
WifiHelper::ScanParams scan_params{
    .ap_count = 10,
    .interval_ms = 10000,
    .timeout_ms = 20000,
};
WifiHelper::call_function_sync(
    WifiHelper::FunctionId::SetScanParams,
    BROOKESIA_DESCRIBE_TO_JSON(scan_params).as_object());

// 启动扫描
WifiHelper::call_function_sync(WifiHelper::FunctionId::TriggerScanStart);
```

### 5. 连接 AP

```cpp
// 设置目标 AP（schema: SSID(String) -> Password(String)）
WifiHelper::call_function_sync(
    WifiHelper::FunctionId::SetConnectAp, "ssid1", "password1", service::helper::Timeout(100));

// 触发连接
GeneralMonitor m2; m2.start();
WifiHelper::call_function_sync(
    WifiHelper::FunctionId::TriggerGeneralAction,
    BROOKESIA_DESCRIBE_TO_STR(WifiHelper::GeneralAction::Connect));

bool connected = m2.wait_for(std::vector<service::EventItem>{
    BROOKESIA_DESCRIBE_TO_STR(WifiHelper::GeneralEvent::Connected), false
}, 10000);
```

### 6. 查询状态与已连接 AP

```cpp
// 当前状态（返回 String，反序列化为枚举）
auto st = WifiHelper::call_function_sync<std::string>(WifiHelper::FunctionId::GetGeneralState);
WifiHelper::GeneralState state = WifiHelper::GeneralState::Max;
BROOKESIA_DESCRIBE_STR_TO_ENUM(st.value(), state);   // Started/Connected/Connecting/...

// 已连接 AP（含历史，返回 Array）
auto aps = WifiHelper::call_function_sync<boost::json::array>(WifiHelper::FunctionId::GetConnectedAps);
std::vector<WifiHelper::ConnectApInfo> infos;
BROOKESIA_DESCRIBE_FROM_JSON(aps.value(), infos);
```

### 7. SoftAP 配网（首次配网）

```cpp
using SoftApMonitor = WifiHelper::EventMonitor<WifiHelper::EventId::SoftApEventHappened>;
SoftApMonitor sm; sm.start();

WifiHelper::call_function_sync(
    WifiHelper::FunctionId::TriggerSoftApProvisionStart,
    service::helper::Timeout(1000));

// 等 SoftAP Started
sm.wait_for(std::vector<service::EventItem>{
    BROOKESIA_DESCRIBE_TO_STR(WifiHelper::SoftApEvent::Started)
}, 10000);

// 读取实际 SoftAP 参数
auto sap = WifiHelper::call_function_sync<boost::json::object>(WifiHelper::FunctionId::GetSoftApParams);
WifiHelper::SoftApParams params;
BROOKESIA_DESCRIBE_FROM_JSON(sap.value(), params);
// params.ssid / params.password —— 手机连接后在配网页输入家中 Wi-Fi

// 等待手机配网成功 -> 设备自动连上家中 Wi-Fi
GeneralMonitor m3; m3.start();
m3.wait_for(std::vector<service::EventItem>{
    BROOKESIA_DESCRIBE_TO_STR(WifiHelper::GeneralEvent::Connected), false
}, 120000);

// 停止配网
WifiHelper::call_function_sync(
    WifiHelper::FunctionId::TriggerSoftApProvisionStop, service::helper::Timeout(1000));
```

### 8. 清空数据（恢复出厂凭据）

```cpp
WifiHelper::call_function_sync(WifiHelper::FunctionId::ResetData);
```

## 常见错误

| 错误 | 原因 | 解决方法 |
|---|---|---|
| 一直等不到 Started | 未先 `TriggerGeneralAction(Start)` | 显式 Start（init 通常会自动触发）|
| 扫描无结果 | 未设置 ScanParams 或超时太短 | 先 `SetScanParams` 再 `TriggerScanStart` |
| 连接失败 | SSID/密码错或顺序反 | schema 顺序 SSID→Password；带 Timeout |
| 配网不进入 | Flash 残留旧凭据 | 先 `ResetData` 或出厂复位后再配网 |
| ESP_HOSTED 模式启动慢 | 等待 Start 超时不够 | `CONFIG_ESP_HOSTED_ENABLED` 时把 Start 超时调到 5000ms |

## 参考

- `examples/service/wifi/main/main.cpp` — 扫描/连接/自动连接/SoftAP 配网/重置全套演示
- `docs/en/service/wifi.rst` — Wi-Fi 服务接口契约
- `docs/en/service/usage.rst` — 服务开发指南
