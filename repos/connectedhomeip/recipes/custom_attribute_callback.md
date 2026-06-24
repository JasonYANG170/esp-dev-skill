# 实现自定义属性回调并响应集群写入

> **适用摘要**: 扩展 `CHIPDeviceManagerCallbacks::PostAttributeChangeCallback`，使其能处理多个集群/属性；并演示如何用 `VerifyOrExit` 安全过滤、用 `EmberAfStatus` 回写服务端属性。

## 触发意图

- "自定义属性回调"
- "响应集群写入"
- "扩展 PostAttributeChangeCallback"
- "处理多个 cluster"
- "EmberAfStatus"

## 前置条件

| 条件 | 要求 |
|---|---|
| 基类 | `CHIPDeviceManagerCallbacks`（`main/include/CHIPDeviceManager.h`） |
| 生成头 | `<app/common/gen/cluster-id.h>`、`attribute-id.h` |
| 工具宏 | `<support/CodeUtils.h>`（`VerifyOrExit`） |

## 分步说明

### 1. 子类化并重写虚函数

```cpp
// main/include/DeviceCallbacks.h
#include "CHIPDeviceManager.h"

class DeviceCallbacks : public chip::DeviceManager::CHIPDeviceManagerCallbacks
{
public:
    void DeviceEventCallback(const chip::DeviceLayer::ChipDeviceEvent * event, intptr_t arg) override;
    void PostAttributeChangeCallback(chip::EndpointId endpoint, chip::ClusterId clusterId,
                                     chip::AttributeId attributeId, uint8_t mask,
                                     uint16_t manufacturerCode, uint8_t type,
                                     uint16_t size, uint8_t * value) override;

private:
    void OnOnOffPostAttributeChangeCallback(chip::EndpointId endpointId, chip::AttributeId attributeId, uint8_t * value);
    void OnInternetConnectivityChange(const chip::DeviceLayer::ChipDeviceEvent * event);
    void OnSessionEstablished(const chip::DeviceLayer::ChipDeviceEvent * event);
};
```

### 2. 在回调中按 cluster 分发

```cpp
#include <app/common/gen/cluster-id.h>

void DeviceCallbacks::PostAttributeChangeCallback(EndpointId endpointId, ClusterId clusterId,
                                                  AttributeId attributeId, uint8_t mask,
                                                  uint16_t manufacturerCode, uint8_t type,
                                                  uint16_t size, uint8_t * value)
{
    ESP_LOGI(TAG, "PostAttributeChangeCallback - Cluster ID: '0x%04x', EndPoint ID: '0x%02x', Attribute ID: '0x%04x'",
             clusterId, endpointId, attributeId);

    switch (clusterId) {
    case ZCL_ON_OFF_CLUSTER_ID:
        OnOnOffPostAttributeChangeCallback(endpointId, attributeId, value);
        break;
    // 在此追加更多集群：level control / door lock / temperature ...
    default:
        ESP_LOGI(TAG, "Unhandled cluster ID: %d", clusterId);
        break;
    }
}
```

### 3. 用 `VerifyOrExit` 做安全过滤

`VerifyOrExit(cond, action)` 在 cond 为假时跳转到 `exit:`，避免误动作：

```cpp
#include <support/CodeUtils.h>

void DeviceCallbacks::OnOnOffPostAttributeChangeCallback(EndpointId endpointId, AttributeId attributeId, uint8_t * value)
{
    VerifyOrExit(attributeId == ZCL_ON_OFF_ATTRIBUTE_ID,
                 ESP_LOGI(TAG, "Unhandled Attribute ID: '0x%04x", attributeId));
    VerifyOrExit(endpointId == 1 || endpointId == 2,
                 ESP_LOGE(TAG, "Unexpected EndPoint ID: `0x%02x'", endpointId));

    // 此处安全地执行业务逻辑
exit:
    return;
}
```

### 4. 服务端属性回写（返回值校验）

```cpp
#include <app/util/af-enums.h>

EmberAfStatus status = emberAfWriteAttribute(1, ZCL_ON_OFF_CLUSTER_ID, ZCL_ON_OFF_ATTRIBUTE_ID,
                                             CLUSTER_MASK_SERVER, (uint8_t *) &v, ZCL_BOOLEAN_ATTRIBUTE_TYPE);
VerifyOrExit(status == EMBER_ZCL_STATUS_SUCCESS, ESP_LOGI(TAG, "write failed: %x", status));
```

## 常见错误

| 错误 | 原因 | 解决方法 |
|---|---|---|
| 回调不被调用 | 未把子类实例传给 `CHIPDeviceManager::Init` | `deviceMgr.Init(&myCallbacks)` |
| `VerifyOrExit` 编译错 | 未包含 `<support/CodeUtils.h>` | 加 `#include <support/CodeUtils.h>` |
| 误处理其他 endpoint | 缺 endpoint 校验 | 加 `VerifyOrExit(endpointId == 1 ...)` |
| `ZCL_*` 宏未定义 | 未包含 gen 头 | `#include <app/common/gen/cluster-id.h>` 等 |
| 多次触发动作 | 未检查动作进行中 | 调用前查 `BoltLockMgr().IsActionInProgress()` |

## 参考

- `examples/lock-app/esp32/main/DeviceCallbacks.cpp` — 完整回调实现
- `examples/lock-app/esp32/main/include/CHIPDeviceManager.h` — 回调基类定义
- `recipes/onoff_cluster_hardware.md` — OnOff → GPIO 绑定
