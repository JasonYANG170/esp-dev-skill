# 门锁集群服务端集成（DoorLock server：LockState / PIN / RFID）

> **适用摘要**: 集成真正的 `DoorLock` 集群（`ZCL_DOOR_LOCK_CLUSTER_ID = 0x0101`）服务端，而不是把锁逻辑挂到 OnOff→GPIO。涵盖 `LockState` 属性写入（`EMBER_ZCL_DOOR_LOCK_STATE_LOCKED/UNLOCKED`）、`emberAfPluginDoorLockServerActivateDoorLockCallback` 动作回调、PIN/RFID 校验（`ApplyPin/ApplyRfid`）、用户表与日志。这是真实 Matter 门锁设备类型的入口；lock-app/esp32 里的 OnOff→继电器映射只是简化版。

> Version: selected repo/component version; align with the user project and dependency manifest.
> Evidence: `repos/connectedhomeip/resources/`, source/examples in `repos/connectedhomeip/`, and this recipe path `repos/connectedhomeip/recipes/door_lock_cluster.md`.
> Validation: draft metadata added from repository routing; verify APIs, Kconfig symbols, and component dependencies against the selected version.

## 触发意图

- "门锁集群"
- "DoorLock cluster server"
- "LockState 属性"
- "emberAfPluginDoorLockServerActivateDoorLockCallback"
- "PIN / RFID 开锁"
- "DOOR_LOCK_SERVER_ENDPOINT"

## 前置条件

| 条件 | 要求 |
|---|---|
| 参考示例 | `examples/all-clusters-app/esp32/`（演示 DoorLock 属性写入）、`examples/lock-app/esp32/`（OnOff→锁动作的简化映射） |
| 服务端头文件 | `<app/clusters/door-lock-server/door-lock-server.h>` |
| 生成宏 | `<app/common/gen/cluster-id.h>`（`ZCL_DOOR_LOCK_CLUSTER_ID = 0x0101`）、`<app/common/gen/attribute-id.h>`（`ZCL_LOCK_STATE_ATTRIBUTE_ID = 0x0000`）、`<app/common/gen/enums.h>`（`EMBER_ZCL_DOOR_LOCK_STATE_LOCKED = 1`、`..._UNLOCKED = 2`） |
| 默认 endpoint | `DOOR_LOCK_SERVER_ENDPOINT`（默认 `1`，见 door-lock-server.h） |
| 主机验证 | `examples/chip-tool/`，`doorlock` 集群命令集 |

## 分步说明

### 1. 写入 LockState 属性（本地状态变化时）

`LockState`（属性 ID `0x0000`）是门锁的核心状态。本地动作（按钮、定时、传感器）完成后，用 `emberAfWriteServerAttribute` 把枚举值写回 server：

```cpp
#include <app/clusters/door-lock-server/door-lock-server.h>
#include <app/common/gen/cluster-id.h>
#include <app/common/gen/attribute-id.h>
#include <app/common/gen/enums.h>
#include <app/util/af.h>

// all-clusters-app 在 SetupPretendDevices() 初始化时写 UNLOCKED：
void SetDoorLockUnlocked()
{
    uint8_t attributeValue = EMBER_ZCL_DOOR_LOCK_STATE_UNLOCKED;  // = 2
    emberAfWriteServerAttribute(DOOR_LOCK_SERVER_ENDPOINT,        // 默认 endpoint 1
                                ZCL_DOOR_LOCK_CLUSTER_ID,
                                ZCL_LOCK_STATE_ATTRIBUTE_ID,
                                &attributeValue, ZCL_INT8U_ATTRIBUTE_TYPE);
}
```

M5Stack 屏幕上 "Open"/"Closed" 切换时也走同一调用（`all-clusters-app/main/main.cpp::EditAttributeListModel::ItemAction`）：

```cpp
if (name == "State" && cluster == "Lock") {
    uint8_t attributeValue = (value == "Closed")
        ? EMBER_ZCL_DOOR_LOCK_STATE_LOCKED      // = 1
        : EMBER_ZCL_DOOR_LOCK_STATE_UNLOCKED;   // = 2
    emberAfWriteServerAttribute(DOOR_LOCK_SERVER_ENDPOINT, ZCL_DOOR_LOCK_CLUSTER_ID,
                                ZCL_LOCK_STATE_ATTRIBUTE_ID,
                                &attributeValue, ZCL_INT8U_ATTRIBUTE_TYPE);
}
```

### 2. 实现 ActivateDoorLock 回调（接收锁门/开门命令）

当 commissioner 下发 `LockDoor` / `UnlockDoor` 命令时，server 插件调用应用提供的弱符号回调。实现它以驱动物理锁机构（电机/继电器）：

```cpp
// src/app/clusters/door-lock-server/door-lock-server.h
bool emberAfPluginDoorLockServerActivateDoorLockCallback(bool activate);
//   activate=true  -> 移动到锁定位置
//   activate=false -> 移动到解锁位置
//   返回 true 表示成功激活
```

应用侧实现示例：

```cpp
#include <app/clusters/door-lock-server/door-lock-server.h>

bool emberAfPluginDoorLockServerActivateDoorLockCallback(bool activate)
{
    if (activate) {
        ESP_LOGI(TAG, "DoorLock: LOCK command received");
        // 驱动电机/继电器到锁定位置
    } else {
        ESP_LOGI(TAG, "DoorLock: UNLOCK command received");
        // 驱动电机/继电器到解锁位置
    }
    return true;  // 返回 false 会向 commissioner 报告失败
}
```

> 注意：这与 `recipes/onoff_cluster_hardware.md` 的 `PostAttributeChangeCallback` 路径不同。DoorLock server 有自己的命令分发，`LockDoor`/`UnlockDoor` 不会作为 OnOff 属性变化出现。

### 3. PIN / RFID 校验（可选，真实门锁常用）

server 头文件提供 PIN/RFID 应用入口与用户表初始化：

```cpp
// 尝试用 PIN 开锁；内部会查用户表并触发 ActivateDoorLockCallback
EmberAfStatus emberAfPluginDoorLockServerApplyPin(uint8_t * pin, uint8_t pinLength);
// 尝试用 RFID 开锁
EmberAfStatus emberAfPluginDoorLockServerApplyRfid(uint8_t * rfid, uint8_t rfidLength);

// 启动时初始化用户表 / 调度表（由 server 插件内部调用）
void emAfPluginDoorLockServerInitUser(void);
void emAfPluginDoorLockServerInitSchedule(void);
bool emAfPluginDoorLockServerSetPinUserType(uint16_t userId, EmberAfDoorLockUserType type);
```

用户表容量由宏控制（默认值见头文件）：

| 宏 | 默认 | 含义 |
|---|---|---|
| `EMBER_AF_PLUGIN_DOOR_LOCK_SERVER_PIN_USER_TABLE_SIZE` | 8 | 支持的 PIN 用户数 |
| `EMBER_AF_PLUGIN_DOOR_LOCK_SERVER_RFID_USER_TABLE_SIZE` | 8 | 支持的 RFID 用户数 |
| `EMBER_AF_PLUGIN_DOOR_LOCK_SERVER_MAX_PIN_LENGTH` | 8 | 最大 PIN 长度 |
| `EMBER_AF_PLUGIN_DOOR_LOCK_SERVER_MAX_RFID_LENGTH` | 8 | 最大 RFID 长度 |
| `EMBER_AF_PLUGIN_DOOR_LOCK_SERVER_MAX_LOG_ENTRIES` | 16 | 日志条目数 |

调用 `ApplyPin` 触发校验：

```cpp
uint8_t pin[] = { '1','2','3','4' };
EmberAfStatus s = emberAfPluginDoorLockServerApplyPin(pin, sizeof(pin));
// 校验通过则自动调用 ActivateDoorLockCallback(false) 解锁
```

### 4. 事件日志（AddLogEntry / GetLogEntry）

门锁事件（开锁/编程）可记录到环形缓冲，供 commissioner 通过 `GetLogRecord` ZCL 命令查询：

```cpp
bool emberAfPluginDoorLockServerAddLogEntry(EmberAfDoorLockEventType eventType,
                                            EmberAfDoorLockEventSource source,
                                            uint8_t eventId,        // EmberAfDoorLockOperationEventCode 或 ProgrammingEventCode
                                            uint16_t userId,
                                            uint8_t pinLength, uint8_t * pin);
bool emberAfPluginDoorLockServerGetLogEntry(uint16_t * entryId,
                                            EmberAfPluginDoorLockServerLogEntry * entry);
```

门状态变化时通知 server（更新 `DoorState` 属性）：

```cpp
EmberAfStatus emAfPluginDoorLockServerNoteDoorStateChanged(EmberAfDoorState state);
```

### 5. 用 chip-tool 验证

`chip-tool` 的 `doorlock` 集群命令集非常完整（lock/unlock/set-pin/set-rfid/set-user-type/schedules 等）：

```bash
# 配网后（默认 disc 3840 / pin 20202021）
chip-tool doorlock
# Commands:
#   lock-door / unlock-door / unlock-with-timeout
#   set-pin / get-pin / clear-pin / clear-all-pins
#   set-rfid / get-rfid / clear-rfid / clear-all-rfids
#   set-user-type / get-user-type
#   set-weekday-schedule / clear-weekday-schedule / get-weekday-schedule
#   set-yearday-schedule / clear-yearday-schedule / get-yearday-schedule
#   set-holiday-schedule / clear-holiday-schedule / get-holiday-schedule
#   discover / read / report

# 锁门 / 开门（endpoint 1）
chip-tool doorlock lock-door 1
chip-tool doorlock unlock-door 1

# 设置一个 PIN 用户
chip-tool doorlock set-pin 1   # 按 chip-tool 提示传参
```

## 常见错误

| 错误 | 原因 | 解决方法 |
|---|---|---|
| `LockDoor` 命令发了但锁不动 | 未实现 `emberAfPluginDoorLockServerActivateDoorLockCallback`（弱符号返回默认） | 在应用中实现该回调并驱动物理机构 |
| 误把 DoorLock 当 OnOff 处理 | 用了 lock-app 的 `PostAttributeChangeCallback` + `ZCL_ON_OFF_CLUSTER_ID` 分支 | DoorLock 命令走独立 server，用 `ActivateDoorLockCallback`，属性写用 `ZCL_DOOR_LOCK_CLUSTER_ID` |
| `emberAfWriteServerAttribute` 报非 SUCCESS | endpoint 不是 `DOOR_LOCK_SERVER_ENDPOINT` 或未配置 DoorLock server | 确认 `.zap` 中该 endpoint 含 DoorLock server；用 `DOOR_LOCK_SERVER_ENDPOINT` 宏 |
| PIN 校验总失败 | 用户表未初始化或 PIN 长度超 `MAX_PIN_LENGTH` | 调 `emAfPluginDoorLockServerInitUser()`；检查 `EMBER_AF_PLUGIN_DOOR_LOCK_SERVER_MAX_PIN_LENGTH` |
| chip-tool `set-pin` 参数错 | 未先 `chip-tool doorlock set-pin` 看参数列表 | 先 `chip-tool doorlock <cmd>` 列参数 |

## 参考项目

- `examples/all-clusters-app/esp32/main/main.cpp` — `SetupPretendDevices()` 与 `EditAttributeListModel::ItemAction()` 演示 `ZCL_DOOR_LOCK_CLUSTER_ID` / `ZCL_LOCK_STATE_ATTRIBUTE_ID` 写入
- `src/app/clusters/door-lock-server/door-lock-server.h` — `ActivateDoorLockCallback`、`ApplyPin/ApplyRfid`、`InitUser`、`AddLogEntry`、`NoteDoorChanged`、`DOOR_LOCK_SERVER_ENDPOINT`、表容量宏
- `examples/lock-app/esp32/` — OnOff→锁动作简化映射（对比，非 DoorLock server）
- `examples/chip-tool/README.md` — `doorlock` 集群完整命令集
