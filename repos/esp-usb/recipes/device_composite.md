# USB 设备：复合设备（CDC + MSC）

> **适用摘要**: 让 ESP 芯片同时作为 USB 串口和大容量存储设备（composite），使用接口关联描述符（IAD）。

> Version: selected repo/component version; align with the user project and dependency manifest.
> Evidence: `repos/esp-usb/resources/`, source/examples in `repos/esp-usb/`, and this recipe path `repos/esp-usb/recipes/device_composite.md`.
> Validation: draft metadata added from repository routing; verify APIs, Kconfig symbols, and component dependencies against the selected version.

## 触发意图

- "USB 复合设备"
- "composite device"
- "同时 CDC + MSC"
- "USB serial + mass storage"
- "IAD descriptor"

## 前置条件

| 条件 | 要求 |
|---|---|
| 组件依赖 | `idf.py add-dependency "espressif/esp_tinyusb^2.2.0"` |
| menuconfig | `CONFIG_TINYUSB_CDC_ENABLED=y` 且 `CONFIG_TINYUSB_MSC_ENABLED=y` |
| 参考示例 | `device/esp_tinyusb/test_apps/cdc/main/test_cdc.c`（描述符构造方式） |

## 分步说明

### 1. 复合设备描述符（IAD）

复合设备必须把设备类设为 `TUSB_CLASS_MISC`、子类 `MISC_SUBCLASS_COMMON`、协议 `MISC_PROTOCOL_IAD`，这样主机才会按接口关联描述符枚举每个功能。

```c
#include "tinyusb.h"
#include "tusb_config.h"

static const tusb_desc_device_t composite_device_descriptor = {
    .bLength            = sizeof(composite_device_descriptor),
    .bDescriptorType    = TUSB_DESC_DEVICE,
    .bcdUSB             = 0x0200,
    .bDeviceClass       = TUSB_CLASS_MISC,
    .bDeviceSubClass    = MISC_SUBCLASS_COMMON,
    .bDeviceProtocol    = MISC_PROTOCOL_IAD,
    .bMaxPacketSize0    = CFG_TUD_ENDPOINT0_SIZE,
    .idVendor           = TINYUSB_ESPRESSIF_VID,
    .idProduct          = 0x4002,
    .bcdDevice          = 0x0100,
    .iManufacturer      = 0x01,
    .iProduct           = 0x02,
    .iSerialNumber      = 0x03,
    .bNumConfigurations = 0x01,
};
```

### 2. 复合配置描述符

用 TinyUSB 的 `TUD_CDC_DESCRIPTOR` 与 `TUD_MSC_DESCRIPTOR` 宏拼接。CDC 与 MSC 的端点地址不能冲突。

```c
// 1 个 CDC + 1 个 MSC
#define CONFIG_TOTAL_LEN  (TUD_CONFIG_DESC_LEN + TUD_CDC_DESC_LEN + TUD_MSC_DESC_LEN)

// EP: CDC notify=0x81, CDC out=0x02, CDC in=0x82, MSC out=0x03, MSC in=0x83
static const uint8_t composite_config_descriptor[] = {
    // Config number, interface count, string idx, total len, attr, max power
    TUD_CONFIG_DESCRIPTOR(1, 4, 0, CONFIG_TOTAL_LEN,
                          TUSB_DESC_CONFIG_ATT_REMOTE_WAKEUP, 100),

    // CDC: itf 0-1, string 4, notify EP 0x81 size 8, out EP 0x02, in EP 0x82, size
    TUD_CDC_DESCRIPTOR(0, 4, 0x81, 8, 0x02, 0x82, (TUD_OPT_HIGH_SPEED ? 512 : 64)),

    // MSC: itf 2, string 5, out EP 0x03, in EP 0x83, size
    TUD_MSC_DESCRIPTOR(2, 5, 0x03, 0x83, (TUD_OPT_HIGH_SPEED ? 512 : 64)),
};
```

> 端点大小：HS 目标用 512，FS 目标用 64。`TUD_OPT_HIGH_SPEED` 在 P4/S31 上为真。

### 3. 安装驱动（注入描述符）

```c
#include "tinyusb_default_config.h"
#include "tinyusb.h"

tinyusb_config_t tusb_cfg = TINYUSB_DEFAULT_CONFIG();
tusb_cfg.descriptor.device = &composite_device_descriptor;
tusb_cfg.descriptor.full_speed_config = composite_config_descriptor;
#if (TUD_OPT_HIGH_SPEED)
tusb_cfg.descriptor.high_speed_config = composite_config_descriptor;
tusb_cfg.descriptor.qualifier = &device_qualifier;   // 见下
#endif
ESP_ERROR_CHECK(tinyusb_driver_install(&tusb_cfg));
```

### 4. 高速目标的 qualifier（P4/S31）

```c
static const tusb_desc_device_qualifier_t device_qualifier = {
    .bLength            = sizeof(tusb_desc_device_qualifier_t),
    .bDescriptorType    = TUSB_DESC_DEVICE_QUALIFIER,
    .bcdUSB             = 0x0200,
    .bDeviceClass       = TUSB_CLASS_MISC,
    .bDeviceSubClass    = MISC_SUBCLASS_COMMON,
    .bDeviceProtocol    = MISC_PROTOCOL_IAD,
    .bMaxPacketSize0    = CFG_TUD_ENDPOINT0_SIZE,
    .bNumConfigurations = 0x01,
    .bReserved          = 0,
};
```

### 5. 初始化两个类

```c
#include "tinyusb_cdc_acm.h"
#include "tinyusb_msc.h"

// CDC
const tinyusb_config_cdcacm_t acm_cfg = {
    .cdc_port = TINYUSB_CDC_ACM_0,
    .callback_rx = NULL,
    .callback_rx_wanted_char = NULL,
    .callback_line_state_changed = NULL,
    .callback_line_coding_changed = NULL,
};
ESP_ERROR_CHECK(tinyusb_cdcacm_init(&acm_cfg));

// MSC（介质初始化见 device_msc_storage.md）
tinyusb_msc_storage_handle_t storage_hdl;
const tinyusb_msc_storage_config_t msc_cfg = {
    .medium.wl_handle = wl_handle,
};
ESP_ERROR_CHECK(tinyusb_msc_new_storage_spiflash(&msc_cfg, &storage_hdl));
```

## 常见错误

| 错误 | 原因 | 解决方法 |
|---|---|---|
| Windows 只识别出其中一个功能 | 设备类没用 IAD | `bDeviceClass=TUSB_CLASS_MISC`、`MISC_SUBCLASS_COMMON`、`MISC_PROTOCOL_IAD` |
| 枚举失败 / 主机报错 | CDC 与 MSC 端点地址冲突 | 每个端点地址唯一；通知 EP 用 IN（0x8x），数据 EP 按方向 |
| HS 目标枚举不稳定 | 缺 qualifier 或 high_speed_config | 同时提供两套速度描述符 + qualifier |
| 接口数与配置描述符不匹配 | `TUD_CONFIG_DESCRIPTOR` 的 interface count 写错 | 总接口数 = CDC(2) + MSC(1) = 3（按实际宏统计） |
| 编译报 `CFG_TUD_MSC` 为 0 | `CONFIG_TINYUSB_MSC_ENABLED` 未开 | menuconfig 同时开 CDC 和 MSC |

## 参考

- `device/esp_tinyusb/include/tinyusb.h` — `tinyusb_config_t`、descriptor 字段
- `device/esp_tinyusb/test_apps/cdc/main/test_cdc.c` — 描述符 + `TUD_*_DESCRIPTOR` 用法
- TinyUSB 提供的宏：`TUD_CONFIG_DESCRIPTOR`、`TUD_CDC_DESCRIPTOR`、`TUD_MSC_DESCRIPTOR`、`TUD_HID_DESCRIPTOR`
- ESP-IDF 示例（仓库文档引用）：`peripherals/usb/device/tusb_composite_msc_serialdevice`
