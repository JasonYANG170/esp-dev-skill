# connectedhomeip — ESP32 API Quick Reference

> 所有签名均来自仓库 `examples/` 与 `src/` 真实代码。模块按 ESP32 设备应用开发顺序分组。

## CHIPDeviceManager（设备管理单例）

来源：`examples/lock-app/esp32/main/include/CHIPDeviceManager.h`

```cpp
namespace chip::DeviceManager {

class CHIPDeviceManagerCallbacks {
public:
    virtual void DeviceEventCallback(const chip::DeviceLayer::ChipDeviceEvent * event, intptr_t arg);
    virtual void PostAttributeChangeCallback(chip::EndpointId endpoint, chip::ClusterId clusterId,
                                             chip::AttributeId attributeId, uint8_t mask,
                                             uint16_t manufacturerCode, uint8_t type,
                                             uint16_t size, uint8_t * value) {}
    virtual ~CHIPDeviceManagerCallbacks() {}
};

class CHIPDeviceManager {
public:
    static CHIPDeviceManager & GetInstance();
    CHIP_ERROR Init(CHIPDeviceManagerCallbacks * cb);
    CHIPDeviceManagerCallbacks * GetCHIPDeviceManagerCallbacks();
    static void CommonDeviceEventHandler(const chip::DeviceLayer::ChipDeviceEvent * event, intptr_t arg);
};

} // namespace chip::DeviceManager
```

典型用法：
```cpp
CHIPDeviceManager & mgr = CHIPDeviceManager::GetInstance();
mgr.Init(&myCallbacks);
```

## Server（Matter 服务层）

来源：`src/app/server/Server.h`

```cpp
// 全局命名空间（非 chip::app 命名空间内，声明在 app/server/Server.h）
void InitServer(AppDelegate * delegate = nullptr);
```

`AppDelegate` 定义于 `src/app/server/AppDelegate.h`；设备示例中常以 `nullptr` 或自定义 callbacks 调用：
```cpp
InitServer();                 // lock-app
InitServer(&callbacks);       // all-clusters-app
```

## PlatformMgr / ConnectivityMgr / ConfigurationMgr（设备层）

来源：`<platform/CHIPDeviceLayer.h>`（`src/platform/ESP32/`）

```cpp
namespace chip::DeviceLayer {

PlatformMgr()     // 单例访问器
  .TryLockChipStack()   -> bool  // 非阻塞加锁，独立任务查询栈状态时使用
  .LockChipStack()      -> void
  .UnlockChipStack()    -> void
  .RunEventLoop()       -> void

ConnectivityMgr()  // 单例访问器
  .NumBLEConnections()              -> uint16_t
  .HaveServiceConnectivity()        -> bool
  .IsThreadProvisioned()            -> bool
  .IsBLEAdvertisingEnabled()        -> bool
  .SetBLEAdvertisingEnabled(bool)
  .SetBLEAdvertisingMode(AdvertisingMode)  // kFastAdvertising 等
  .kFastAdvertising

ConfigurationMgr() // 单例访问器
  .LogDeviceConfig()
  .InitiateFactoryReset()

} // namespace chip::DeviceLayer
```

> 设备事件通过 `ChipDeviceEvent`（同一头文件族）在 `CHIPDeviceManagerCallbacks::DeviceEventCallback` 中接收。

## mDNS 服务

来源：`src/app/server/Mdns.h`

```cpp
namespace chip::app::Mdns {
void StartServer();
} // namespace chip::app::Mdns
```
典型场景：接口 IP 分配 / 互联网连接建立后调用 `chip::app::Mdns::StartServer()`。

## Onboarding Codes（配网码 / QR）

来源：`src/app/server/OnboardingCodesUtil.h`

```cpp
void PrintOnboardingCodes(chip::RendezvousInformationFlag rendezvousFlags);
```
示例用法：
```cpp
PrintOnboardingCodes(chip::RendezvousInformationFlag(chip::RendezvousInformationFlag::kBLE));
```

## ZCL / Ember 应用框架（服务端属性访问）

来源：`<app/common/gen/cluster-id.h>`、`<app/common/gen/attribute-id.h>`、`<app/common/gen/attribute-type.h>`、`<app/common/gen/enums.h>`、`<app/util/af-enums.h>`、`<app/util/attribute-storage.h>`

```cpp
// 写服务端属性（带 mask）
EmberAfStatus emberAfWriteAttribute(uint8_t endpoint, uint32_t clusterId, uint16_t attributeId,
                                    uint8_t mask, uint8_t * value, uint8_t type);
// 写服务端属性（便捷封装，等价于 mask=CLUSTER_MASK_SERVER）
EmberAfStatus emberAfWriteServerAttribute(uint8_t endpoint, uint32_t clusterId, uint16_t attributeId,
                                          uint8_t * value, uint8_t type);
// 读服务端属性
EmberAfStatus emberAfReadAttribute(uint8_t endpoint, uint32_t clusterId, uint16_t attributeId,
                                   uint8_t mask, uint8_t * buffer, uint16_t bufferLength, uint8_t type);

// 常用集群 / 属性宏
ZCL_ON_OFF_CLUSTER_ID            // 0x0006
ZCL_LEVEL_CONTROL_CLUSTER_ID     // 0x0008
ZCL_DOOR_LOCK_CLUSTER_ID         // 0x0101
ZCL_TEMP_MEASUREMENT_CLUSTER_ID  // 0x0402

ZCL_ON_OFF_ATTRIBUTE_ID          // 0x0000
ZCL_CURRENT_LEVEL_ATTRIBUTE_ID   // 0x0000  (Level Control)
ZCL_LOCK_STATE_ATTRIBUTE_ID      // 0x0000  (Door Lock)
ZCL_TEMP_MEASURED_VALUE_ATTRIBUTE_ID  // 0x0000  (TemperatureMeasurement)

// 类型宏
ZCL_BOOLEAN_ATTRIBUTE_TYPE
ZCL_INT8U_ATTRIBUTE_TYPE         // uint8_t（Level/DoorLock state 用）

// 枚举（Door Lock state，enums.h）
EMBER_ZCL_DOOR_LOCK_STATE_LOCKED    // = 1
EMBER_ZCL_DOOR_LOCK_STATE_UNLOCKED  // = 2

CLUSTER_MASK_SERVER
EMBER_ZCL_STATUS_SUCCESS
EMBER_ZCL_STATUS_INVALID_VALUE

// VerifyOrExit 宏
#include <support/CodeUtils.h>
VerifyOrExit(cond, onFailureAction);
```

典型回写（OnOff / DoorLock / Level 通用模式）：
```cpp
uint8_t v = 1;
EmberAfStatus status = emberAfWriteAttribute(1, ZCL_ON_OFF_CLUSTER_ID, ZCL_ON_OFF_ATTRIBUTE_ID,
                                             CLUSTER_MASK_SERVER, &v, ZCL_BOOLEAN_ATTRIBUTE_TYPE);

// DoorLock state（见 all-clusters-app/main/main.cpp::SetupPretendDevices）
uint8_t lockState = EMBER_ZCL_DOOR_LOCK_STATE_UNLOCKED;
emberAfWriteServerAttribute(DOOR_LOCK_SERVER_ENDPOINT, ZCL_DOOR_LOCK_CLUSTER_ID,
                            ZCL_LOCK_STATE_ATTRIBUTE_ID, &lockState, ZCL_INT8U_ATTRIBUTE_TYPE);

// Level Control current level（见 all-clusters-app/main/main.cpp::SetupInitialLevelControlValues）
uint8_t level = UINT8_MAX;
emberAfWriteAttribute(1, ZCL_LEVEL_CONTROL_CLUSTER_ID, ZCL_CURRENT_LEVEL_ATTRIBUTE_ID,
                      CLUSTER_MASK_SERVER, &level, ZCL_INT8U_ATTRIBUTE_TYPE);
```

## TemperatureMeasurement 集群服务端

来源：`src/app/clusters/temperature-measurement-server/temperature-measurement-server.h`

```cpp
// measuredValue 单位为 0.01 °C（int16_t，有符号，支持负温）
EmberAfStatus emberAfTemperatureMeasurementClusterSetMeasuredValueCallback(chip::EndpointId endpoint, int16_t measuredValue);
EmberAfStatus emberAfTemperatureMeasurementClusterSetMinMeasuredValueCallback(chip::EndpointId endpoint, int16_t minMeasuredValue);
EmberAfStatus emberAfTemperatureMeasurementClusterSetMaxMeasuredValueCallback(chip::EndpointId endpoint, int16_t maxMeasuredValue);

EmberAfStatus emberAfTemperatureMeasurementClusterGetMeasuredValue(chip::EndpointId endpoint, int16_t * measuredValue);
EmberAfStatus emberAfTemperatureMeasurementClusterGetMinMeasuredValue(chip::EndpointId endpoint, int16_t * minMeasuredValue);
EmberAfStatus emberAfTemperatureMeasurementClusterGetMaxMeasuredValue(chip::EndpointId endpoint, int16_t * maxMeasuredValue);
```

典型上报（sensor 模式，与 actuator 的 PostAttributeChangeCallback 相对）：
```cpp
// 21.00 °C -> 2100
emberAfTemperatureMeasurementClusterSetMeasuredValueCallback(1, static_cast<int16_t>(21 * 100));
```

## DoorLock 集群服务端

来源：`src/app/clusters/door-lock-server/door-lock-server.h`

```cpp
// 命令入口回调：LockDoor/UnlockDoor 命令到达时由 server 插件调用
bool emberAfPluginDoorLockServerActivateDoorLockCallback(bool activate);  // true=锁定, false=解锁

// PIN / RFID 校验（自动触发 ActivateDoorLockCallback）
EmberAfStatus emberAfPluginDoorLockServerApplyPin(uint8_t * pin, uint8_t pinLength);
EmberAfStatus emberAfPluginDoorLockServerApplyRfid(uint8_t * rfid, uint8_t rfidLength);

// 用户表 / 调度表
void emAfPluginDoorLockServerInitUser(void);
void emAfPluginDoorLockServerInitSchedule(void);
bool emAfPluginDoorLockServerSetPinUserType(uint16_t userId, EmberAfDoorLockUserType type);

// 日志
bool emberAfPluginDoorLockServerAddLogEntry(EmberAfDoorLockEventType eventType, EmberAfDoorLockEventSource source,
                                            uint8_t eventId, uint16_t userId, uint8_t pinLength, uint8_t * pin);
bool emberAfPluginDoorLockServerGetLogEntry(uint16_t * entryId, EmberAfPluginDoorLockServerLogEntry * entry);

// 门状态变化通知
EmberAfStatus emAfPluginDoorLockServerNoteDoorStateChanged(EmberAfDoorState state);

// 默认 endpoint（可被 DOOR_LOCK_SERVER_ENDPOINT 宏覆盖）
#ifndef DOOR_LOCK_SERVER_ENDPOINT
#define DOOR_LOCK_SERVER_ENDPOINT 1
#endif

// 表容量宏（默认值）
EMBER_AF_PLUGIN_DOOR_LOCK_SERVER_PIN_USER_TABLE_SIZE        // 8
EMBER_AF_PLUGIN_DOOR_LOCK_SERVER_RFID_USER_TABLE_SIZE       // 8
EMBER_AF_PLUGIN_DOOR_LOCK_SERVER_MAX_PIN_LENGTH             // 8
EMBER_AF_PLUGIN_DOOR_LOCK_SERVER_MAX_RFID_LENGTH            // 8
EMBER_AF_PLUGIN_DOOR_LOCK_SERVER_MAX_LOG_ENTRIES            // 16
```

## Level Control 集群服务端

来源：`src/app/clusters/level-control/level-control.h`

```cpp
// 启动时 / endpoint OnOff 状态解析后调用，用于同步硬件（PWM 等）到当前 level
void emberAfPluginLevelControlClusterServerPostInitCallback(chip::EndpointId endpoint);

// tick 频率（编译期可重定义，默认 32 Hz）
#ifndef EMBER_AF_PLUGIN_LEVEL_CONTROL_TICKS_PER_SECOND
#define EMBER_AF_PLUGIN_LEVEL_CONTROL_TICKS_PER_SECOND 32
#endif
```

## CHIP Shell（设备端 CLI）

来源：`examples/platform/esp32/shell_extension/launch.h`、`launch.cpp`、`<lib/shell/Engine.h>`

```cpp
namespace chip {
void LaunchShell();   // 创建 "chip_cli" 任务（栈 2048，优先级 5）运行 Engine::Root().RunMainLoop()
}
```

调用点（`app_main`，需 `CONFIG_ENABLE_CHIP_SHELL=y`）：
```cpp
#if CONFIG_ENABLE_CHIP_SHELL
    chip::LaunchShell();
#endif
```

shell 命令（见 `examples/shell/README*.md`）：`help`、`device config`、`device get <param>`、`device start`、`echo`、`version`、`rand`、`base64 encode|decode`、`ping`、`exit`、`otcli`（启用 Thread 时）。

## Pigweed RPC（主机 ↔ 设备 RPC 通道）

来源：`examples/common/pigweed/RpcService.h`、`examples/pigweed-app/esp32/main/main.cpp`

```cpp
namespace chip { namespace rpc {
class Mutex {
public:
    virtual void Lock() = 0;
    virtual void Unlock() = 0;
    virtual ~Mutex() {}
};
void Start(void (*RegisterServices)(pw::rpc::Server &), ::chip::rpc::Mutex * uart_mutex_);
}}
```

设备侧注册 EchoService：
```cpp
pw::rpc::EchoService echo_service;
void RegisterServices(pw::rpc::Server & server) { server.RegisterService(echo_service); }
void RunRpcService(void *) {
    ::chip::rpc::Start(RegisterServices, &::chip::rpc::logger_mutex);
}
```

主机侧（需 `CONFIG_ENABLE_PW_RPC=y` 的设备 + CHIP 主机环境）：
```bash
python -m pw_hdlc.rpc_console --device /dev/ttyUSB0 -b 115200 \
    $CHIP_ROOT/third_party/pigweed/repo/pw_rpc/pw_rpc_protos/echo.proto -o /tmp/pw_rpc.out
# 在 console 内：
rpcs.pw.rpc.EchoService.Echo(msg="hi")
```

## BoltLockManager（lock-app 设备逻辑模板）

来源：`examples/lock-app/esp32/main/include/BoltLockManager.h`

```cpp
class BoltLockManager {
public:
    enum Action_t { LOCK_ACTION = 0, UNLOCK_ACTION, INVALID_ACTION };
    enum State_t  { kState_LockingInitiated, kState_LockingCompleted,
                    kState_UnlockingInitiated, kState_UnlockingCompleted };

    int  Init();
    bool IsUnlocked();
    bool IsActionInProgress();
    bool InitiateAction(int32_t aActor, Action_t aAction);
    void EnableAutoRelock(bool aOn);
    void SetAutoLockDuration(uint32_t aDurationInSecs);

    typedef void (*Callback_fn_initiated)(Action_t, int32_t aActor);
    typedef void (*Callback_fn_completed)(Action_t);
    void SetCallbacks(Callback_fn_initiated, Callback_fn_completed);
};

BoltLockManager & BoltLockMgr(void);  // 单例访问
```

## AppTask / AppEvent（应用任务与事件）

来源：`examples/lock-app/esp32/main/include/AppTask.h`、`AppEvent.h`

```cpp
class AppTask {
public:
    int  StartAppTask();
    void PostEvent(const AppEvent * aEvent);
    void DispatchEvent(AppEvent * aEvent);
    void PostLockActionRequest(int32_t aActor, BoltLockManager::Action_t aAction);
    // ...ButtonEventHandler / FunctionHandler / LockActionEventHandler 等
};
AppTask & GetAppTask();  // 单例访问
```

`AppEvent` 事件类型（`AppEvent.h`）：`kEventType_Button`、`kEventType_Lock`、`kEventType_Timer`。

## ESP-IDF 集成入口

来源：`examples/lock-app/esp32/main/main.cpp`

```cpp
extern "C" void app_main();   // ESP-IDF 入口

// 典型初始化链
nvs_flash_init();
CHIPDeviceManager::GetInstance().Init(&callbacks);
InitServer();
GetAppTask().StartAppTask();
```

ESP-IDF API（来自 `esp_idf`，非 CHIP）：`nvs_flash_init`、`esp_wifi_*`、`esp_log.h`（`ESP_LOGI/ESP_LOGE`）。

## 日志辅助

来源：`<support/ErrorStr.h>`

```cpp
const char * ErrorStr(CHIP_ERROR err);   // 把 CHIP_ERROR 转为可读字符串
```
示例：`ESP_LOGE(TAG, "init failed: %s", ErrorStr(err));`
