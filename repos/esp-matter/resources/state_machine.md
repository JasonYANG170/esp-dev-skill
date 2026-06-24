# Matter 节点生命周期 / 状态

> esp-matter 没有显式的"状态机枚举"，但 Matter 设备从启动到入网、被控制、解入网的全过程由 `esp_matter::start(callback)` 注册的事件回调（`event_callback_t`，参数 `const ChipDeviceEvent *event`）驱动。下面的事件类型取自 `components/esp_matter/esp_matter.h` 的 `DeviceEventType` 枚举与 `examples/light/main/app_main.cpp` 的 `app_event_cb`。

## 启动 → 入网 → 受控事件流

```
app_main
  ├─ nvs_flash_init()
  ├─ driver init
  ├─ node::create() + endpoint::create()
  ├─ esp_matter::start(app_event_cb)   ← Matter 线程启动，开始 DNS-SD/BLE 广告
  │
  └─ 事件回调陆续收到：
      kInterfaceIpAddressChanged        设备拿到 IP（Wi-Fi 关联成功 / Thread 附着）
          ↓
      kCommissioningSessionStarted      commissioner 发起 PASE 会话
          ↓
      kCommissioningSessionStopped      PASE 会话结束
          ↓
      kCommissioningWindowOpened / kCommissioningWindowClosed
          ↓
      kCommissioningComplete            入网完成（NOC 已签发，operational 网络可用）
          ↓
      kFabricCommitted / kFabricUpdated / kFabricWillBeRemoved / kFabricRemoved
          ↓
      kBLEDeinitialized                 （可选）BLE 内存释放
```

## 关键事件含义（来自 `esp_matter.h` `DeviceEventType`）

| 事件 | 含义 |
|---|---|
| `kInterfaceIpAddressChanged` | 网络接口 IP 变化（获得/失去 IP） |
| `kCommissioningSessionStarted` | 入网 PASE 会话开始 |
| `kCommissioningSessionStopped` | 入网 PASE 会话停止 |
| `kCommissioningWindowOpened` | 入网窗口打开（可被 commissioner 发现） |
| `kCommissioningWindowClosed` | 入网窗口关闭 |
| `kCommissioningComplete` | 入网完成 |
| `kFailSafeTimerExpired` | 入网失败（fail-safe 超时） |
| `kFabricCommitted` | fabric 提交（NOC 写入） |
| `kFabricUpdated` | fabric 更新 |
| `kFabricWillBeRemoved` | fabric 即将被删 |
| `kFabricRemoved` | fabric 已删 |
| `kBLEDeinitialized` | BLE 协议栈释放（内存回收） |

## Fabric 移除后重新打开入网窗口

`examples/light/main/app_main.cpp` 在 `kFabricRemoved` 且无残留 fabric 时，重新打开基础入网窗口（DNS-SD only）：

```cpp
case chip::DeviceLayer::DeviceEventType::kFabricRemoved: {
    if (chip::Server::GetInstance().GetFabricTable().FabricCount() == 0) {
        auto &cm = chip::Server::GetInstance().GetCommissioningWindowManager();
        constexpr auto kTimeout = chip::System::Clock::Seconds16(300);
        if (!cm.IsCommissioningWindowOpen()) {
            cm.OpenBasicCommissioningWindow(
                kTimeout, chip::CommissioningWindowAdvertisement::kDnssdOnly);
        }
    }
    break;
}
```

## 回调签名

```cpp
// Header: components/esp_matter/esp_matter_core.h
typedef void (*event_callback_t)(const ChipDeviceEvent *event, intptr_t arg);

// 注册
esp_err_t esp_matter::start(event_callback_t callback, intptr_t callback_arg = 0);
```

## 控制端（chip-tool）对应状态

从 commissioner 视角，设备经历：`pairing ble-wifi/ble-thread`（建立 PASE → 配网 → 签 NOC）→ 进入 operational fabric → 之后 `onoff/levelcontrol/...` cluster 命令走 CASE 加密通道。chip-tool 用 `interactive start` 时复用 CASE 会话，命令更快。
