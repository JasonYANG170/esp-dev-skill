---
name: tinyusb-skill
description: >-
  AI Skill for TinyUSB USB Host/Device stack development. Used when users need to create, modify,
  or debug TinyUSB device or host firmware, including CDC virtual serial, HID, MSC mass storage,
  Audio/MIDI, DFU, vendor class, descriptors, tusb_config.h, and board bring-up across 50+ MCU
  families (Espressif ESP32-S2/S3/C3/C6/P4, STM32, RP2040, nRF, etc.).
  Trigger words: "TinyUSB", "tinyusb", "tud_task", "tuh_task", "tusb_config", "CDC", "HID", "MSC", "USB device", "USB host", "usb_descriptors", "ESP32", "USB协议栈", "虚拟串口", "U盘"
license: MIT
metadata:
  author: Community
  version: "1.1.0"
---

# tinyusb-skill

面向 TinyUSB USB 协议栈（开源、跨平台 USB Host/Device 栈）的 AI Skill。TinyUSB 以“无动态内存分配 + 中断延迟到任务上下文处理”为核心设计，广泛用于嵌入式 USB 设备与主机开发。本 Skill 提供场景化配方（recipes）、真实 API/配置速查（resources）与高频陷阱（pitfalls），所有 API、宏、配置项、示例路径均来自 TinyUSB 真实源码与文档，绝不臆造。

## Core Principles

1. **绝不臆造 API** — 函数、回调、宏必须能在 `resources/api_reference.md` 或源码头文件中查到；查不到即视为不存在
2. **命名前缀即分层** — `tusb_` 核心栈（init/中断）、`tud_` 设备栈、`tuh_` 主机栈、`tu_` 内部工具（应用一般不用）
3. **回调无需注册** — 实现“规定函数名”（如 `tud_cdc_rx_cb()`、`tud_mount_cb()`）即可被栈自动调用，不要去找注册函数
4. **中断延迟模型** — USB ISR（`tusb_int_handler()`）只做事件入队；真正的协议处理、应用回调都在 `tud_task()`/`tuh_task()` 任务上下文里执行
5. **主循环必须喂任务** — `tud_task()`（设备）与 `tuh_task()`（主机）必须在主循环或高优先级 RTOS 任务里被频繁调用（建议 <1ms），否则设备掉线/枚举失败
6. **`tusb_config.h` 是开关总闸** — `CFG_TUD_*`/`CFG_TUH_*` 决定使能哪些类、`CFG_TUSB_MCU`/`CFG_TUSB_OS` 决定目标与 RTOS，未定义 `CFG_TUSB_MCU` 会编译报错
7. **描述符靠回调提供** — 设备侧须实现 `tud_descriptor_device_cb()`、`tud_descriptor_configuration_cb()`、`tud_descriptor_string_cb()`；可用 `TUD_CDC_DESCRIPTOR()`、`TUD_HID_DESCRIPTOR()` 等宏拼接配置描述符
8. **全静态内存** — 所有端点/类缓冲区在编译期固定，大小由 `CFG_TUD_*_EP_BUFSIZE`、`CFG_TUD_*_RX/TX_BUFSIZE` 控制；改大小即改 RAM 占用
9. **类驱动被栈自动发现** — 不要手写 `cdcd_init()` 调用，类初始化由 `tusb_init()` 内部完成；应用只实现“应用级回调”（如 MSC 的 `tud_msc_read10_cb`）
10. **DCD/HCD 是移植层** — `src/portable/<VENDOR>/<FAMILY>/` 提供 MCU 底层驱动，应用代码只调用 `tud_*`/`tuh_*`，绝不直接调 `dcd_*`/`hcd_*`
11. **VID/PID 与接口组合需唯一** — 同一 VID/PID 不同接口组合会被主机复用旧驱动导致“设备错误”，例子里用 `PID_MAP()` 位图生成 PID
12. **多实例用 `_n` 后缀** — 多路 CDC/HID 用 `tud_cdc_n_write(itf, ...)` 等带实例号的 API，单实例默认 instance 0

## When to Use

**适用：**
- 基于 TinyUSB 创建 USB 设备（CDC 串口、HID 键鼠、MSC U盘、Audio、MIDI、DFU、Vendor）
- 创建 USB 主机（CDC-ACM、MSC、HID、Hub、FTDI/CP210x/CH34x 等串口转换）
- 配置 `tusb_config.h`（使能类、端点缓冲、RTOS、MCU、速度）
- 编写/修改 `usb_descriptors.c`（设备/配置/字符串描述符、接口与端点分配）
- 在自定义固件中集成 TinyUSB（STM32CubeIDE、ESP-IDF、裸机 main loop、FreeRTOS 任务）
- 移植/适配新板卡（BSP）或排查枚举、掉线、传输失败

**不适用：**
- 与 USB 无关的通用嵌入式开发
- 使用 USB Host Controller 之外的 USB-IF 协议（如 USB 3.0 SuperSpeed，TinyUSB 不支持）
- PCB 硬件设计与原理图

---

## Scenario Quick Reference (Recipes)

当用户意图命中下表某个场景时，**先读对应 recipe** — 它包含完整调用链、分步说明、真实代码与常见错误。

### 设备栈（Device Stack）

| recipe | scenario |
|---|---|
| `recipes/device_getting_started.md` | 从零集成 TinyUSB 设备栈：board_init → tusb_init → tud_task 循环 + 描述符回调 |
| `recipes/cdc_virtual_serial.md` | CDC 虚拟串口：`tusb_config.h` 使能 CDC、收发读写（`tud_cdc_read/write`）、`tud_cdc_rx_cb` |
| `recipes/hid_device.md` | HID 设备：键盘/鼠标/游戏手柄 report、`tud_hid_keyboard_report`、report descriptor |
| `recipes/msc_device.md` | MSC U 盘：RAM/Flash 后端、`tud_msc_read10_cb`/`tud_msc_write10_cb`、多 LUN |
| `recipes/descriptors_config.md` | 描述符编写：设备/配置/字符串描述符、`TUD_*_DESCRIPTOR` 宏、接口端点编号 |
| `recipes/vendor_and_webusb.md` | Vendor 类 + WebUSB/WinUSB：缓冲模式、`tud_vendor_read/write`、MS OS 2.0 |
| `recipes/audio_uac2_device.md` | UAC2 音频设备：麦克风（IN）/扬声器（OUT+反馈端点）/耳机、`tud_audio_write/read`、异步 feedback endpoint、采样率协商 |
| `recipes/dfu_device.md` | DFU 固件升级：DFU 模式 vs DFU Runtime、`tud_dfu_download_cb/upload_cb`、多分区（alt）、`tud_dfu_finish_flashing` |
| `recipes/midi_device.md` | MIDI 设备：USB-MIDI 4 字节事件包、cable number、Note On/Off、`tud_midi_packet_read/write` |
| `recipes/usbtmc_device.md` | USBTMC 测试测量仪器（SCPI/VISA）：USB488、`*IDN?`、Bulk-IN/OUT、status byte（STB/MAV/SRQ）、`tud_usbtmc_transmit_dev_msg_data` |
| `recipes/build_and_flash.md` | 编译与烧录：CMake/Make、`BOARD=`、get_deps、jlink/openocd/uf2 target |

### 主机栈（Host Stack）

| recipe | scenario |
|---|---|
| `recipes/host_cdc_msc_hid.md` | USB 主机：`tuh_task`、设备挂载回调、CDC/HID/MSC host API |

### Type-C / Power Delivery

| recipe | scenario |
|---|---|
| `recipes/typec_power_delivery.md` | USB PD 3.0 / Type-C：`tuc_init/task`、PDO 解析、RDO 请求、Accept/PS_READY（WIP，仅 STM32 G4） |

### 配置与故障

| recipe | scenario |
|---|---|
| `recipes/tusb_config_guide.md` | `tusb_config.h` 全量配置指南：MCU/OS/类使能/缓冲区/端点 0 |

---

## 支持芯片速查（节选）

来源：`src/tusb_option.h`（`OPT_MCU_*`）与 `README.rst` 支持列表。仅列举常见系列，完整列表见 `resources/config_reference.md`。

| 厂商 | 系列 | 设备 | 主机 | 高速 | 驱动 | MCU 宏 |
|---|---|---|---|---|---|---|
| Espressif | ESP32-S2 / S3 | ✔ | ✔ | ✖ | dwc2 | `OPT_MCU_ESP32S2` / `OPT_MCU_ESP32S3` |
| Espressif | ESP32-P4 | ✔ | ✔ | ✔ | dwc2 | `OPT_MCU_ESP32P4` |
| Espressif | ESP32-C3 / C6 | ✔ |  | ✖ | dwc2 | `OPT_MCU_ESP32C3` / `OPT_MCU_ESP32C6` |
| Espressif | ESP32（主机） |  | ✔（max3421e） | ✖ | max3421 | `OPT_MCU_ESP32` |
| Raspberry Pi | RP2040 | ✔ | ✔（PIO-USB） | ✖ | rp2040 | `OPT_MCU_RP2040` |
| ST | STM32F4/F7/H7 | ✔ | ✔ | ✔ | dwc2 | `OPT_MCU_STM32F4` 等 |
| ST | STM32G0/G4/L4（FSDEV） | ✔ |  | ✖ | fsdev | `OPT_MCU_STM32G4` 等 |
| Nordic | nRF52840/5340 | ✔ |  | ✖ | nrf | `OPT_MCU_NRF5X` |
| NXP | LPC55xx / iMX RT | ✔ | ✔ | ✔ | dwc2/ehci | `OPT_MCU_LPC55XX` / `OPT_MCU_IMXRT` |
| Microchip | SAMD21/51 | ✔ |  | ✖ | samd | `OPT_MCU_SAMD21` 等 |

> 注：SuperSpeed（USB 3.0，5Gbps）**不受支持**。

## 设备类使能速查

`tusb_config.h` 中以 `CFG_TUD_<CLASS>` 控制设备类实例数（0 = 关闭）。

| 类 | 使能宏 | 设备类描述 |
|---|---|---|
| CDC（虚拟串口） | `CFG_TUD_CDC` | 通信设备类，CDC-ACM |
| MSC（U盘） | `CFG_TUD_MSC` | 大容量存储 |
| HID（键鼠/通用） | `CFG_TUD_HID` | 人机接口设备 |
| MIDI | `CFG_TUD_MIDI` | 音乐设备 |
| Audio（UAC2） | `CFG_TUD_AUDIO` | USB 音频 2.0 |
| DFU | `CFG_TUD_DFU` / `CFG_TUD_DFU_RUNTIME` | 固件升级 |
| Vendor | `CFG_TUD_VENDOR` | 厂商自定义（含 WebUSB） |
| BTH HCI | `CFG_TUD_BTH` | 蓝牙 HCI |
| MTP | `CFG_TUD_MTP` | 媒体传输协议 |
| Video（UVC） | `CFG_TUD_VIDEO` | 视频类（WIP） |

主机侧对应 `CFG_TUH_CDC` / `CFG_TUH_MSC` / `CFG_TUH_HID` / `CFG_TUH_HUB` 等，主机栈总开关 `CFG_TUH_ENABLED`。

## USB 设备状态机（由 `src/device/usbd.c` 自动管理）

```
Attached → Powered → Default(addr 0) → Address → Configured ⇄ Suspended
```

应用通过回调感知状态：
- `tud_mount_cb()` / `tud_umount_cb()` — 配置被选中 / 取消（即“设备被识别/拔出配置”）
- `tud_suspend_cb(bool remote_wakeup_en)` / `tud_resume_cb()` — 总线挂起 / 恢复
- `tud_connected()` / `tud_mounted()` / `tud_suspended()` — 查询当前状态
- `tud_remote_wakeup()` — 请求主机远程唤醒（需 `CFG_TUD_USBD_ENABLE_REMOTE_WAKEUP`）

主机侧：`tuh_mount_cb(daddr)` / `tuh_umount_cb(daddr)` 在设备地址变化时触发。

---

## Critical Pitfalls (Must Read)

以下为最高频错误，任一违规都会导致设备不工作。

### 1. 主循环不调用 tud_task / tuh_task

```c
// ❌ WRONG — 从不调任务函数，USB 事件永远不处理，设备无法枚举
int main(void) {
  board_init();
  tusb_init(0, NULL);
  // 没有 tud_task()，事件队列堆积，主机识别不到设备
  while (1) { app_logic(); }
}

// ✅ CORRECT — 频繁喂任务（裸机主循环）
int main(void) {
  board_init();
  tusb_rhport_init_t dev_init = {.role = TUSB_ROLE_DEVICE, .speed = TUSB_SPEED_AUTO};
  tusb_init(BOARD_TUD_RHPORT, &dev_init);
  board_init_after_tusb();
  while (1) {
    tud_task();      // 必须在主循环或 <1ms RTOS 任务里
    app_logic();
  }
}
```

### 2. USB ISR 没有转发给 TinyUSB

```c
// ❌ WRONG — USB 中断里什么也不做，DCD 永远不处理硬件事件
void USB0_IRQHandler(void) { /* 空 */ }

// ✅ CORRECT — 转发到栈（tusb_init 已在该 rhport 调用过）
void USB0_IRQHandler(void) {
  tusb_int_handler(0, true);   // rhport 0
}
// STM32CubeIDE 集成：在 stm32xxx_it.c 的 OTG_FS_IRQHandler 里调用 tud_int_handler(0)
```

### 3. tusb_config.h 缺少 CFG_TUSB_MCU

```c
// ❌ WRONG — 报错 #error CFG_TUSB_MCU must be defined
// #include "tusb.h" 后 tusb_option.h 强制要求此宏

// ✅ CORRECT — 用编译器宏传入（由 board.mk / CMake 设置），或直接定义
#define CFG_TUSB_MCU      OPT_MCU_ESP32S3   // 来自 src/tusb_option.h
#define CFG_TUSB_OS       OPT_OS_NONE       // OPT_OS_FREERTOS / RTTHREAD / PICO / ZEPHYR ...
#define CFG_TUD_ENABLED   1
```

### 4. 描述符里端点号/接口号与栈不匹配

```c
// ❌ WRONG — ITF_NUM_TOTAL 少算接口；CDC 实际占 2 个接口（通知+数据）
enum { ITF_NUM_CDC = 0, ITF_NUM_CDC_DATA, ITF_NUM_MSC, /* 漏了 CDC 占用 */ };

// ✅ CORRECT — CDC 用 IAD，占 2 个接口；MSC 占 1 个
enum { ITF_NUM_CDC = 0, ITF_NUM_CDC_DATA, ITF_NUM_MSC, ITF_NUM_TOTAL };
// 配置描述符总长需含每个 TUD_*_DESC_LEN
uint8_t const desc_configuration[] = {
  TUD_CONFIG_DESCRIPTOR(1, ITF_NUM_TOTAL, 0, CONFIG_TOTAL_LEN, ...),
  TUD_CDC_DESCRIPTOR(ITF_NUM_CDC, 4, EPNUM_CDC_NOTIF, 8, EPNUM_CDC_OUT, EPNUM_CDC_IN, 64),
  TUD_MSC_DESCRIPTOR(ITF_NUM_MSC, 5, EPNUM_MSC_OUT, EPNUM_MSC_IN, 512),
};
```

### 5. CDC 写后忘 flush

```c
// ❌ WRONG — 数据只进了 TX FIFO，没真正发到总线
tud_cdc_write(buf, len);

// ✅ CORRECT — write 后调用 flush 触发实际传输
tud_cdc_write(buf, len);
tud_cdc_write_flush();
// 发送前可查可用空间：tud_cdc_write_available()
```

### 6. HID 上报前没查 ready，或一帧发多 report 不链式

```c
// ❌ WRONG — 端点忙时直接丢包，且一次循环连发多 report 会互相覆盖
tud_hid_keyboard_report(REPORT_ID_KEYBOARD, 0, keycode);
tud_hid_mouse_report(REPORT_ID_MOUSE, 0, dx, dy, 0, 0);

// ✅ CORRECT — 先查 ready，多 report 用 tud_hid_report_complete_cb() 链式续发
if (tud_hid_ready()) {
  tud_hid_keyboard_report(REPORT_ID_KEYBOARD, 0, keycode);
}
// 在回调里发下一个 report
void tud_hid_report_complete_cb(uint8_t instance, uint8_t const *report, uint16_t len) {
  uint8_t next_id = report[0] + 1u;
  if (next_id < REPORT_ID_COUNT) send_hid_report(next_id, btn);
}
```

### 7. MSC 后端回调返回值/越界错误

```c
// ❌ WRONG — 不做边界检查，返回字节数不规范
int32_t tud_msc_read10_cb(uint8_t lun, uint32_t lba, uint32_t off,
                          void *buf, uint32_t bufsize) {
  memcpy(buf, &disk[lba][off], bufsize);
  return bufsize;   // 无越界保护
}

// ✅ CORRECT — 校验边界，失败用 set_sense 并返回 -1
int32_t tud_msc_read10_cb(uint8_t lun, uint32_t lba, uint32_t off,
                          void *buf, uint32_t bufsize) {
  (void)lun;
  if (lba >= DISK_BLOCK_NUM) { tud_msc_set_sense(lun, SCSI_SENSE_ILLEGAL_REQUEST, 0x3a, 0x00); return -1; }
  memcpy(buf, &disk[lba][off], bufsize);
  return (int32_t)bufsize;
}
```

### 8. tusb_init 调用形式错误（宏重载）

```c
// ❌ WRONG — 误以为 tusb_init 无参版永远存在
tusb_init();   // 仅当定义了 CFG_TUSB_RHPORT0_MODE/1_MODE 时才有无参形式

// ✅ CORRECT — 显式传 rhport + 初始化结构（推荐，跨板通用）
tusb_rhport_init_t dev_init = {.role = TUSB_ROLE_DEVICE, .speed = TUSB_SPEED_AUTO};
tusb_init(BOARD_TUD_RHPORT, &dev_init);
```

### 9. Vendor 类关闭缓冲后仍用 read/write

```c
// ❌ WRONG — tusb_config.h 设 CFG_TUD_VENDOR_RX_BUFSIZE=0 后，这些函数不可用
#define CFG_TUD_VENDOR_RX_BUFSIZE 0
// 应用里却：tud_vendor_read(buf, len);   // 编译/运行异常

// ✅ CORRECT — 缓冲关闭时数据直达回调，须在回调里处理
void tud_vendor_rx_cb(uint8_t itf, uint8_t const* buffer, uint16_t bufsize) {
  // buffer 是端点原始数据，在此处理；不要调 tud_vendor_read/write
}
// 或：恢复缓冲（设非 0）后用 tud_vendor_read/write
```

### 10. 与厂商 USB 中间件冲突

```c
// ❌ WRONG — STM32CubeMX 同时启用 ST 的 USB Device Library 与 TinyUSB，符号/中断冲突

// ✅ CORRECT — 在 CubeMX 里禁用 USB 代码生成，让 TinyUSB 接管所有 USB 功能
// 文档原文：Disable USB code generation in STM32CubeMX and let TinyUSB handle all USB functionality.
```

### 11. 混淆设备/主机方向术语

```text
// ❌ WRONG — 把 IN/OUT 当主机视角
// 说 "OUT 端点是主机接收" → 误解

// ✅ CORRECT — IN/OUT 永远以设备视角（TinyUSB 代码惯例）：
//   IN  = 设备 → 主机（设备发，如 tud_cdc_tx_complete_cb）
//   OUT = 主机 → 设备（设备收）
```

### 12. 编译前没拉取 MCU 依赖（hw/mcu 缺失）

```bash
# ❌ WRONG — 直接 make/cmake，因 hw/mcu/<vendor> 为空而失败
cd examples/device/cdc_msc && cmake -DBOARD=stm32h743eval -B build

# ✅ CORRECT — 先按板/家族拉依赖，再构建
python tools/get_deps.py -b stm32h743eval   # 或 python tools/get_deps.py stm32h7
cd examples/device/cdc_msc && cmake -DBOARD=stm32h743eval -B build && cmake --build build
```

---

## Execution Workflow

| Step | Name | Description |
|------|------|-------------|
| 1 | 定位意图 | 确认是设备侧（`tud_`）还是主机侧（`tuh_`），以及目标类（CDC/HID/MSC/...） |
| 2 | 查 recipe | 在 `recipes/` 找匹配场景，读取其调用链与分步说明 |
| 3 | 查 API | recipe 未覆盖的 API，查 `resources/api_reference.md`（按 `tud_*`/`tuh_*` 分组） |
| 4 | 查配置 | 配置项/宏查 `resources/config_reference.md`，示例查 `resources/example_list.md` |
| 5 | 验证 | 核对：`tusb_config.h` 宏、描述符接口/端点编号、回调函数名、`tud_task` 在主循环 |
| 6 | 复制示例 | **新建项目**：复制最接近的真实示例（见 `resources/example_list.md`）再改；**已有项目**：就地编辑 |
| 7 | 核对陷阱 | 走一遍 `resources/pitfalls.md` 与上文 Critical Pitfalls |
| 8 | 构建 | CMake（首选）或 Make，`BOARD=<board>`；首次先 `get_deps.py` |
| 9 | 烧录/验证 | `*-jlink`/`-stlink`/`-openocd`/`-uf2` target；主机端用 `lsusb` / 设备管理器确认枚举 |

### Step 6 — 示例复制策略

**目标目录无项目（首次创建）时**，按需求选最接近示例整体复制再改：

- CDC 虚拟串口 → `examples/device/cdc_msc/src/`（含 CDC）或 `examples/device/webusb_serial/`
- HID 键盘/鼠标/游戏手柄 → `examples/device/hid_composite/`（多 report）或 `examples/device/hid_boot_interface/`
- MSC U 盘 → `examples/device/cdc_msc/src/msc_disk.c`（RAM 盘）或 `examples/device/msc_dual_lun/`
- Vendor / WebUSB → `examples/device/webusb_serial/` 或 `examples/device/hid_generic_inout/`
- 主机 CDC/MSC/HID → `examples/host/cdc_msc_hid/`
- 主机 U 盘文件浏览 → `examples/host/msc_file_explorer/`
- Audio/MIDI/DFU/Video → `examples/device/audio_*`、`examples/device/midi_test/`、`examples/device/dfu/`、`examples/device/video_capture/`
  - UAC2 音频 → `examples/device/uac2_headset/`（耳机）、`examples/device/uac2_speaker_fb/`（喇叭+反馈）、`examples/device/audio_4_channel_mic/`（麦克风）
  - DFU → `examples/device/dfu/`（DFU 模式）、`examples/device/dfu_runtime/`（DFU Runtime）
  - MIDI → `examples/device/midi_test/`（或 `midi_test_freertos/`）
  - USBTMC（仪器/SCPI） → `examples/device/usbtmc/`（含 `visaQuery.py` 主机测试）
  - Type-C / PD → `examples/typec/power_delivery/`（WIP，仅 STM32 G4）
- FreeRTOS 集成 → 选 `*_freertos` 后缀示例（如 `cdc_msc_freertos`、`hid_composite_freertos`）
- 双角色（OTG） → `examples/dual/host_hid_to_device_cdc/`

**目标目录已有项目时**，就地编辑，不要整体覆盖。

---

## Failure Strategies

| 情景 | 处理 |
|---|---|
| API 在 `resources/` 查不到 | 立即停止，告知用户该 API 可能不存在；去源码 `src/class/*/` 头文件二次确认 |
| 设备无法枚举 | 检查：`tud_task()` 是否在主循环；USB ISR 是否转发 `tusb_int_handler()`；描述符接口/端点编号是否一致；`CFG_TUD_ENABLED` 是否 1 |
| `#error CFG_TUSB_MCU must be defined` | 通过 board.mk 或 CMake 传入 `CFG_TUSB_MCU`；ESP-IDF 项目由 IDF 配置 |
| 数据收发异常 | 检查 CDC 是否 `write_flush()`；HID 是否查 `tud_hid_ready()`；缓冲区大小 `CFG_TUD_*_BUFSIZE` 是否够 |
| 链接缺 MCU 驱动 | 运行 `python tools/get_deps.py <family>` 拉取 `hw/mcu/<vendor>` |
| RAM 不够 | 调小 `CFG_TUD_*_EP_BUFSIZE` / `*_RX/TX_BUFSIZE`，关闭不用的类实例数（置 0） |
| 与厂商 USB 库冲突 | 禁用 ST/厂商 USB 中间件代码生成，由 TinyUSB 接管 |

## References

- 场景配方 → `recipes/` 目录
- API 速查（按模块分组） → `resources/api_reference.md`
- 配置宏/选项速查 → `resources/config_reference.md`
- 高频陷阱汇总 → `resources/pitfalls.md`
- 真实示例索引 → `resources/example_list.md`
- 仓库原始文档 → `D:/esp-skill/espressif-repos/tinyusb/docs/`（getting_started.rst、integration.rst、reference/）
