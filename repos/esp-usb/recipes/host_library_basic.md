# USB 主机：Host Library 基本用法

> **适用摘要**: USB Host Library 的安装、Daemon Task、client 注册、设备打开/接口 claim/裸传输，以及完整的卸载流程。

## 触发意图

- "USB host library"
- "usb_host_install"
- "USB 主机底层 API"
- "raw USB transfer"
- "自定义 USB class driver"

## 前置条件

| 条件 | 要求 |
|---|---|
| 组件依赖 | `idf.py add-dependency "espressif/usb^1.4.1"` |
| ESP-IDF | `>= 5.5.3` |
| 目标 | S2/S3/S31/P4/H4（需 USB-OTG，`SOC_USB_OTG_SUPPORTED`） |
| 参考文档 | `docs/en/usb_host.rst`（Usage / Clients & Class Driver 段） |

## 分步说明

### 1. 安装 Host Library + 启动 Daemon Task

Host Library 不自己建任务，必须由应用建 Daemon Task 调 `usb_host_lib_handle_events`。

```c
#include "usb/usb_host.h"

static void daemon_task(void *arg) {
    bool exit = false;
    while (!exit) {
        uint32_t flags;
        usb_host_lib_handle_events(portMAX_DELAY, &flags);
        if (flags & USB_HOST_LIB_EVENT_FLAGS_NO_CLIENTS) {
            // 所有 client 已注销；尝试释放设备
            if (usb_host_device_free_all() == ESP_OK) {
                // 设备已全部释放
            }
        }
        if (flags & USB_HOST_LIB_EVENT_FLAGS_ALL_FREE) {
            exit = true;
        }
    }
    usb_host_uninstall();
    vTaskDelete(NULL);
}

void app_main(void) {
    const usb_host_config_t host_cfg = {
        .intr_flags = ESP_INTR_FLAG_LEVEL1,
    };
    ESP_ERROR_CHECK(usb_host_install(&host_cfg));

    xTaskCreate(daemon_task, "usb_daemon", 4096, NULL, 10, NULL);
    // ... 启动 client 任务 ...
}
```

### 2. Client 注册与事件回调

每个 client 一般对应一个任务，且该 client 的所有 `usb_host_client_*` 调用都应来自同一任务。

```c
#define ACTION_OPEN_DEV   0x01
#define ACTION_XFER       0x02
#define ACTION_CLOSE_DEV  0x03

struct driver_ctrl {
    uint32_t actions;
    uint8_t dev_addr;
    usb_host_client_handle_t client_hdl;
    usb_device_handle_t dev_hdl;
};

static void client_event_cb(const usb_host_client_event_msg_t *msg, void *arg) {
    struct driver_ctrl *c = arg;
    switch (msg->event) {
    case USB_HOST_CLIENT_EVENT_NEW_DEV:
        c->actions |= ACTION_OPEN_DEV;
        c->dev_addr = msg->new_dev.address;
        break;
    case USB_HOST_CLIENT_EVENT_DEV_GONE:
        c->actions |= ACTION_CLOSE_DEV;
        break;
    default:
        break;
    }
}

static void transfer_cb(usb_transfer_t *transfer) {
    struct driver_ctrl *c = transfer->context;
    printf("xfer status=%d, bytes=%d\n", transfer->status, transfer->actual_num_bytes);
    c->actions |= ACTION_CLOSE_DEV;
}
```

> 回调在 `usb_host_client_handle_events` 内部执行，**不要阻塞**，只置标志，实际处理放任务循环里。

### 3. Client Task：打开设备、claim 接口、提交传输

```c
static void client_task(void *arg) {
    struct driver_ctrl c = {0};

    usb_host_client_config_t client_cfg = {
        .is_synchronous = false,
        .max_num_event_msg = 5,
        .async.client_event_callback = client_event_cb,
        .async.callback_arg = &c,
    };
    ESP_ERROR_CHECK(usb_host_client_register(&client_cfg, &c.client_hdl));

    // 预分配一个传输（带 1024 字节数据缓冲）
    usb_transfer_t *transfer;
    ESP_ERROR_CHECK(usb_host_transfer_alloc(1024, 0, &transfer));

    bool exit = false;
    while (!exit) {
        usb_host_client_handle_events(c.client_hdl, portMAX_DELAY);

        if (c.actions & ACTION_OPEN_DEV) {
            c.actions &= ~ACTION_OPEN_DEV;
            ESP_ERROR_CHECK(usb_host_device_open(c.client_hdl, c.dev_addr, &c.dev_hdl));
            ESP_ERROR_CHECK(usb_host_interface_claim(c.client_hdl, c.dev_hdl, 1, 0));
            c.actions |= ACTION_XFER;
        }
        if (c.actions & ACTION_XFER) {
            c.actions &= ~ACTION_XFER;
            memset(transfer->data_buffer, 0xAA, 1024);
            transfer->num_bytes = 1024;
            transfer->device_handle = c.dev_hdl;
            transfer->bEndpointAddress = 0x01;   // OUT EP1
            transfer->callback = transfer_cb;
            transfer->context = &c;
            ESP_ERROR_CHECK(usb_host_transfer_submit(transfer));
        }
        if (c.actions & ACTION_CLOSE_DEV) {
            c.actions &= ~ACTION_CLOSE_DEV;
            usb_host_interface_release(c.client_hdl, c.dev_hdl, 1);
            usb_host_device_close(c.client_hdl, c.dev_hdl);
            exit = true;
        }
    }

    usb_host_transfer_free(transfer);
    usb_host_client_deregister(c.client_hdl);
    vTaskDelete(NULL);
}
```

### 4. 解析描述符

打开设备后可读设备/配置描述符判断设备类型：

```c
const usb_device_desc_t *dev_desc;
const usb_config_desc_t *cfg_desc;
usb_host_get_device_descriptor(c.dev_hdl, &dev_desc);
usb_host_get_active_config_descriptor(c.dev_hdl, &cfg_desc);
// 用 usb/usb_helpers.h 里的 helper 解析端点/接口
```

### 5. 控制传输

控制传输用 `usb_host_transfer_submit_control`，需先填好 8 字节 setup 包：

```c
// setup 在 data_buffer 前 8 字节
usb_setup_packet_t *setup = (usb_setup_packet_t *)transfer->data_buffer;
setup->bmRequestType = ...;
setup->bRequest = ...;
transfer->num_bytes = 8 + data_len;   // setup + data
transfer->bEndpointAddress = 0;        // EP0
transfer->callback = ctrl_cb;
usb_host_transfer_submit_control(c.client_hdl, transfer);
```

### 6. Client 事件类型速查

| 事件 | 含义 |
|---|---|
| `USB_HOST_CLIENT_EVENT_NEW_DEV` | 新设备已枚举，含 `new_dev.address` |
| `USB_HOST_CLIENT_EVENT_DEV_GONE` | 本 client 打开的设备消失，含 `dev_gone.dev_hdl`（清理信号） |
| `USB_HOST_CLIENT_EVENT_DEV_SUSPENDED` / `_RESUMED` | 本 client 打开的设备挂起/恢复 |
| `USB_HOST_CLIENT_EVENT_DEV_REMOVED` | 任意设备移除（需 `flags.notify_dev_removed=1`），只有地址 |

## 常见错误

| 错误 | 原因 | 解决方法 |
|---|---|---|
| `ESP_ERR_INVALID_STATE` 安装失败 | 已安装或状态不对 | 先卸载；确保只 install 一次 |
| 设备永远到不了 NEW_DEV | Daemon Task 没跑 `usb_host_lib_handle_events` | 确保 Daemon Task 在循环里调该函数 |
| 提交传输失败/行为异常 | 对应接口未 claim | 先 `usb_host_interface_claim` 再 submit |
| 回调里做重活导致其他事件饿死 | 回调内阻塞 | 回调只置标志，处理放任务循环 |
| 卸载返回 INVALID_STATE | 设备未全部释放 | 先 `usb_host_device_free_all`，等 `ALL_FREE` 再 uninstall |
| 多个 client 抢同一接口 | 同一接口只能被一个 client claim | 不同 client 用不同接口 |

## 参考

- `host/usb/include/usb/usb_host.h` — 全部 Host Library API
- `host/usb/include/usb/usb_helpers.h` — 描述符解析 helper
- `docs/en/usb_host.rst` — Usage、Lifecycle、Power Management 章节
- ESP-IDF 示例（仓库文档引用）：`peripherals/usb/host/usb_host_lib`
