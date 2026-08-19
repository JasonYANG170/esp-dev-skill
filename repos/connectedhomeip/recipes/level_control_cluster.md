# 调光 / Level Control 集群服务端（current-level 写入与硬件同步）

> **适用摘要**: 集成 `Level Control` 集群（`ZCL_LEVEL_CONTROL_CLUSTER_ID = 0x0008`）服务端，实现亮度/级别控制。涵盖 `CurrentLevel` 属性初始化（`ZCL_CURRENT_LEVEL_ATTRIBUTE_ID = 0x0000`，`uint8_t` 0–255）、启动时硬件状态同步钩子 `emberAfPluginLevelControlClusterServerPostInitCallback`，以及 chip-tool 的完整调光命令集。本仓库无 `lighting-app/esp32`，因此 `all-clusters-app` 是唯一的 ESP32 参考实现。

> Version: selected repo/component version; align with the user project and dependency manifest.
> Validation: draft metadata added from repository routing; verify APIs, Kconfig symbols, and component dependencies against the selected version.

## 触发意图

- "调光"
- "Level Control cluster"
- "亮度控制 / dimming"
- "emberAfPluginLevelControlClusterServerPostInitCallback"
- "ZCL_CURRENT_LEVEL_ATTRIBUTE_ID"
- "move / step / move-to-level"

## 前置条件

| 条件 | 要求 |
|---|---|
| 参考示例 | `examples/all-clusters-app/esp32/`（仓库中唯一演示 Level Control 的 ESP32 示例） |
| 服务端头文件 | `<app/clusters/level-control/level-control.h>` |
| 生成宏 | `<app/common/gen/cluster-id.h>`（`ZCL_LEVEL_CONTROL_CLUSTER_ID = 0x0008`）、`<app/common/gen/attribute-id.h>`（`ZCL_CURRENT_LEVEL_ATTRIBUTE_ID = 0x0000`）、`<app/common/gen/attribute-type.h>`（`ZCL_INT8U_ATTRIBUTE_TYPE`） |
| 主机验证 | `examples/chip-tool/`，`levelcontrol` 集群命令集 |

## 分步说明

### 1. 启动时初始化 CurrentLevel（all-clusters 模式）

`CurrentLevel` 是 `uint8_t`（0–255），0 通常表示"最小亮度/关"，255 表示"最大"。`all-clusters-app` 在 `app_main` 初始化 server 后，对每个照明 endpoint 调用 `SetupInitialLevelControlValues` 写入最大值：

```cpp
// examples/all-clusters-app/esp32/main/main.cpp
void SetupInitialLevelControlValues(chip::EndpointId endpointId)
{
    uint8_t level = UINT8_MAX;  // 255

    emberAfWriteAttribute(endpointId, ZCL_LEVEL_CONTROL_CLUSTER_ID,
                          ZCL_CURRENT_LEVEL_ATTRIBUTE_ID, CLUSTER_MASK_SERVER,
                          &level, ZCL_INT8U_ATTRIBUTE_TYPE);
}

// 在 app_main 中，InitServer() 之后调用：
SetupInitialLevelControlValues(/* endpointId = */ 1);
SetupInitialLevelControlValues(/* endpointId = */ 2);
```

> 注意这里用的是 `emberAfWriteAttribute`（带 `CLUSTER_MASK_SERVER`），不是 `emberAfWriteServerAttribute`——两者等价，后者是前者的便捷封装。类型必须用 `ZCL_INT8U_ATTRIBUTE_TYPE`（uint8）。

### 2. 实现 PostInit 回调（硬件状态同步）

Level Control server 头文件定义了一个钩子，用于在启动时 / endpoint 解析完 OnOff 状态后，把硬件（PWM、LED 驱动）同步到当前 level 值：

```cpp
// src/app/clusters/level-control/level-control.h
/** Following resolution of the On/Off state at startup for this endpoint,
 *  perform any additional initialization needed; e.g., synchronize hardware state. */
void emberAfPluginLevelControlClusterServerPostInitCallback(chip::EndpointId endpoint);
```

应用实现该弱符号，读取当前 `CurrentLevel` 并设置 PWM 占空比：

```cpp
#include <app/clusters/level-control/level-control.h>
#include <app/common/gen/cluster-id.h>
#include <app/common/gen/attribute-id.h>
#include <app/util/af.h>

void emberAfPluginLevelControlClusterServerPostInitCallback(chip::EndpointId endpoint)
{
    uint8_t currentLevel = 0;
    EmberAfStatus status = emberAfReadAttribute(endpoint, ZCL_LEVEL_CONTROL_CLUSTER_ID,
                                                ZCL_CURRENT_LEVEL_ATTRIBUTE_ID,
                                                CLUSTER_MASK_SERVER, &currentLevel,
                                                sizeof(currentLevel), ZCL_INT8U_ATTRIBUTE_TYPE);
    if (status == EMBER_ZCL_STATUS_SUCCESS) {
        // 把 currentLevel (0-255) 映射到 PWM 占空比
        uint32_t duty = (currentLevel * 1023) / 255;  // LEDC 10-bit
        // ledc_set_duty(...); ledc_update_duty(...);
        ESP_LOGI(TAG, "LevelControl PostInit ep %u -> level %u", endpoint, currentLevel);
    }
}
```

server 插件内部以固定频率（默认 `EMBER_AF_PLUGIN_LEVEL_CONTROL_TICKS_PER_SECOND = 32`）驱动 level 变化（move/step 的平滑过渡），应用只需在 PostInit / 属性变化时同步硬件。

### 3. 响应 level 变化（属性回调）

如果你要在 controller 改 `CurrentLevel` 时同步硬件，在 `PostAttributeChangeCallback` 加分支（与 OnOff 同一入口）：

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
    case ZCL_LEVEL_CONTROL_CLUSTER_ID:
        if (attributeId == ZCL_CURRENT_LEVEL_ATTRIBUTE_ID && size == 1) {
            uint8_t level = *value;
            // 同步 PWM 占空比
            ESP_LOGI(TAG, "level -> %u on ep %u", level, endpointId);
        }
        break;
    default:
        break;
    }
}
```

### 4. 用 chip-tool 下发调光命令

`levelcontrol` 集群命令集覆盖了全部调光语义（move / move-to-level / step / stop，各带 `-with-on-off` 变体）：

```bash
chip-tool levelcontrol
# Commands:
#   move / move-with-on-off
#   move-to-level / move-to-level-with-on-off
#   step / step-with-on-off
#   stop / stop-with-on-off
#   discover / read / report

# 平滑移动到 level 128（endpoint 1）
chip-tool levelcontrol move-to-level 128 0 0 1

# 查看 move-to-level 的参数列表
chip-tool levelcontrol move-to-level

# 读取当前 level
chip-tool levelcontrol read current-level 1

# 订阅 level 变化
chip-tool levelcontrol report current-level 1
```

## 常见错误

| 错误 | 原因 | 解决方法 |
|---|---|---|
| 写入后 chip-tool 读到旧值 | 用了错误的 mask 或类型 | `CLUSTER_MASK_SERVER` + `ZCL_INT8U_ATTRIBUTE_TYPE`（uint8） |
| PostInit 不被调用 | 未正确链接 server 插件或函数符号名拼错 | 函数名必须严格为 `emberAfPluginLevelControlClusterServerPostInitCallback` |
| move/step 平滑过渡卡顿 | `EMBER_AF_PLUGIN_LEVEL_CONTROL_TICKS_PER_SECOND` 太低 | 默认 32；按需在编译期重定义 |
| Level Control 不联动 OnOff | endpoint 上 OnOff 与 Level Control 未同时配置 | zap 中同时启用两者；调光通常依赖 OnOff 关联 |
| `read current-level` 报 NOT_FOUND | 该 endpoint 无 Level Control server | 在 zap 加 Level Control server 后重新生成 |
| PWM 不跟随 | 仅在 PostInit 同步、未在属性回调同步 | 在 `PostAttributeChangeCallback` 的 `ZCL_LEVEL_CONTROL_CLUSTER_ID` 分支也同步硬件 |

## 参考项目

- `examples/all-clusters-app/esp32/main/main.cpp` — `SetupInitialLevelControlValues()` 写入 `ZCL_CURRENT_LEVEL_ATTRIBUTE_ID`（endpoint 1/2）
- `src/app/clusters/level-control/level-control.h` — `emberAfPluginLevelControlClusterServerPostInitCallback`、`EMBER_AF_PLUGIN_LEVEL_CONTROL_TICKS_PER_SECOND`
- `examples/chip-tool/README.md` — `levelcontrol` 集群命令集（move/move-to-level/step/stop 及 with-on-off 变体）
- `recipes/onoff_cluster_hardware.md` — OnOff → GPIO（调光通常与 OnOff 关联）
- `recipes/custom_attribute_callback.md` — 通用属性回调分发框架
