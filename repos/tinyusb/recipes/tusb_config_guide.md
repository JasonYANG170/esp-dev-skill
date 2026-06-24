# tusb_config.h 配置指南

> **适用摘要**: 系统说明 TinyUSB 核心配置宏（`CFG_TUSB_*` / `CFG_TUD_*` / `CFG_TUH_*`）：MCU、OS、类使能、缓冲区、端点 0、速度、内存对齐。

## 触发意图

- "tusb_config.h 怎么配"
- "CFG_TUSB_MCU / CFG_TUSB_OS"
- "使能哪些类（CFG_TUD_*）"
- "端点缓冲区大小"
- "TinyUSB 高速/全速配置"

## 前置条件

| 条件 | 要求 |
|---|---|
| 参考示例 | 任一 `examples/device/*/src/tusb_config.h` |
| 参考源码 | `src/tusb_option.h`（所有 `CFG_*` 默认值与 `OPT_*` 定义） |

## 分步说明

### 1. 三大基础宏（必须）

```c
#ifndef CFG_TUSB_MCU
#error CFG_TUSB_MCU must be defined      // 由 board.mk / CMake 注入
#endif

#define CFG_TUSB_OS        OPT_OS_NONE    // OPT_OS_NONE/FREERTOS/RTTHREAD/MYNEWT/PICO/CUSTOM/RTX4/ZEPHYR
#define CFG_TUSB_DEBUG     0              // 0 关闭，1/2/3 越来越详细
```

### 2. 设备/主机使能与速度

```c
// 设备栈
#define CFG_TUD_ENABLED    1
#define CFG_TUD_MAX_SPEED  BOARD_TUD_MAX_SPEED   // 通常 OPT_MODE_DEFAULT_SPEED

// 主机栈
#define CFG_TUH_ENABLED    1
#define CFG_TUH_MAX_SPEED  BOARD_TUH_MAX_SPEED
```

> 板级默认 `BOARD_TUD_RHPORT` / `BOARD_TUD_MAX_SPEED` 由 `board.mk` 提供，示例 config.h 用 `#ifndef` 兜底。

### 3. 设备类实例数与缓冲

```c
#define CFG_TUD_ENDPOINT0_SIZE   64

#define CFG_TUD_CDC              1
#define CFG_TUD_CDC_NOTIFY       1
#define CFG_TUD_CDC_RX_BUFSIZE   (TUD_OPT_HIGH_SPEED ? 512 : 64)
#define CFG_TUD_CDC_TX_BUFSIZE   (TUD_OPT_HIGH_SPEED ? 512 : 64)
#define CFG_TUD_CDC_EP_BUFSIZE   (TUD_OPT_HIGH_SPEED ? 512 : 64)

#define CFG_TUD_MSC              1
#define CFG_TUD_MSC_EP_BUFSIZE   512

#define CFG_TUD_HID              1
#define CFG_TUD_HID_EP_BUFSIZE   64

#define CFG_TUD_VENDOR           0
#define CFG_TUD_MIDI             0
// 其余类（AUDIO/DFU/MTP/VIDEO/BTH）按需开
```

> `TUD_OPT_HIGH_SPEED` 由栈据 `CFG_TUD_MAX_SPEED` 自动求出，可用来按速度切缓冲大小。

### 4. 主机类开关

```c
#define CFG_TUH_HUB             1
#define CFG_TUH_CDC             1
#define CFG_TUH_MSC             1
#define CFG_TUH_HID             1
#define CFG_TUH_VENDOR          0
// ESP32 主机：CFG_TUH_MAX3421=1；RP2040 PIO-USB：CFG_TUH_RPI_PIO_USB=1
```

### 5. 内存对齐（USB DMA 专用 SRAM）

```c
#ifndef CFG_TUSB_MEM_SECTION
#define CFG_TUSB_MEM_SECTION            // 如 __attribute__((section(".usb_ram")))
#endif
#ifndef CFG_TUSB_MEM_ALIGN
#define CFG_TUSB_MEM_ALIGN              __attribute__((aligned(4)))
#endif
// D-Cache 相关（部分高端 MCU）
// CFG_TUD_MEM_DCACHE_ENABLE, CFG_TUSB_MEM_DCACHE_LINE_SIZE
```

### 6. 速度模式宏（`OPT_MODE_*`，来自 `src/tusb_option.h`）

| 宏 | 含义 |
|---|---|
| `OPT_MODE_DEFAULT_SPEED` | 默认（MCU 支持的最大速度） |
| `OPT_MODE_LOW_SPEED` | 低速 1.5Mbps |
| `OPT_MODE_FULL_SPEED` | 全速 12Mbps |
| `OPT_MODE_HIGH_SPEED` | 高速 480Mbps |
| `OPT_MODE_DEVICE` / `OPT_MODE_HOST` | 设备/主机角色位 |

### 7. OS 选项（`OPT_OS_*`）

| 宏 | OS |
|---|---|
| `OPT_OS_NONE` | 裸机 |
| `OPT_OS_FREERTOS` | FreeRTOS |
| `OPT_OS_RTTHREAD` | RT-Thread |
| `OPT_OS_MYNEWT` | Mynewt |
| `OPT_OS_PICO` | Raspberry Pi Pico SDK |
| `OPT_OS_ZEPHYR` | Zephyr |
| `OPT_OS_CUSTOM` | 自定义 OSAL（应用实现） |
| `OPT_OS_RTX4` | Keil RTX4 |

## 常见错误

| 错误 | 原因 | 解决方法 |
|---|---|---|
| `#error CFG_TUSB_MCU must be defined` | 未注入 MCU 宏 | board.mk / CMake 设 `CFG_TUSB_MCU=OPT_MCU_*` |
| 高速设备 RAM 爆 | 全速缓冲用在高速 | 用 `TUD_OPT_HIGH_SPEED ? 512 : 64` 动态设 |
| 类未生效 | 实例数为 0 | 置非 0，并在描述符里加对应接口 |
| FreeRTOS 下栈异常 | OS 选错 | `CFG_TUSB_OS=OPT_OS_FREERTOS`，并在高优先级任务里跑 `tud_task` |
| 端点 0 包丢失 | `CFG_TUD_ENDPOINT0_SIZE` 过小 | 默认 64；低速/部分 MCU 用 8 |

## 参考

- `examples/device/cdc_msc/src/tusb_config.h` — 完整配置模板
- `src/tusb_option.h` — 所有 `CFG_*` 默认值与 `OPT_*` 定义
- `docs/reference/usb_concepts.rst` — 速度/类使能说明
