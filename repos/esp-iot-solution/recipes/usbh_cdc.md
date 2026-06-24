# USB 主机 CDC 串口（iot_usbh_cdc）

> **适用摘要**: 使用 `iot_usbh_cdc` 组件在 ESP32-S2/ESP32-S3/ESP32-P4 等 USB Host 上接入 USB CDC-ACM 串口设备（如 4G 模组、USB 转串口），安装驱动、打开端口、收发数据并处理设备连接/断开事件。驱动提供带环形缓冲与不带缓冲两种模式，以及自定义控制传输接口。

## 触发意图

- "USB CDC 串口"
- "USB 转 TTL 收发"
- "iot_usbh_cdc"
- "USB Host 读 CDC 设备"
- "usbh_cdc_driver_install"

## 前置条件

| 条件 | 要求 |
|---|---|
| 目标芯片 | 带 USB OTG/Host 的 ESP 芯片（ESP32-S2/S3/P4 等） |
| IDF 环境 | ESP-IDF v5.0+（master 需 v5.3+） |
| 组件依赖 | `espressif/iot_usbh_cdc`（及共享的 `espressif/usb_stream` 或 USB Host 栈） |
| 参考示例 | `examples/usb/host/usb_cdc_basic/`、`examples/usb/host/usb_cdc_4g_module/` |

## 分步说明

### 1. 添加组件依赖

```bash
idf.py set-target esp32s3
idf.py add-dependency "espressif/iot_usbh_cdc"
```

### 2. 安装 CDC 驱动

```c
#include "iot_usbh_cdc.h"

usbh_cdc_driver_config_t drv_cfg = {
    .task_stack_size = 4 * 1024,
    .task_priority   = 5,
    .task_coreid     = -1,                 // 不指定核
    .skip_init_usb_host_driver = false,    // 由本组件初始化底层 USB Host 栈
};
ESP_ERROR_CHECK(usbh_cdc_driver_install(&drv_cfg));
```

### 3. 注册设备连接/断开回调

设备匹配用 `usb_device_match_id_t` 列表，末尾以 NULL 项结束。回调收到 `CDC_HOST_DEVICE_EVENT_CONNECTED` / `CDC_HOST_DEVICE_EVENT_DISCONNECTED`。

```c
static void cdc_dev_event_cb(usbh_cdc_device_event_t event,
                             usbh_cdc_device_event_data_t *data, void *ctx)
{
    if (event == CDC_HOST_DEVICE_EVENT_CONNECTED) {
        ESP_LOGI(TAG, "CDC dev 0x%02x connected, itf=%d",
                 data->new_dev.dev_addr, data->new_dev.matched_intf_num);
    } else {
        ESP_LOGI(TAG, "CDC dev gone");
    }
}

static const usb_device_match_id_t dev_match_list[] = {
    { .idVendor = 0x1234, .idProduct = 0x5678 },  // 按需填 VID/PID
    { 0 },                                         // NULL 结束项
};
usbh_cdc_register_dev_event_cb(dev_match_list, cdc_dev_event_cb, NULL);
```

### 4. 打开端口（带环形缓冲）

```c
static usbh_cdc_port_handle_t s_port = NULL;

static void on_recv(usbh_cdc_port_handle_t h, void *user_data)
{
    uint8_t buf[256];
    size_t len = sizeof(buf);
    if (usbh_cdc_read_bytes(h, buf, &len, 0) == ESP_OK && len > 0) {
        ESP_LOG_BUFFER_HEX(TAG, buf, len);
    }
}

void open_cdc_port(uint8_t dev_addr, uint8_t itf_num)
{
    usbh_cdc_port_config_t port_cfg = {
        .dev_addr                = dev_addr,
        .itf_num                 = itf_num,
        .in_ringbuf_size         = 4 * 1024,    // >0 启用接收环形缓冲
        .out_ringbuf_size        = 4 * 1024,    // >0 启用发送环形缓冲
        .in_transfer_buffer_size = 512,
        .out_transfer_buffer_size= 512,
        .cbs = {
            .recv_data = on_recv,               // 有数据时回调
            .closed    = NULL,
            .notif_cb  = NULL,
            .user_data = NULL,
        },
    };
    ESP_ERROR_CHECK(usbh_cdc_port_open(&port_cfg, &s_port));
}
```

> 不使用环形缓冲时把 `in_ringbuf_size`/`out_ringbuf_size` 设为 0：此时需在 `recv_data` 回调里直接 `usbh_cdc_read_bytes`（`ticks_to_wait=0`），且 `length` 必须等于 `in_transfer_buffer_size`。

### 5. 读写数据

```c
// 发送（带环形缓冲时为异步，写入发送环形缓冲）
size_t tx_len = 5;
usbh_cdc_write_bytes(s_port, (const uint8_t *)"hello", tx_len, pdMS_TO_TICKS(100));

// 读取（从接收环形缓冲弹出）
uint8_t rbuf[256];
size_t rlen = sizeof(rbuf);
usbh_cdc_read_bytes(s_port, rbuf, &rlen, pdMS_TO_TICKS(100));
```

### 6. 自定义控制传输（设置波特率等）

CDC-ACM 设置线路参数属类特定请求（`bmRequestType=0x21`，`bRequest=0x03 SET_LINE_CODING`）：

```c
uint8_t line_coding[7] = {
    0x00, 0xC2, 0x01, 0x00,   // 115200 baud (LE)
    0x00,                     // 1 stop bit
    0x00,                     // no parity
    0x08,                     // 8 data bits
};
usbh_cdc_send_custom_request(s_port,
    0x21, 0x03, 0x0000, 0x0000, sizeof(line_coding), line_coding);
```

### 7. 关闭与卸载

```c
usbh_cdc_port_close(s_port);
usbh_cdc_driver_uninstall();   // 须所有端口已关闭
```

## 常见错误

| 错误 | 原因 | 解决方法 |
|---|---|---|
| `ESP_ERR_INVALID_STATE`（驱动已安装） | 重复调用 `usbh_cdc_driver_install` | 驱动全局单例，只装一次 |
| `usbh_cdc_port_open` 返回 `INVALID_STATE` | 设备未连接 | 在 `CONNECTED` 事件里再 open；或先 `usb_streaming_connect_wait` 等待 |
| 无 `recv_data` 回调 | 用了无环形缓冲模式但未注册回调 | 无缓冲模式必须靠回调读；或改用 `in_ringbuf_size>0` 后轮询 `usbh_cdc_read_bytes` |
| 无缓冲读返回 `INVALID_ARG` | `length` 与 `in_transfer_buffer_size` 不等 | 无缓冲模式下读长度必须等于内部传输缓冲大小 |
| `usbh_cdc_driver_uninstall` 失败 | 仍有端口未关闭 | 先 `usbh_cdc_port_close` 所有端口 |
| 收到乱码 | 未设置波特率/线路参数 | 用 `usbh_cdc_send_custom_request` 发 SET_LINE_CODING |

## 参考

- 组件头文件：`components/usb/iot_usbh_cdc/include/iot_usbh_cdc.h`、`iot_usbh_cdc_type.h`、`usbh_helper.h`
- 在线文档：`docs/en/usb/usb_host/usb_host_iot_usbh_cdc.rst`
- 真实示例：`examples/usb/host/usb_cdc_basic/`、`examples/usb/host/usb_cdc_4g_module/`
