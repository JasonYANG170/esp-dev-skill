# 创建自定义 ESP32 Matter 设备示例

> **适用摘要**: 以 `examples/lock-app/esp32` 为模板，派生一个自定义 CHIP 设备应用：复制目录、调整 `app_main` 初始化顺序、注册回调、设置 GPIO 与 Kconfig，最小改动获得可编译可配网的固件。

## 触发意图

- "创建新的 ESP32 Matter 应用"
- "基于 lock-app 写我自己的设备"
- "自定义 CHIP 设备示例"
- "新建一个 Matter 固件工程"

## 前置条件

| 条件 | 要求 |
|---|---|
| 模板示例 | `examples/lock-app/esp32/`（最简）或 `examples/all-clusters-app/esp32/`（含设备类型选择） |
| 环境 | ESP-IDF v4.3 + CHIP `scripts/activate.sh` |
| 参考文件 | `main/main.cpp`、`main/AppTask.cpp`、`main/DeviceCallbacks.cpp` |

## 分步说明

### 1. 复制模板目录

```bash
cp -r examples/lock-app/esp32 examples/myapp/esp32
```

保留 `CMakeLists.txt`、`sdkconfig.defaults`、`partitions.csv`、`main/` 整体结构。

### 2. 保持 `app_main` 初始化顺序不变

`main/main.cpp` 中的 `app_main()` 必须遵循 `nvs → CHIPDeviceManager::Init → InitServer → StartAppTask`：

```cpp
extern "C" void app_main()
{
    nvs_flash_init();

#if CONFIG_ENABLE_CHIP_SHELL
    chip::LaunchShell();
#endif

    CHIPDeviceManager & deviceMgr = CHIPDeviceManager::GetInstance();
    deviceMgr.Init(&EchoCallbacks);   // 你自己的回调子类实例

    InitServer();

    GetAppTask().StartAppTask();
}
```

### 3. 调整 `AppConfig.h` 中的 GPIO / 任务参数

```cpp
// main/include/AppConfig.h
#define APP_TASK_NAME               "APP"
#define APP_TASK_STACK_SIZE         (3000)
#define APP_TASK_PRIORITY           2
#define APP_EVENT_QUEUE_SIZE        10

#define STATUS_LED_GPIO_NUM         GPIO_NUM_2   // 状态 LED
#define LOCK_STATE_LED              GPIO_NUM_2
#define APP_LOCK_BUTTON             0
#define APP_FUNCTION_BUTTON         0
#define APP_BUTTON_DEBOUNCE_PERIOD_MS 10
```

### 4. 在 `DeviceCallbacks.cpp` 处理你的集群

把 OnOff 逻辑替换为你的设备行为；务必校验 endpoint / attribute：

```cpp
#include <app/common/gen/attribute-id.h>
#include <app/common/gen/cluster-id.h>

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

### 5. 调整 `Kconfig.projbuild`（如需新 Rendezvous 选项或设备类型）

模板已含 `Demo -> Rendezvous Mode` choice（Bypass/Wi-Fi/BLE/Thread/Ethernet）与 `CONFIG_RENDEZVOUS_MODE` 数值映射（0/1/2/4/8）。可直接复用，或在 `menu "Demo"` 内追加自定义选项。

### 6. 构建、烧录、验证

```bash
idf.py set-target esp32
idf.py menuconfig       # 选 Rendezvous 模式
idf.py build
idf.py -p /dev/ttyUSB0 flash monitor
```

## 常见错误

| 错误 | 原因 | 解决方法 |
|---|---|---|
| 编译报 `ZCL_ON_OFF_CLUSTER_ID` 未定义 | 未包含 `gen/cluster-id.h` | `#include <app/common/gen/cluster-id.h>` |
| `InitServer` 未定义 | 未包含 `app/server/Server.h` | `#include <app/server/Server.h>` |
| 改动 `gen/*.h` 后数据模型不一致 | 手改了生成文件 | 还原 `gen/*.h`，改用 `emberAfWriteAttribute` 写属性 |
| 配网后不响应 | `PostAttributeChangeCallback` 未校验 endpoint | 加 `VerifyOrExit(endpointId == 1 ...)` |
| `app_main` 编译警告 unused | 删了某 init 但保留变量 | 同步清理相关声明 |

## 参考

- `examples/lock-app/esp32/main/main.cpp` — 标准入口
- `examples/lock-app/esp32/main/AppTask.cpp` — 任务循环 + TryLockChipStack 用法
- `examples/lock-app/esp32/main/DeviceCallbacks.cpp` — 回调实现模板
- `examples/lock-app/esp32/main/Kconfig.projbuild` — Rendezvous 菜单定义
- `recipes/custom_attribute_callback.md` — 自定义属性回调详解
