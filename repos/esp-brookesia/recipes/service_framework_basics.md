# 服务框架通用范式：调用与订阅

> **适用摘要**: 掌握 ESP-Brookesia 服务框架的统一调用范式——ServiceManager 启动、bind、同步/异步函数调用、事件订阅与 EventMonitor 阻塞等待。这是使用所有具体服务（Wi-Fi/NVS/Audio 等）的共同基础。

## 触发意图

- "怎么调用 Brookesia 服务函数"
- "call_function_sync 和 async 区别"
- "怎么订阅服务事件"
- "EventMonitor 怎么用"
- "subscribe_event 的连接怎么保活"

## 前置条件

| 条件 | 要求 |
|---|---|
| 参考示例 | `examples/service/wifi/main/main.cpp` |
| 参考文档 | `docs/en/service/usage.rst` |

## 分步说明

### 1. 启动 ServiceManager 与绑定服务

```cpp
auto &service_manager = service::ServiceManager::get_instance();
service_manager.init();
service_manager.start();

// 可用性检查
if (!WifiHelper::is_available()) { /* 服务组件未链接 */ }

// bind 返回 RAII 对象，存活期间服务运行
auto binding = service_manager.bind(WifiHelper::get_name().data());
if (!binding.is_valid()) { /* 启动失败 */ }
```

### 2. 同步调用（阻塞至完成或超时）

```cpp
// 无返回值：省略模板参数
auto r0 = WifiHelper::call_function_sync(
    WifiHelper::FunctionId::SetConnectAp, "ssid1", "password1", service::helper::Timeout(100));
if (!r0) { BROOKESIA_LOGE("Failed: %1%", r0.error()); }

// 有返回值：显式模板参数
auto r1 = WifiHelper::call_function_sync<boost::json::array>(WifiHelper::FunctionId::GetConnectedAps);
if (r1) {
    std::vector<WifiHelper::ConnectApInfo> infos;
    BROOKESIA_DESCRIBE_FROM_JSON(r1.value(), infos);
}
```

`Timeout(ms)` 省略时用默认 `BROOKESIA_SERVICE_MANAGER_DEFAULT_CALL_FUNCTION_TIMEOUT_MS`。

### 3. 异步调用（立即返回，结果走 handler）

```cpp
auto on_done = [](service::FunctionResult &&result) {
    if (!result.success) {
        BROOKESIA_LOGE("Failed: %1%", result.error_message);
        return;
    }
    // auto &v = result.get_data<ReturnType>();
};
auto r = WifiHelper::call_function_async(WifiHelper::FunctionId::GetConnectedAps, on_done);
if (!r) { /* 提交失败 */ }
```

> 同一服务的连续 async 调用按提交顺序在服务内部串行执行。

### 4. 订阅事件（RAII 连接）

```cpp
// 事件 ScanApInfosUpdated 的 item: ApInfos(Array)
auto on_scan = [](const std::string & event_name, const boost::json::array & ap_infos) {
    std::vector<WifiHelper::ScanApInfo> scanned_aps;
    BROOKESIA_DESCRIBE_FROM_JSON(ap_infos, scanned_aps);
    BROOKESIA_LOGI("Scanned %1% APs", scanned_aps.size());
};
auto conn = WifiHelper::subscribe_event(WifiHelper::EventId::ScanApInfosUpdated, on_scan);
if (!conn.connected()) { /* 订阅失败 */ }
// conn 析构即取消订阅；需长期保活则存入容器（见下）
```

长期保活（chatbot 示例做法）：

```cpp
std::vector<service::EventRegistry::SignalConnection> global_connections;
global_connections.push_back(std::move(conn));
```

### 5. EventMonitor 阻塞等待

```cpp
// GeneralEventHappened schema: Event(String), IsUnexpected(Boolean)
using GeneralEventMonitor = WifiHelper::EventMonitor<WifiHelper::EventId::GeneralEventHappened>;
GeneralEventMonitor monitor;
if (!monitor.start()) { /* 启动失败 */ }

// 触发会引发该事件的动作
WifiHelper::call_function_async(WifiHelper::FunctionId::TriggerGeneralAction,
                                BROOKESIA_DESCRIBE_TO_STR(WifiHelper::GeneralAction::Start));

// 等待匹配 item 的事件
bool got = monitor.wait_for(std::vector<service::EventItem>{
    BROOKESIA_DESCRIBE_TO_STR(WifiHelper::GeneralEvent::Started), false
}, 5000);
if (!got) { /* 超时 */ }

monitor.stop();
monitor.clear();
```

`wait_for_any(timeout_ms)` 等任意一次；`get_last<T>()` 返回 `std::optional<std::tuple<T>>`，用 `std::get<0>` 取值。

## 常见错误

| 错误 | 原因 | 解决方法 |
|---|---|---|
| 同步调用返回错误 / 取不到返回值 | 未显式给返回类型模板参数 | Array 用 `<boost::json::array>`，Object 用 `<boost::json::object>` |
| 参数类型不匹配 | 直接传 enum 而非字符串 | 用 `BROOKESIA_DESCRIBE_TO_STR(Enum::Val)` |
| 回调不触发 | `subscribe_event` 在 `start()` 之前调用 | 先 `start()`，再订阅 |
| 订阅随即失效 | connection 是栈对象，出作用域析构 | 存入成员/全局容器 |
| EventMonitor 一直超时 | `wait_for` 的 item 类型/顺序与 schema 不符 | 核对事件 schema 后再构造 item |
| 同步调用卡死 | 用了默认超时且服务繁忙 | 传 `service::helper::Timeout(ms)` |

## 参考

- `examples/service/wifi/main/main.cpp` — 订阅、同步/异步、EventMonitor 的最完整范例
- `docs/en/service/usage.rst` — 应用开发指南（依赖→启动→调用→订阅→monitor）
- `docs/en/service/helper/base.rst` — Helper CRTP 基类
