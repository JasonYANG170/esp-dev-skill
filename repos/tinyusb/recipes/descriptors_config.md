# USB 描述符编写

> **适用摘要**: 编写 TinyUSB 设备侧描述符回调：设备/配置/字符串描述符，使用 `TUD_*_DESCRIPTOR` 宏组装配置描述符，正确分配接口号与端点号。

> Version: selected repo/component version; align with the user project and dependency manifest.
> Evidence: `repos/tinyusb/resources/`, source/examples in `repos/tinyusb/`, and this recipe path `repos/tinyusb/recipes/descriptors_config.md`.
> Validation: draft metadata added from repository routing; verify APIs, Kconfig symbols, and component dependencies against the selected version.

## 触发意图

- "usb_descriptors.c 怎么写"
- "TinyUSB 配置描述符"
- "TUD_CDC_DESCRIPTOR / TUD_MSC_DESCRIPTOR"
- "接口号/端点号怎么分配"
- "USB 字符串描述符"

## 前置条件

| 条件 | 要求 |
|---|---|
| 参考示例 | `examples/device/cdc_msc/src/usb_descriptors.c`、`examples/device/hid_composite/src/usb_descriptors.c` |
| 总开关 | `CFG_TUD_*` 决定包含哪些类描述符 |

## 分步说明

### 1. 设备描述符 + device_cb

```c
#define USB_VID 0xCafe
#define USB_BCD 0x0200

// 多类组合 PID 位图（避免与现有组合冲突）
#define PID_MAP(itf, n) ((CFG_TUD_##itf) ? (1 << (n)) : 0)
#define USB_PID (0x4000 | PID_MAP(CDC,0) | PID_MAP(MSC,1) | PID_MAP(HID,2) | PID_MAP(MIDI,3) | PID_MAP(VENDOR,4))

static tusb_desc_device_t const desc_device = {
    .bLength            = sizeof(tusb_desc_device_t),
    .bDescriptorType    = TUSB_DESC_DEVICE,
    .bcdUSB             = USB_BCD,
    // CDC 需 IAD：device class = MISC, subclass = COMMON, protocol = IAD
    .bDeviceClass       = TUSB_CLASS_MISC,
    .bDeviceSubClass    = MISC_SUBCLASS_COMMON,
    .bDeviceProtocol    = MISC_PROTOCOL_IAD,
    .bMaxPacketSize0    = CFG_TUD_ENDPOINT0_SIZE,
    .idVendor           = USB_VID,
    .idProduct          = USB_PID,
    .bcdDevice          = 0x0100,
    .iManufacturer      = 0x01,
    .iProduct           = 0x02,
    .iSerialNumber      = 0x03,
    .bNumConfigurations = 0x01
};

uint8_t const *tud_descriptor_device_cb(void) {
  return (uint8_t const *)&desc_device;
}
```

### 2. 接口号枚举（CDC 占 2 个接口）

```c
enum {
  ITF_NUM_CDC = 0,
  ITF_NUM_CDC_DATA,   // CDC 数据接口（CDC-ACM 用 IAD 占 2 接口）
  ITF_NUM_MSC,
  ITF_NUM_TOTAL
};
```

### 3. 端点号（设备视角，IN=0x80|n，OUT=n）

```c
// 大多数 MCU 可同号不同向（EP1 IN 与 EP1 OUT 共存）
#define EPNUM_CDC_NOTIF   0x81
#define EPNUM_CDC_OUT     0x02
#define EPNUM_CDC_IN      0x82
#define EPNUM_MSC_OUT     0x03
#define EPNUM_MSC_IN      0x83
```

> 部分控制器（LPC17xx、CXD56，或定义了 `TUD_ENDPOINT_ONE_DIRECTION_ONLY` 的 MCU）端点号方向固定，参考示例里的条件分支。

### 4. 配置描述符用宏拼装

```c
#define CONFIG_TOTAL_LEN  (TUD_CONFIG_DESC_LEN + TUD_CDC_DESC_LEN + TUD_MSC_DESC_LEN)

uint8_t const desc_configuration[] = {
  // config: cfg_num=1, itf_count, str_idx=0, total_len, attr, power(mA/2)
  TUD_CONFIG_DESCRIPTOR(1, ITF_NUM_TOTAL, 0, CONFIG_TOTAL_LEN,
                        TUSB_DESC_CONFIG_ATT_REMOTE_WAKEUP, 100),

  // CDC: itf, str, notif_ep, notif_size, out_ep, in_ep, ep_size
  TUD_CDC_DESCRIPTOR(ITF_NUM_CDC, 4, EPNUM_CDC_NOTIF, 8,
                     EPNUM_CDC_OUT, EPNUM_CDC_IN, 64),

  // MSC: itf, str, out_ep, in_ep, ep_size
  TUD_MSC_DESCRIPTOR(ITF_NUM_MSC, 5, EPNUM_MSC_OUT, EPNUM_MSC_IN, 512)
};

uint8_t const *tud_descriptor_configuration_cb(uint8_t index) {
  (void)index;
  return desc_configuration;
}
```

### 5. HID 接口的配置描述符宏

```c
// 仅 IN 端点（如键盘/鼠标）
TUD_HID_DESCRIPTOR(itf, str_idx, boot_proto, report_desc_len, epin, epsize, ep_interval)
// IN + OUT（通用 inout）
TUD_HID_INOUT_DESCRIPTOR(itf, str_idx, boot_proto, report_desc_len, epout, epin, epsize, ep_interval)
```

### 6. 字符串描述符（UTF-16LE）

```c
char const *string_desc_arr[] = {
  (const char[]) { 0x09, 0x04 },   // 0: English (0x0409)
  "TinyUSB Manufacturer",          // 1: iManufacturer
  "TinyUSB Device",                // 2: iProduct
  "12345678"                       // 3: iSerial
};

uint16_t const *tud_descriptor_string_cb(uint8_t index, uint16_t langid) {
  (void)langid;
  static uint16_t _desc_str[32 + 1];   // +1 for header
  uint8_t chr_count;

  if (index == 0) {
    memcpy(&_desc_str[1], string_desc_arr[0], 2);
    chr_count = 1;
  } else {
    if (index >= sizeof(string_desc_arr)/sizeof(char*)) return NULL;
    const char *str = string_desc_arr[index];
    chr_count = strlen(str);
    if (chr_count > 31) chr_count = 31;
    for (uint8_t i = 0; i < chr_count; i++) {
      _desc_str[1 + i] = str[i];   // ASCII → UTF-16LE
    }
  }

  _desc_str[0] = (uint16_t)((TUSB_DESC_STRING << 8) | (2 * chr_count + 2));
  return _desc_str;
}
```

## 常见错误

| 错误 | 原因 | 解决方法 |
|---|---|---|
| 枚举失败 | `CONFIG_TOTAL_LEN` 与实际长度不符 | 用 `TUD_*_DESC_LEN` 求和，不可硬编码 |
| 接口重叠 | CDC 漏算数据接口 | CDC-ACM 占 2 接口（IAD），`ITF_NUM_TOTAL` 含两者 |
| 主机识别不到类 | device class 与接口类不一致 | CDC 用 `TUSB_CLASS_MISC` + IAD；单类可用 0x00 由接口定义 |
| 字符串乱码 | 未返回 UTF-16LE 或头长度错 | 首字节 = `TUSB_DESC_STRING<<8 \| (2*chr+2)` |
| 端点冲突 | MCU 不支持同号双向 | 用示例里 `TUD_ENDPOINT_ONE_DIRECTION_ONLY` 分支 |
| VID/PID 冲突 | 与已有组合复用 | 用 `PID_MAP()` 位图生成 PID |

## 参考

- `examples/device/cdc_msc/src/usb_descriptors.c` — CDC+MSC 组合描述符
- `examples/device/hid_composite/src/usb_descriptors.c` — 多 report HID 描述符
- `src/device/usbd.h` — `TUD_*_DESCRIPTOR` 宏定义
