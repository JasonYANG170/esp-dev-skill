# 处理 CHIP 设备事件（网络 / IP / 会话）

> **适用摘要**: 在 `CHIPDeviceManagerCallbacks::DeviceEventCallback` 中处理 `ChipDeviceEvent`：网络连接变化、IP 地址变化（需重启 mDNS）、安全会话建立等关键事件，确保设备配网后可被 commissioner 发现与控制。

> Version: selected repo/component version; align with the user project and dependency manifest.
> Evidence: `repos/connectedhomeip/resources/`, source/examples in `repos/connectedhomeip/`, and this recipe path `repos/connectedhomeip/recipes/device_event_handling.md`.
> Validation: draft metadata added from repository routing; verify APIs, Kconfig symbols, and component dependencies against the selected version.

## 触发意图

- "处理 CHIP 设备事件"
- "kInternetConnectivityChange"
- "kInterfaceIpAddressChanged"
- "kSessionEstablished"
- "Mdns::StartServer"

## 前置条件

| 条件 | 要求 |
|---|---|
| 头文件 | `<platform/CHIPDeviceLayer.h>`、`<app/server/Mdns.h>` |
| 回调 | `DeviceEventCallback(const ChipDeviceEvent *, intptr_t)` |
| 参考文件 | `examples/lock-app/esp32/main/DeviceCallbacks.cpp` |

## 分步说明

### 1. 重写 `DeviceEventCallback` 并按 Type 分发

```cpp
#include <platform/CHIPDeviceLayer.h>
#include <app/server/Mdns.h>

using namespace chip;
using namespace chip::DeviceLayer;

void DeviceCallbacks::DeviceEventCallback(const ChipDeviceEvent * event, intptr_t arg)
{
    switch (event->Type) {
    case DeviceEventType::kInternetConnectivityChange:
        OnInternetConnectivityChange(event);
        break;
    case DeviceEventType::kSessionEstablished:
        OnSessionEstablished(event);
        break;
    case DeviceEventType::kInterfaceIpAddressChanged:
        if (event->InterfaceIpAddressChanged.Type == InterfaceIpChangeType::kIpV4_Assigned ||
            event->InterfaceIpAddressChanged.Type == InterfaceIpChangeType::kIpV6_Assigned) {
            chip::app::Mdns::StartServer();
        }
        break;
    }
}
```

### 2. 处理网络连接变化（IPv4/IPv6）

```cpp
void DeviceCallbacks::OnInternetConnectivityChange(const ChipDeviceEvent * event)
{
    if (event->InternetConnectivityChange.IPv4 == kConnectivity_Established) {
        ESP_LOGI(TAG, "Server ready at: %s:%d", event->InternetConnectivityChange.address, CHIP_PORT);
        chip::app::Mdns::StartServer();
    } else if (event->InternetConnectivityChange.IPv4 == kConnectivity_Lost) {
        ESP_LOGE(TAG, "Lost IPv4 connectivity...");
    }
    if (event->InternetConnectivityChange.IPv6 == kConnectivity_Established) {
        ESP_LOGI(TAG, "IPv6 Server ready...");
        chip::app::Mdns::StartServer();
    } else if (event->InternetConnectivityChange.IPv6 == kConnectivity_Lost) {
        ESP_LOGE(TAG, "Lost IPv6 connectivity...");
    }
}
```

### 3. 检测 Commissioner 会话

```cpp
void DeviceCallbacks::OnSessionEstablished(const ChipDeviceEvent * event)
{
    if (event->SessionEstablished.IsCommissioner) {
        ESP_LOGI(TAG, "Commissioner detected!");
    }
}
```

### 4. 应用任务中安全查询连接状态（加锁）

```cpp
// 在独立任务循环中查询 CHIP 栈状态必须加锁
if (PlatformMgr().TryLockChipStack()) {
    sHaveBLEConnections      = (ConnectivityMgr().NumBLEConnections() != 0);
    sHaveServiceConnectivity = ConnectivityMgr().HaveServiceConnectivity();
    PlatformMgr().UnlockChipStack();
}
```

## 关键事件类型速查

| 事件 Type | 含义 | 典型处理 |
|---|---|---|
| `kInternetConnectivityChange` | 互联网连接建立/丢失 | 启停 mDNS |
| `kInterfaceIpAddressChanged` | 接口 IP（v4/v6）分配/释放 | `Mdns::StartServer()` 刷新监听 |
| `kSessionEstablished` | 安全会话建立（含 Commissioner 标志） | 记录 commissioner 到达 |
| `kServiceProvisioningChange` | 服务配置变化 | 视业务需要 |
| `kCHIPoBLEConnectionEstablished` | BLE 连接建立 | 配网阶段 |

## 常见错误

| 错误 | 原因 | 解决方法 |
|---|---|---|
| 拿到 IP 后 commissioner 找不到设备 | 未在 IP 变化时重启 mDNS | 在 `kInterfaceIpAddressChanged` 调 `chip::app::Mdns::StartServer()` |
| 任务循环读栈状态崩溃/数据竞争 | 未加 CHIP 栈锁 | 用 `PlatformMgr().TryLockChipStack()/UnlockChipStack()` |
| `CHIP_PORT` 未定义 | 未包含核心头 | `#include <core/CHIPCore.h>`（提供 `CHIP_PORT`） |
| 事件 Type 未识别 | 拼错枚举名 | 用 `DeviceEventType::kInternetConnectivityChange` 等 |
| `Mdns::StartServer` 未声明 | 缺头文件 | `#include <app/server/Mdns.h>` |

## 参考

- `examples/lock-app/esp32/main/DeviceCallbacks.cpp` — `DeviceEventCallback` 完整实现
- `examples/lock-app/esp32/main/AppTask.cpp` — `AppTaskMain` 中加锁查询连接状态
- `recipes/onoff_cluster_hardware.md` — 属性变化回调
