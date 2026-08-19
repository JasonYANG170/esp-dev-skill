# 将 OnOff 集群属性绑定到 GPIO（LED / 继电器）

> **适用摘要**: 把 Matter OnOff 集群（`ZCL_ON_OFF_CLUSTER_ID`）的属性写入事件映射到物理 GPIO（LED 或继电器），并在本地状态变化时回写服务端属性。这是 lock-app 与 all-clusters-app 的核心模式。

> Version: selected repo/component version; align with the user project and dependency manifest.
> Evidence: `repos/connectedhomeip/resources/`, source/examples in `repos/connectedhomeip/`, and this recipe path `repos/connectedhomeip/recipes/onoff_cluster_hardware.md`.
> Validation: draft metadata added from repository routing; verify APIs, Kconfig symbols, and component dependencies against the selected version.

## 触发意图

- "OnOff 控制 LED"
- "集群属性驱动 GPIO"
- "继电器 Matter 控制"
- "PostAttributeChangeCallback 用法"
- "emberAfWriteAttribute"

## 前置条件

| 条件 | 要求 |
|---|---|
| 参考示例 | `examples/lock-app/esp32/` |
| 关键文件 | `main/DeviceCallbacks.cpp`、`main/AppTask.cpp`、`main/include/BoltLockManager.h` |
| 头文件 | `<app/common/gen/cluster-id.h>`、`<app/common/gen/attribute-id.h>` |

## 分步说明

### 1. 在回调中分发 OnOff 写入

`DeviceCallbacks::PostAttributeChangeCallback` 是属性变化的统一入口。按 clusterId 分发：

```cpp
#include <app/common/gen/cluster-id.h>
#include <app/common/gen/attribute-id.h>

void DeviceCallbacks::PostAttributeChangeCallback(EndpointId endpointId, ClusterId clusterId,
                                                  AttributeId attributeId, uint8_t mask,
                                                  uint16_t manufacturerCode, uint8_t type,
                                                  uint16_t size, uint8_t * value)
{
    switch (clusterId) {
    case ZCL_ON_OFF_CLUSTER_ID:
        OnOnOffPostAttributeChangeCallback(endpointId, attributeId, value);
        break;
    default:
        ESP_LOGI(TAG, "Unhandled cluster ID: %d", clusterId);
        break;
    }
}
```

### 2. 校验 endpoint/attribute 后驱动动作

```cpp
void DeviceCallbacks::OnOnOffPostAttributeChangeCallback(EndpointId endpointId, AttributeId attributeId, uint8_t * value)
{
    VerifyOrExit(attributeId == ZCL_ON_OFF_ATTRIBUTE_ID, ESP_LOGI(TAG, "Unhandled Attribute ID: '0x%04x", attributeId));
    VerifyOrExit(endpointId == 1 || endpointId == 2, ESP_LOGE(TAG, "Unexpected EndPoint ID: `0x%02x'", endpointId));

    if (*value) {
        BoltLockMgr().InitiateAction(AppEvent::kEventType_Lock, BoltLockManager::LOCK_ACTION);
    } else {
        BoltLockMgr().InitiateAction(AppEvent::kEventType_Lock, BoltLockManager::UNLOCK_ACTION);
    }
exit:
    return;
}
```

### 3. 本地状态变化时回写服务端属性

按钮或自动动作完成后，用 `emberAfWriteAttribute` 同步 OnOff 服务端属性：

```cpp
#include <app/common/gen/attribute-id.h>
#include <app/common/gen/attribute-type.h>
#include <app/common/gen/cluster-id.h>
#include <app/util/af-enums.h>
#include <app/util/attribute-storage.h>

void AppTask::UpdateClusterState(void)
{
    uint8_t newValue = !BoltLockMgr().IsUnlocked();

    EmberAfStatus status = emberAfWriteAttribute(
        1, ZCL_ON_OFF_CLUSTER_ID, ZCL_ON_OFF_ATTRIBUTE_ID,
        CLUSTER_MASK_SERVER, (uint8_t *) &newValue, ZCL_BOOLEAN_ATTRIBUTE_TYPE);

    if (status != EMBER_ZCL_STATUS_SUCCESS) {
        ESP_LOGI(TAG, "ERR: updating on/off %x", status);
    }
}
```

### 4. GPIO 初始化（状态 LED）

```cpp
#define STATUS_LED_GPIO_NUM GPIO_NUM_2   // AppConfig.h

LEDWidget sStatusLED;
sStatusLED.Init(SYSTEM_STATE_LED);       // LEDWidget 封装 GPIO + 闪烁
```

## 常见错误

| 错误 | 原因 | 解决方法 |
|---|---|---|
| 任何写入都触发动作 | 未校验 endpoint/attribute | 加 `VerifyOrExit(attributeId == ZCL_ON_OFF_ATTRIBUTE_ID)` |
| 回写后不生效 | type/cluster 宏用错 | 用 `ZCL_BOOLEAN_ATTRIBUTE_TYPE` + `ZCL_ON_OFF_*` 宏 |
| `emberAfWriteAttribute` 报非 EMBER_ZCL_STATUS_SUCCESS | endpoint 未实现 OnOff server | 确认该 endpoint 在 zap 配置中含 OnOff server |
| 本地按钮变状态但 controller 看不到 | 未调用 UpdateClusterState | 动作完成回调里调用 `UpdateClusterState()` |
| LED 不亮 | GPIO 与板子不符 | 查 `AppConfig.h` 的 `STATUS_LED_GPIO_NUM` / 板子原理图 |

## 参考

- `examples/lock-app/esp32/main/DeviceCallbacks.cpp` — `OnOnOffPostAttributeChangeCallback`
- `examples/lock-app/esp32/main/AppTask.cpp` — `UpdateClusterState`（`emberAfWriteAttribute`）
- `examples/all-clusters-app/esp32/main/main.cpp` — OnOff server 注册与 GPIO 映射
- `recipes/custom_attribute_callback.md` — 通用属性回调扩展
