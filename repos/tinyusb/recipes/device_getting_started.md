# 设备栈从零集成

> **适用摘要**: 在自定义固件中集成 TinyUSB 设备栈，完成 board_init → tusb_init → tud_task 主循环与必需的描述符回调，使设备可被主机枚举。

> Version: selected repo/component version; align with the user project and dependency manifest.
> Evidence: `repos/tinyusb/resources/`, source/examples in `repos/tinyusb/`, and this recipe path `repos/tinyusb/recipes/device_getting_started.md`.
> Validation: draft metadata added from repository routing; verify APIs, Kconfig symbols, and component dependencies against the selected version.

## 触发意图

- "TinyUSB 设备初始化"
- "怎么用 TinyUSB"
- "tud_task 在哪里调用"
- "USB 中断怎么转发给 TinyUSB"
- "TinyUSB 集成到我的项目"

## 前置条件

| 条件 | 要求 |
|---|---|
| 参考示例 | `examples/device/cdc_msc/src/main.c` |
| 参考文档 | `docs/integration.rst`、`docs/getting_started.rst` |
| 必需文件 | `tusb_config.h`（含 `CFG_TUSB_MCU` 等）、`usb_descriptors.c`（设备侧） |

## 分步说明

### 1. 最小 `tusb_config.h`

```c
#ifndef TUSB_CONFIG_H_
#define TUSB_CONFIG_H_

// 由 board.mk / CMake 注入，否则编译报 #error
#ifndef CFG_TUSB_MCU
#error CFG_TUSB_MCU must be defined   // 例如 OPT_MCU_ESP32S3 / OPT_MCU_STM32H7
#endif

#define CFG_TUSB_OS               OPT_OS_NONE   // OPT_OS_FREERTOS / RTTHREAD / PICO / ZEPHYR
#define CFG_TUSB_DEBUG            0

#define CFG_TUD_ENABLED           1             // 使能设备栈

#ifndef CFG_TUD_ENDPOINT0_SIZE
#define CFG_TUD_ENDPOINT0_SIZE    64
#endif

// 先开一个 CDC（其余类置 0）
#define CFG_TUD_CDC               1
#define CFG_TUD_MSC               0
#define CFG_TUD_HID               0

#define CFG_TUD_CDC_RX_BUFSIZE    (TUD_OPT_HIGH_SPEED ? 512 : 64)
#define CFG_TUD_CDC_TX_BUFSIZE    (TUD_OPT_HIGH_SPEED ? 512 : 64)
#define CFG_TUD_CDC_EP_BUFSIZE    (TUD_OPT_HIGH_SPEED ? 512 : 64)

#endif
```

### 2. `main.c` 标准骨架

```c
#include "bsp/board_api.h"
#include "tusb.h"

int main(void) {
  board_init();

  // 显式传 rhport + 角色 + 速度（跨板通用，优于无参 tusb_init()）
  tusb_rhport_init_t dev_init = {
    .role  = TUSB_ROLE_DEVICE,
    .speed = TUSB_SPEED_AUTO
  };
  tusb_init(BOARD_TUD_RHPORT, &dev_init);

  board_init_after_tusb();

  while (1) {
    tud_task();          // 必须频繁调用（建议 <1ms）
    // app_task();
  }
}

// USB 中断转发到栈（函数名按芯片向量表命名）
void USB0_IRQHandler(void) {
  tusb_int_handler(BOARD_TUD_RHPORT, true);
}
```

### 3. 必需的描述符回调（在 `usb_descriptors.c`）

```c
#include "tusb.h"

#define USB_VID 0xCafe
#define USB_BCD 0x0200

static tusb_desc_device_t const desc_device = {
    .bLength            = sizeof(tusb_desc_device_t),
    .bDescriptorType    = TUSB_DESC_DEVICE,
    .bcdUSB             = USB_BCD,
    .bDeviceClass       = 0x00,   // 由接口描述符定义类
    .bMaxPacketSize0    = CFG_TUD_ENDPOINT0_SIZE,
    .idVendor           = USB_VID,
    .idProduct          = 0x4001,
    .bcdDevice          = 0x0100,
    .iManufacturer      = 0x01,
    .iProduct           = 0x02,
    .iSerialNumber      = 0x03,
    .bNumConfigurations = 0x01
};

uint8_t const *tud_descriptor_device_cb(void) {
  return (uint8_t const *)&desc_device;
}

// 字符串描述符（数组里 0=langid，后续为厂商/产品/序列号）
uint16_t const *tud_descriptor_string_cb(uint8_t index, uint16_t langid) {
  (void)langid;
  // ... 返回 UTF-16LE 字符串描述符指针
  return NULL;
}
```

> 配置描述符 `tud_descriptor_configuration_cb()` 的具体写法见 `recipes/descriptors_config.md`。

### 4. 设备状态回调（可选但推荐）

```c
void tud_mount_cb(void)      { /* 配置被主机选中 */ }
void tud_umount_cb(void)     { /* 设备配置被取消 */ }
void tud_suspend_cb(bool remote_wakeup_en) { (void)remote_wakeup_en; }
void tud_resume_cb(void)     { }
```

## 常见错误

| 错误 | 原因 | 解决方法 |
|---|---|---|
| 设备完全识别不到 | 主循环没调 `tud_task()` | 在 while(1) 里加 `tud_task()`，确保被频繁执行 |
| 枚举一半失败 | USB ISR 没转发 | 在 USB 中断函数里调 `tusb_int_handler(rhport, true)` |
| 编译报 `CFG_TUSB_MCU must be defined` | 未定义 MCU 宏 | 由 board.mk / CMake 注入，或 `#define CFG_TUSB_MCU OPT_MCU_*` |
| `tusb_init()` 无参版报错 | 未定义 `CFG_TUSB_RHPORT0/1_MODE` | 改用显式 `tusb_init(rhport, &init)` 形式 |
| `tud_descriptor_*_cb` 链接报重复定义 | 回调名写错或重复实现 | 回调函数名必须与头文件声明一致，全项目唯一 |

## 参考

- `examples/device/cdc_msc/src/main.c` — 标准设备主循环
- `examples/device/cdc_msc/src/tusb_config.h` — 配置模板
- `docs/integration.rst` — 官方集成指南（含 STM32CubeIDE 集成）
- `docs/getting_started.rst` — 快速开始
