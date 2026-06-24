# TinyUSB 配置参考（tusb_config.h / tusb_option.h）

> 所有宏来自 `src/tusb_option.h` 与示例 `tusb_config.h`。`tusb_option.h` 给出默认值与 `OPT_*` 枚举。

## 基础必填

| 宏 | 说明 | 取值 |
|---|---|---|
| `CFG_TUSB_MCU` | 目标 MCU（必须，否则 `#error`） | `OPT_MCU_*`（见下表节选） |
| `CFG_TUSB_OS` | RTOS 选择 | `OPT_OS_*` |
| `CFG_TUSB_DEBUG` | 日志级别 | 0（关）/1/2/3 |

### `OPT_OS_*`（OS 选项）

| 宏 | 含义 |
|---|---|
| `OPT_OS_NONE` (1) | 裸机 |
| `OPT_OS_FREERTOS` (2) | FreeRTOS |
| `OPT_OS_MYNEWT` (3) | Mynewt |
| `OPT_OS_CUSTOM` (4) | 自定义 OSAL |
| `OPT_OS_PICO` (5) | Raspberry Pi Pico SDK |
| `OPT_OS_RTTHREAD` (6) | RT-Thread |
| `OPT_OS_RTX4` (7) | Keil RTX 4 |
| `OPT_OS_ZEPHYR` (8) | Zephyr |

### `OPT_MODE_*`（速度/角色位）

| 宏 | 值 | 含义 |
|---|---|---|
| `OPT_MODE_NONE` | 0x0000 | 禁用 |
| `OPT_MODE_DEVICE` | 0x0001 | 设备角色 |
| `OPT_MODE_HOST` | 0x0002 | 主机角色 |
| `OPT_MODE_DEFAULT_SPEED` | 0x0000 | 默认最大速度 |
| `OPT_MODE_LOW_SPEED` | 0x0100 | 低速 1.5Mbps |
| `OPT_MODE_FULL_SPEED` | 0x0200 | 全速 12Mbps |
| `OPT_MODE_HIGH_SPEED` | 0x0400 | 高速 480Mbps |
| `OPT_MODE_SPEED_MASK` | 0xff00 | 速度位掩码 |

> 注：USB 3.0 SuperSpeed（5Gbps）不受支持。

## `OPT_MCU_*`（节选，来自 `src/tusb_option.h`）

| MCU 宏 | 芯片 |
|---|---|
| `OPT_MCU_ESP32S2` (900) | Espressif ESP32-S2 |
| `OPT_MCU_ESP32S3` (901) | Espressif ESP32-S3 |
| `OPT_MCU_ESP32` (902) | Espressif ESP32（主机 max3421e） |
| `OPT_MCU_ESP32C3` (903) | Espressif ESP32-C3 |
| `OPT_MCU_ESP32C6` (904) | Espressif ESP32-C6 |
| `OPT_MCU_ESP32C2` (905) | Espressif ESP32-C2 |
| `OPT_MCU_ESP32H2` (906) | Espressif ESP32-H2 |
| `OPT_MCU_ESP32P4` (907) | Espressif ESP32-P4 |
| `OPT_MCU_ESP32C5` (908) | Espressif ESP32-C5 |
| `OPT_MCU_ESP32C61` (909) | Espressif ESP32-C61 |
| `OPT_MCU_RP2040` | Raspberry Pi RP2040 |
| `OPT_MCU_STM32F4` / `F7` / `H7` | STM32（dwc2，支持高速） |
| `OPT_MCU_STM32G0` / `G4` / `L4` | STM32（fsdev） |
| `OPT_MCU_NRF5X` | Nordic nRF52840/5340 |
| `OPT_MCU_LPC55XX` / `LPC54XXX` | NXP LPC55 |
| `OPT_MCU_IMXRT` | NXP i.MX RT |
| `OPT_MCU_SAMD21` / `SAMD51` | Microchip SAMD |

## 设备栈开关与缓冲

| 宏 | 默认 | 说明 |
|---|---|---|
| `CFG_TUD_ENABLED` | （由 RHPORT 推导） | 设备栈总开关，置 1 使能 |
| `CFG_TUD_MAX_SPEED` | `BOARD_TUD_MAX_SPEED` | 设备最大速度 |
| `CFG_TUD_ENDPOINT0_SIZE` | 64 | 端点 0 包大小 |
| `CFG_TUD_CDC` | — | CDC 实例数 |
| `CFG_TUD_CDC_NOTIFY` | — | CDC 通知端点（1 开） |
| `CFG_TUD_CDC_RX_BUFSIZE` | — | CDC RX FIFO |
| `CFG_TUD_CDC_TX_BUFSIZE` | — | CDC TX FIFO |
| `CFG_TUD_CDC_EP_BUFSIZE` | — | CDC 端点缓冲 |
| `CFG_TUD_MSC` | — | MSC 实例数 |
| `CFG_TUD_MSC_EP_BUFSIZE` | — | MSC 端点缓冲（建议 512） |
| `CFG_TUD_HID` | — | HID 实例数 |
| `CFG_TUD_HID_EP_BUFSIZE` | — | HID 端点缓冲 |
| `CFG_TUD_MIDI` | — | MIDI 实例数 |
| `CFG_TUD_AUDIO` | — | Audio（UAC2）实例数 |
| `CFG_TUD_DFU` / `CFG_TUD_DFU_RUNTIME` | — | DFU |
| `CFG_TUD_VENDOR` | — | Vendor 实例数 |
| `CFG_TUD_VENDOR_RX_BUFSIZE` / `TX_BUFSIZE` | — | 设为 0 关闭内部缓冲（直通模式） |
| `CFG_TUD_BTH` | — | 蓝牙 HCI |
| `CFG_TUD_MTP` | — | MTP |
| `CFG_TUD_VIDEO` | — | UVC（WIP） |
| `CFG_TUD_USBTMC` | 0 | USBTMC 测试测量类（`CFG_TUD_USBTMC_ENABLE_488=1` 开 USB488，`CFG_TUD_USBTMC_ENABLE_INT_EP=1` 开中断端点） |
| `CFG_TUD_DFU_XFER_BUFSIZE` | 必填 | DFU 模式传输缓冲，须等于描述符 `_xfer_size` |
| `CFG_TUD_AUDIO_ENABLE_EP_IN/OUT` | 0 | Audio IN(TX/麦克风)/OUT(RX/扬声器) 端点开关 |
| `CFG_TUD_AUDIO_ENABLE_FEEDBACK_EP` | 0 | Audio 异步反馈端点（异步 sink 必需） |
| `CFG_TUD_AUDIO_FUNC_1_EP_IN/OUT_SZ_MAX` | 必填 | Audio 各功能最大端点尺寸（须 `TUD_AUDIO_EP_SIZE` 算） |
| `CFG_TUD_AUDIO_FUNC_1_EP_IN/OUT_SW_BUF_SZ` | 0 | Audio 软件缓冲（FIFO_COUNT 法需 ≥ 4×EP size） |

> `TUD_OPT_HIGH_SPEED`：由 `CFG_TUD_MAX_SPEED` 自动求出，可用来按速度切缓冲。

## Type-C / Power Delivery 开关

| 宏 | 默认 | 说明 |
|---|---|---|
| `CFG_TUC_ENABLED` | 0 | Type-C/PD 栈总开关（独立于 `CFG_TUD_*`/`CFG_TUH_*`，WIP 仅 STM32 G4） |
| `CFG_TUC_TASK_QUEUE_SZ` | 8 | PD 任务队列深度 |

## 主机栈开关

| 宏 | 默认 | 说明 |
|---|---|---|
| `CFG_TUH_ENABLED` | （由 RHPORT 推导） | 主机栈总开关 |
| `CFG_TUH_MAX_SPEED` | `BOARD_TUH_MAX_SPEED` | 主机最大速度 |
| `CFG_TUH_HUB` | — | Hub 支持 |
| `CFG_TUH_CDC` | — | CDC-ACM 主机 |
| `CFG_TUH_MSC` | — | MSC 主机 |
| `CFG_TUH_HID` | — | HID 主机 |
| `CFG_TUH_VENDOR` | — | Vendor 主机（FTDI/CP210x/CH34x/PL2303） |
| `CFG_TUH_MAX3421` | 0 | MAX3421E 外置主机控制器（via SPI，如 ESP32 主机） |
| `CFG_TUH_RPI_PIO_USB` | 0 | RP2040 PIO-USB 主机 |

## 内存 / DMA

| 宏 | 默认 | 说明 |
|---|---|---|
| `CFG_TUSB_MEM_SECTION` | （空） | USB DMA 专用段，如 `__attribute__((section(".usb_ram")))` |
| `CFG_TUSB_MEM_ALIGN` | `__attribute__((aligned(4)))` | 对齐 |
| `CFG_TUSB_MEM_DCACHE_LINE_SIZE` | 1 | D-Cache 行大小（高端 MCU） |
| `CFG_TUD_MEM_DCACHE_ENABLE` | 0 | 设备 DMA D-Cache 维护 |
| `CFG_TUSB_OS_INC_PATH` | — | RTOS 头文件包含路径 |

## DWC2 / CI HS（部分 MCU 的控制器调优）

| 宏 | 默认 | 说明 |
|---|---|---|
| `CFG_TUD_DWC2_SLAVE_ENABLE` | 1 | DWC2 从机模式 |
| `CFG_TUD_DWC2_DMA_ENABLE` | 0 | DWC2 DMA 模式 |
| `CFG_TUH_DWC2_SLAVE_ENABLE` | 1 | DWC2 主机从机模式 |
| `CFG_TUH_DWC2_DMA_ENABLE` | 0 | DWC2 主机 DMA |
| `CFG_TUD_CI_HS_VBUS_CHARGE` | 0 | CI HS VBUS 充电 |

## 远程唤醒

| 宏 | 说明 |
|---|---|
| `CFG_TUD_USBD_ENABLE_REMOTE_WAKEUP` | 使能设备远程唤醒（配合 `tud_remote_wakeup()`） |

> 精确默认值以 `src/tusb_option.h` 为准；本表仅汇总高频项。
