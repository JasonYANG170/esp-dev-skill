# USB 设备：驱动安装/卸载与事件、VBUS 监测

> **适用摘要**: TinyUSB 设备驱动的安装与卸载生命周期、ATTACHED/DETACHED/SUSPEND/RESUME 事件回调、以及自供电设备的 VBUS 监测配置。

## 触发意图

- "USB 设备热插拔检测"
- "USB 插拔事件"
- "tinyusb_driver_install / uninstall"
- "自供电 USB 设备"
- "VBUS 监测"
- "USB 挂起/恢复"

## 前置条件

| 条件 | 要求 |
|---|---|
| 组件依赖 | `idf.py add-dependency "espressif/esp_tinyusb^2.2.0"` |
| menuconfig | 按需开类；`CONFIG_TINYUSB_SUSPEND_CALLBACK=y` / `CONFIG_TINYUSB_RESUME_CALLBACK=y`（要用挂起/恢复事件时） |
| 参考示例 | `device/esp_tinyusb/test_apps/dconn_detection/main/test_dconn_detection.c`（连接/断开检测） |

## 分步说明

### 1. 事件回调

`tinyusb_event_cb_t` 回调接收 `tinyusb_event_t`，其 `id` 为以下之一：

| 事件 | 含义 | 前置 menuconfig |
|---|---|---|
| `TINYUSB_EVENT_ATTACHED` | 设备被主机配置（SetConfiguration）后附加 | 始终可用 |
| `TINYUSB_EVENT_DETACHED` | 设备脱离主机（VBUS 掉落等） | 始终可用 |
| `TINYUSB_EVENT_SUSPENDED` | 设备进入挂起 | `CONFIG_TINYUSB_SUSPEND_CALLBACK=y` |
| `TINYUSB_EVENT_RESUMED` | 设备从挂起恢复 | `CONFIG_TINYUSB_RESUME_CALLBACK=y` |

```c
#include "tinyusb.h"

static void my_event_cb(tinyusb_event_t *event, void *arg) {
    switch (event->id) {
    case TINYUSB_EVENT_ATTACHED:
        printf("USB attached\n");
        break;
    case TINYUSB_EVENT_DETACHED:
        printf("USB detached\n");
        break;
    case TINYUSB_EVENT_SUSPENDED:
        // event->suspended.remote_wakeup 表示主机是否使能了远程唤醒
        printf("USB suspended (remote_wakeup=%d)\n", event->suspended.remote_wakeup);
        break;
    case TINYUSB_EVENT_RESUMED:
        printf("USB resumed\n");
        break;
    default:
        break;
    }
}
```

### 2. 安装（带事件回调）

```c
#include "tinyusb_default_config.h"
#include "tinyusb.h"

tinyusb_config_t tusb_cfg = TINYUSB_DEFAULT_CONFIG();
tusb_cfg.event_cb  = my_event_cb;
tusb_cfg.event_arg = NULL;
ESP_ERROR_CHECK(tinyusb_driver_install(&tusb_cfg));
```

### 3. 自供电设备 + VBUS 监测

USB 规范要求自供电设备监测 VBUS（>4.75 V 视为有效，<4.35 V 无效）。ESP 引脚 3.3 V 容限，需用分压电阻或比较器，且拔线后 3 ms 内 sensing 引脚必须为低。

```c
#define VBUS_GPIO_NUM   GPIO_NUM_6   // 经分压/比较器接到此引脚

tinyusb_config_t tusb_cfg = TINYUSB_DEFAULT_CONFIG();
tusb_cfg.phy.self_powered  = true;
tusb_cfg.phy.vbus_monitor_io = VBUS_GPIO_NUM;
tusb_cfg.event_cb = my_event_cb;
ESP_ERROR_CHECK(tinyusb_driver_install(&tusb_cfg));
```

> ESP32-P4 高速端口启用 VBUS 监测时，需在 `tinyusb_driver_install` 之前调用 `gpio_install_isr_service()`。

> VBUS 监测仅在下沿（VBUS 掉落）触发 DETACHED 事件。

### 4. 卸载（teardown）

```c
ESP_ERROR_CHECK(tinyusb_driver_uninstall());
```

`tinyusb_driver_uninstall()` 会停止 TinyUSB 任务、拆卸 TinyUSB 栈、释放已准备的描述符，并删除它创建的 USB PHY（若由 `tinyusb_driver_install` 创建）。若驱动未安装则返回 `ESP_ERR_INVALID_STATE`。

### 5. 挂起回调的两种用法

注意 `CONFIG_TINYUSB_SUSPEND_CALLBACK`/`CONFIG_TINYUSB_RESUME_CALLBACK` 与自定义 `tud_suspend_cb`/`tud_resume_cb` 互斥：

- 选项打开：esp_tinyusb 提供强定义的 `tud_suspend_cb`，并把事件转为 `TINYUSB_EVENT_SUSPENDED` 投给 `event_cb`。此时**应用不得**再定义 `tud_suspend_cb`，否则链接报错（多重定义）。
- 选项关闭：TinyUSB 提供弱定义，应用可自己写强定义的 `tud_suspend_cb`/`tud_resume_cb`。

### 6. 远程唤醒

设备挂起且主机使能远程唤醒时，可主动唤醒主机：

```c
// 仅在设备处于挂起、且主机使能远程唤醒时调用
esp_err_t ret = tinyusb_remote_wakeup();
if (ret == ESP_ERR_INVALID_STATE) {
    // 主机未使能远程唤醒
}
```

## 常见错误

| 错误 | 原因 | 解决方法 |
|---|---|---|
| 收不到 DETACHED 事件 | 非自供电且无 VBUS 监测 | 设 `self_powered=true` 并接好 `vbus_monitor_io` |
| 拔线后 3 ms 内 sensing 引脚没拉低 | 分压点没设计好 | 调整分压比（约 0.75×Vdd@4.4V）或用比较器 |
| 链接报 `tud_suspend_cb` 多重定义 | 同时开了 Kconfig 选项又自定义了回调 | 二选一：关选项改用 event_cb，或删自定义回调 |
| 收不到 SUSPENDED 事件 | `CONFIG_TINYUSB_SUSPEND_CALLBACK=y` 没开 | menuconfig 打开该选项 |
| P4 高速端口 VBUS 监测异常 | ISR 服务未安装 | 在 `tinyusb_driver_install` 前调 `gpio_install_isr_service()` |
| `tinyusb_driver_uninstall` 返回 INVALID_STATE | 驱动未安装或重复卸载 | 仅在已安装后卸载一次 |

## 参考

- `device/esp_tinyusb/include/tinyusb.h` — `tinyusb_event_t`、`tinyusb_phy_config_t`、`tinyusb_remote_wakeup`
- `device/esp_tinyusb/Kconfig` — `CONFIG_TINYUSB_SUSPEND_CALLBACK` / `CONFIG_TINYUSB_RESUME_CALLBACK`
- `device/esp_tinyusb/test_apps/dconn_detection/main/test_dconn_detection.c` — 插拔/挂起/恢复事件
- `docs/en/usb_device.rst` — Self-Powered Device 章节
