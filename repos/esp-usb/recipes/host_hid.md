# USB 主机：HID 主机驱动（键盘/鼠标）

> **适用摘要**: 用 HID 主机驱动与 USB HID 设备（键盘、鼠标等）通信，处理驱动级与接口级事件、获取输入报告。

> Version: selected repo/component version; align with the user project and dependency manifest.
> Evidence: `repos/esp-usb/resources/`, source/examples in `repos/esp-usb/`, and this recipe path `repos/esp-usb/recipes/host_hid.md`.
> Validation: draft metadata added from repository routing; verify APIs, Kconfig symbols, and component dependencies against the selected version.

## 触发意图

- "USB 读键盘/鼠标"
- "HID 主机"
- "usb host hid"
- "HID input report"
- "hid_host"

## 前置条件

| 条件 | 要求 |
|---|---|
| 组件依赖 | `idf.py add-dependency "espressif/usb_host_hid^1.2.0"`（会带 `usb`） |
| ESP-IDF | `>= 5.5.3` |
| 参考头文件 | `host/class/hid/usb_host_hid/include/usb/hid_host.h`、`hid.h`、`hid_usage_keyboard.h`、`hid_usage_mouse.h` |

## 分步说明

### 1. 安装 HID 主机驱动

```c
#include "usb/hid_host.h"

static void hid_driver_event_cb(hid_host_device_handle_t hid_dev_handle,
                                const hid_host_driver_event_t event,
                                void *arg) {
    if (event == HID_HOST_DRIVER_EVENT_CONNECTED) {
        printf("HID device connected\n");
        // 通常在此打开接口（见下）
    } else if (event == HID_HOST_DRIVER_EVENT_DISCONNECTED) {
        printf("HID device disconnected\n");
    }
}

const hid_host_driver_config_t drv_cfg = {
    .create_background_task = true,    // true: 自建任务；false: 应用轮询
    .task_priority = 10,
    .stack_size    = 4096,
    .core_id       = tskNO_AFFINITY,
    .callback      = hid_driver_event_cb,
    .callback_arg  = NULL,
};
ESP_ERROR_CHECK(hid_host_install(&drv_cfg));
```

> 若 `create_background_task = false`，应用必须周期性调用 `hid_host_handle_events(timeout)`。

### 2. 打开 HID 接口并注册接口事件

HID 设备可能有多个接口（如键盘 + 鼠标），每个接口分别打开。

```c
static void hid_interface_event_cb(hid_host_device_handle_t hid_dev_handle,
                                   const hid_host_interface_event_t event,
                                   void *arg) {
    switch (event) {
    case HID_HOST_INTERFACE_EVENT_INPUT_REPORT:
        ; uint8_t report[64];
        uint16_t len = sizeof(report);
        // 拿原始输入报告
        if (hid_host_device_get_raw_input_report_data(hid_dev_handle, report, &len) == ESP_OK) {
            ESP_LOG_BUFFER_HEX("hid", report, len, ESP_LOG_INFO);
        }
        break;
    case HID_HOST_INTERFACE_EVENT_DISCONNECTED:
        printf("HID interface disconnected\n");
        hid_host_device_close(hid_dev_handle);
        break;
    case HID_HOST_INTERFACE_EVENT_TRANSFER_ERROR:
        printf("HID transfer error\n");
        break;
    default:
        break;
    }
}

// 在 HID_HOST_DRIVER_EVENT_CONNECTED 时：
const hid_host_device_config_t dev_cfg = {
    .callback      = hid_interface_event_cb,
    .callback_arg  = NULL,
};
ESP_ERROR_CHECK(hid_host_device_open(hid_dev_handle, &dev_cfg));
ESP_ERROR_CHECK(hid_host_device_start(hid_dev_handle));   // 开始轮询输入报告
```

### 3. 查询接口参数

```c
hid_host_dev_params_t params;
ESP_ERROR_CHECK(hid_host_device_get_params(hid_dev_handle, &params));
// params 含 device_type（HID_PROTOCOL_KEYBOARD / HID_PROTOCOL_MOUSE 等）、iface_index 等
```

### 4. 设备信息与报告描述符

```c
hid_host_dev_info_t info;
ESP_ERROR_CHECK(hid_host_get_device_info(hid_dev_handle, &info));

uint16_t desc_len = 0;
uint8_t *desc = hid_host_get_report_descriptor(hid_dev_handle, &desc_len);
// 解析 desc 来理解 report 格式
```

### 5. HID 类请求

```c
// Get Report
uint8_t report_buf[64];
hid_class_request_get_report(hid_dev_handle,
    /*report_id=*/0, /*report_type=*/1, report_buf, sizeof(report_buf));

// Get Idle
uint8_t idle_rate;
hid_class_request_get_idle(hid_dev_handle, 0, &idle_rate);

// Get Protocol
uint8_t protocol;
hid_class_request_get_protocol(hid_dev_handle, &protocol);
```

### 6. 停止、关闭、卸载

```c
hid_host_device_stop(hid_dev_handle);
hid_host_device_close(hid_dev_handle);
hid_host_uninstall();
```

### 7. 远程唤醒

HID 设备常支持远程唤醒。在挂起状态下，由设备主动发起唤醒信号（需主机先前使能）。

```c
// 使能/禁用远程唤醒
hid_host_enable_remote_wakeup(hid_dev_handle, true);
```

> 远程唤醒只能由 class 驱动使能，Host Library 本身不自动使能。

## 常见错误

| 错误 | 原因 | 解决方法 |
|---|---|---|
| 没收到 CONNECTED 事件 | 没建后台任务也没轮询 | `create_background_task=true` 或周期调 `hid_host_handle_events` |
| 收不到输入报告 | 接口未 `start` | `hid_host_device_open` 后调 `hid_host_device_start` |
| 多接口设备只识别一个 | 每个接口要分别 open/start | 在驱动事件里遍历接口逐个打开 |
| 卸载驱动失败 | 仍有接口/设备句柄 | 先 stop/close 所有句柄再 `hid_host_uninstall` |
| 报告解析错误 | 未解析报告描述符 | 用 `hid_host_get_report_descriptor` 拿原始描述符自行解析 |
| 挂起后无响应 | 未使能远程唤醒 / 未恢复 | `hid_host_enable_remote_wakeup`，主机端处理 `DEV_SUSPENDED/RESUMED` |

## 参考

- `host/class/hid/usb_host_hid/include/usb/hid_host.h` — HID 主机 API
- `host/class/hid/usb_host_hid/include/usb/hid.h` — HID 通用定义
- `host/class/hid/usb_host_hid/include/usb/hid_usage_keyboard.h`、`hid_usage_mouse.h` — 用法页
- ESP-IDF 示例（仓库文档引用）：`peripherals/usb/host/hid`
