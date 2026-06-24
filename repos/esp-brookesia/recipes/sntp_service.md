# SNTP 服务：网络时间同步

> **适用摘要**: 使用 `brookesia_service_sntp` 配置 NTP 服务器与时区，查询同步状态、服务器列表与时区。SNTP 可选依赖 NVS 做持久化，网络可用后自动同步。

## 触发意图

- "同步网络时间 NTP"
- "设置时区"
- "配置 NTP 服务器"
- "查询时间是否已同步"

## 前置条件

| 条件 | 要求 |
|---|---|
| 依赖 | `espressif/brookesia_service_sntp`（可选 `brookesia_service_nvs` 持久化、`brookesia_service_wifi` 提供网络）|
| 网络 | 需先连上 Wi-Fi（Agent 的 TimeSyncing 状态依赖 SNTP）|
| 参考示例 | `examples/agent/chatbot/main/modules/general_services.cpp`（`start_sntp()`）|

## 分步说明

### 1. 启动并绑定

```cpp
using SntpHelper = service::helper::SNTP;
auto &service_manager = service::ServiceManager::get_instance();
service_manager.start();
if (SntpHelper::is_available()) {
    auto binding = service_manager.bind(SntpHelper::get_name().data());
}
```

> 在 chatbot 中，`GeneralServices::start_sntp()` 即执行上述绑定；SNTP 与 NVS/Wi-Fi 一同启动。

### 2. 配置 NTP 服务器与时区（schema 参数为 Object/Array）

默认 NTP 服务器为 `"pool.ntp.org"`，默认时区 `CST-8`（UTC+8）。可通过对应 Function 设置（具体 FunctionId 见组件 Helper 头 `brookesia/service_helper/sntp.hpp` 与 `docs/en/service/sntp.rst` 契约）：

- 设置 NTP 服务器列表
- 设置时区（标准 zone 字符串，如 `UTC`、`CST-8`、`EST-5`）
- 启动/停止同步
- 查询同步状态、服务器列表、时区
- `ResetData` 恢复默认

### 3. 与 Agent 状态机联动

`AgentManager` 的状态机含可选的 `TimeSyncing` 状态：激活 Agent 前会先等待时间同步成功（或 Bypass 跳过）。因此 SNTP 服务通常在启动 Agent 之前绑定。

```cpp
GeneralServices::get_instance().start_nvs();
GeneralServices::get_instance().start_sntp();   // 绑定 SNTP
// ... 之后初始化并启动 Agent ...
```

## 常见错误

| 错误 | 原因 | 解决方法 |
|---|---|---|
| 不同步 | 未连 Wi-Fi | 先启动 Wi-Fi 服务并连网 |
| 时区不对 | 默认 CST-8 与目标不符 | 设置目标时区 zone 字符串 |
| Agent 卡在 TimeSyncing | SNTP 未绑定或未成功 | 确认 SNTP 服务已 bind；网络可用 |
| 配置丢失 | 未依赖 NVS | 加 `espressif/brookesia_service_nvs` 以持久化 |

## 参考

- `docs/en/service/sntp.rst` — SNTP 服务接口契约
- `examples/agent/chatbot/main/modules/general_services.cpp` — `start_sntp()` 集成方式
