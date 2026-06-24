# Vendor 类与 WebUSB

> **适用摘要**: 用 TinyUSB Vendor 类实现厂商自定义 USB 通信（含 WebUSB/WinUSB），包括缓冲模式与零缓冲直通模式、收发 API、WebUSB URL 描述符。

## 触发意图

- "Vendor class"
- "自定义 USB 设备"
- "WebUSB"
- "tud_vendor_read / tud_vendor_write"
- "WinUSB / MS OS 2.0"

## 前置条件

| 条件 | 要求 |
|---|---|
| 参考示例 | `examples/device/webusb_serial/`、`examples/device/hid_generic_inout/` |
| 配置项 | `CFG_TUD_VENDOR=1`、`CFG_TUD_VENDOR_RX_BUFSIZE`/`TX_BUFSIZE` |

## 分步说明

### 1. tusb_config.h 使能 Vendor（缓冲模式）

```c
#define CFG_TUD_VENDOR             1
#define CFG_TUD_VENDOR_RX_BUFSIZE  (TUD_OPT_HIGH_SPEED ? 512 : 64)
#define CFG_TUD_VENDOR_TX_BUFSIZE  (TUD_OPT_HIGH_SPEED ? 512 : 64)
```

### 2. 缓冲模式收发（类似 CDC）

```c
void vendor_task(void) {
  if (tud_vendor_available()) {
    uint8_t buf[64];
    uint32_t n = tud_vendor_read(buf, sizeof(buf));
    // 回显
    tud_vendor_write(buf, n);
  }
}

// 收到数据回调（任务上下文）
void tud_vendor_rx_cb(uint8_t itf) {
  (void)itf;
}
```

### 3. 零缓冲直通模式（数据直达回调）

```c
// tusb_config.h
#define CFG_TUD_VENDOR_RX_BUFSIZE  0   // 关闭内部缓冲
// 此模式下 tud_vendor_read()/write() 不可用，数据在回调里直接处理

void tud_vendor_rx_cb(uint8_t itf, uint8_t const *buffer, uint16_t bufsize) {
  (void)itf;
  // buffer 指向端点原始数据，bufsize 为本次接收字节数
  // 在此处理；注意此回调仍在任务上下文
}
```

### 4. WebUSB：URL 描述符 + landing page

WebUSB 用 vendor 接口承载，并通过 BOS 描述符声明 `WEBUSB_URL`。参考 `examples/device/webusb_serial/`：

```c
// BOS 描述符回调（声明平台能力，含 WebUSB）
uint8_t const *tud_descriptor_bos_cb(void) {
  return desc_bos;
}

// WebUSB 供应商请求回调（GET URL 等）
bool tud_vendor_control_xfer_cb(uint8_t rhport, uint8_t stage,
                                tusb_control_request_t const *request) {
  // 处理 WebUSB vendor request（bRequest = 0x01 等）
  return true;
}
```

### 5. 配置描述符里的 Vendor 接口

```c
// Vendor: itf, str, epout, epin, epsize
TUD_VENDOR_DESCRIPTOR(ITF_NUM_VENDOR, 0, EPNUM_VENDOR_OUT, EPNUM_VENDOR_IN, 64)
```

## 常见错误

| 错误 | 原因 | 解决方法 |
|---|---|---|
| `tud_vendor_read/write` 链接失败 | 缓冲设为 0 | 直通模式下不要用 read/write，改在 `tud_vendor_rx_cb` 里处理；或恢复非 0 缓冲 |
| WebUSB 不识别 | 缺 BOS / vendor request 回调 | 实现 `tud_descriptor_bos_cb` 与 `tud_vendor_control_xfer_cb` |
| 数据丢包 | 端点未就绪即写 | 缓冲模式用 `tud_vendor_available()` 查询再处理 |
| 主机加载错驱动 | 缺 MS OS 2.0 兼容描述符 | 参考示例声明 WinUSB 兼容 ID |

## 参考

- `examples/device/webusb_serial/` — WebUSB + vendor 串口
- `examples/device/hid_generic_inout/` — 通用 vendor 风格 IN/OUT
- `src/class/vendor/vendor_device.h` — Vendor 类 API
- `docs/reference/usb_concepts.rst` — Vendor 类缓冲模式说明（含零缓冲）
