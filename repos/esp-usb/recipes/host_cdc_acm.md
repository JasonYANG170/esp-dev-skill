# USB 主机：CDC-ACM 主机驱动

> **适用摘要**: 用 CDC-ACM 主机驱动与 USB 串口设备/调制解调器通信，包括 CP210x、FTDI、CH34x 等 vendor-specific 芯片。

> Version: selected repo/component version; align with the user project and dependency manifest.
> Evidence: `repos/esp-usb/resources/`, source/examples in `repos/esp-usb/`, and this recipe path `repos/esp-usb/recipes/host_cdc_acm.md`.
> Validation: draft metadata added from repository routing; verify APIs, Kconfig symbols, and component dependencies against the selected version.

## 触发意图

- "USB CDC-ACM 主机"
- "读取 USB 串口模块"
- "连接 USB 调制解调器 / modem"
- "CP210x / FTDI / CH34x"
- "cdc_acm_host"

## 前置条件

| 条件 | 要求 |
|---|---|
| 组件依赖 | `idf.py add-dependency "espressif/usb_host_cdc_acm^2.4.0"`（会带 `usb`） |
| ESP-IDF | `>= 5.5.3` |
| 参考头文件 | `host/class/cdc/usb_host_cdc_acm/include/usb/cdc_acm_host.h`、`cdc_host_types.h` |

## 分步说明

### 1. 安装 CDC-ACM 主机驱动

```c
#include "usb/cdc_acm_host.h"

const cdc_acm_host_driver_config_t driver_cfg = {
    .driver_task_stack_size = 4096,
    .driver_task_priority   = 10,    // 应高于使用它的应用任务
    .xCoreID                = 0,
    .new_dev_cb             = NULL,  // 可选：新设备连入回调
};
ESP_ERROR_CHECK(cdc_acm_host_install(&driver_cfg));
```

> 驱动内部会处理与 Host Library 的交互，应用通常不用直接管理 Daemon Task。

### 2. 打开 CDC-ACM 设备

`cdc_acm_host_open` 是一个泛型宏，按第一参数类型分派到两种形式：

- **Form 1（推荐，VID/PID/地址感知）**：传 `const cdc_acm_host_open_config_t *`
- **Form 2（传统，带单接口配置）**：传 `uint16_t vid`，再 `uint16_t pid, uint8_t interface_idx, const cdc_acm_host_device_config_t *`

```c
// Form 1: 按 VID/PID 打开
const cdc_acm_host_open_config_t open_cfg = {
    .vid = 0x10C4,                  // CP210x 的 VID，或用 CDC_HOST_ANY_VID
    .pid = 0xEA60,                  // 或 CDC_HOST_ANY_PID
    .interface_idx = 0,
    .dev_addr = CDC_HOST_ANY_DEV_ADDR,
    .connection_timeout_ms = 5000,
    .out_buffer_size = 512,
    .in_buffer_size  = 512,
    .event_cb = NULL,
    .data_cb  = cdc_data_cb,        // 收数回调
    .user_arg = NULL,
};
cdc_acm_dev_hdl_t cdc_hdl;
ESP_ERROR_CHECK(cdc_acm_host_open(&open_cfg, &cdc_hdl));
```

```c
// Form 2: 传统，用于 vendor-specific 或需细粒度配置
const cdc_acm_host_device_config_t dev_cfg = {
    .connection_timeout_ms = 3000,
    .out_buffer_size = 512,
    .in_buffer_size  = 512,
    .event_cb = NULL,
    .data_cb  = cdc_data_cb,
    .user_arg = NULL,
    .dev_addr = CDC_HOST_ANY_DEV_ADDR,
};
cdc_acm_dev_hdl_t cdc_hdl;
ESP_ERROR_CHECK(cdc_acm_host_open(0x10C4, 0xEA60, 0, &dev_cfg, &cdc_hdl));
// vendor-specific 芯片可用 cdc_acm_host_open_vendor_specific() 宏（等价 Form 2）
```

### 3. 接收数据回调

`cdc_acm_data_callback_t`：返回 `true` 表示数据已处理、可清空 RX 缓冲；返回 `false` 表示未处理、新数据追加。

```c
static bool cdc_data_cb(const uint8_t *data, size_t data_len, void *user_arg) {
    // 只读类设备（write-only 时 data_cb 可为 NULL）
    printf("RX %u bytes\n", (unsigned)data_len);
    ESP_LOG_BUFFER_HEX("cdc", data, data_len, ESP_LOG_INFO);
    return true;   // 处理完毕，可清空
}
```

### 4. 发送数据（阻塞）

```c
uint8_t out[] = "AT\r\n";
ESP_ERROR_CHECK(cdc_acm_host_data_tx_blocking(cdc_hdl, out, sizeof(out) - 1, 100));
// 第三个参数为超时 ms
```

### 5. 设备事件 / 协议信息

```c
// 设备事件回调（在 dev_cfg/open_cfg.event_cb 注册）
static void cdc_event_cb(const cdc_acm_host_dev_event_data_t *event, void *ctx) {
    switch (event->type) {
    case CDC_ACM_HOST_DEVICE_DISCONNECTED:
        printf("CDC device disconnected\n");
        break;
    case CDC_ACM_HOST_ERROR:
        printf("CDC error: %d\n", event->data.error);
        break;
    case CDC_ACM_HOST_SERIAL_STATE:
        // event->data.serial_state 为串口状态位图
        break;
    default:
        break;
    }
}

// 查询通信/数据协议
cdc_comm_protocol_t comm;
cdc_data_protocol_t data_proto;
cdc_acm_host_protocols_get(cdc_hdl, &comm, &data_proto);
```

### 6. 自定义/类请求

```c
// 发送自定义 CDC 请求（bmRequestType/bRequest/wValue/wIndex/wLength + data）
cdc_acm_host_send_custom_request(cdc_hdl,
    /*bmRequestType=*/0x21, /*bRequest=*/0x20,
    /*wValue=*/0, /*wIndex=*/0, /*wLength=*/0, NULL);

// 打印设备描述符（调试用）
cdc_acm_host_desc_print(cdc_hdl);
```

### 7. 关闭与卸载

```c
ESP_ERROR_CHECK(cdc_acm_host_close(cdc_hdl));   // 所有设备关闭后才能卸载
ESP_ERROR_CHECK(cdc_acm_host_uninstall());
```

### 8. Vendor-specific VCP 子组件

对于非标准 CDC-ACM 的桥接芯片，仓库提供独立子组件，配合 `cdc_acm_host_open_vendor_specific` 使用：

| 子组件 | 芯片 |
|---|---|
| `usb_host_ch34x_vcp` (`usb/vcp_ch34x.h`) | CH340/CH34x |
| `usb_host_cp210x_vcp` (`usb/vcp_cp210x.h`) | CP210x |
| `usb_host_ftdi_vcp` (`usb/vcp_ftdi.h`) | FTDI FT23x |

## 常见错误

| 错误 | 原因 | 解决方法 |
|---|---|---|
| 打开超时 | 设备未连入或 VID/PID 不匹配 | 用 `CDC_HOST_ANY_VID/ANY_PID` 或核对芯片 VID/PID |
| `ESP_ERR_INVALID_STATE` 卸载失败 | 有设备未关闭 | 先对每个 handle 调 `cdc_acm_host_close` |
| 收不到数据 | `data_cb` 为 NULL（只读设备必须有） | 设置 `data_cb` |
| 发送大块数据被拆包 | 超出 `out_buffer_size` | 增大 `out_buffer_size`（驱动会自动拆分多次传输） |
| CH34x/CP210x 枚举异常 | 用了通用 CDC-ACM 而非 vendor 驱动 | 用对应 `usb_host_*_vcp` 子组件 + `cdc_acm_host_open_vendor_specific` |
| 卸载前 host 仍占用 | 驱动内部 client 未注销 | 确保所有设备 `close` 后再 `uninstall` |

## 参考

- `host/class/cdc/usb_host_cdc_acm/include/usb/cdc_acm_host.h` — 驱动 API 与 `cdc_acm_host_open` 宏
- `host/class/cdc/usb_host_cdc_acm/include/usb/cdc_host_types.h` — 配置结构、回调、事件
- `host/class/cdc/usb_host_ch34x_vcp/include/usb/vcp_ch34x.h` 等 — vendor VCP
- ESP-IDF 示例（仓库文档引用）：`peripherals/usb/host/cdc`、`esp_modem` examples（蜂窝 modem）
