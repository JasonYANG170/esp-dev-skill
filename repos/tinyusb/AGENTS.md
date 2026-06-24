# AGENTS.md — TinyUSB 补充开发约定

> 核心规则、配方索引、陷阱、执行工作流均在 `SKILL.md`。本文件仅补充 `SKILL.md` 未涵盖的工程约定与工具指引，不重复内容。

## Project Context

- **���言**：C99（无动态内存分配）
- **目标**：跨平台 USB Host/Device 栈，50+ MCU 家族（Espressif ESP32-S2/S3/P4/C3/C6、STM32、RP2040、nRF、LPC、Kinetis、SAMD、RA 等）
- **工具链/构建**：CMake + Ninja（首选）或 Make；部分家族（espressif、rp2040）仅支持 CMake
- **RTOS**：可选，裸机 / FreeRTOS / RT-Thread / Mynewt / Pico SDK / Zephyr（`CFG_TUSB_OS`）

## 代码生成约定

### 文件命名与目录

- 示例源码统一放在 `src/` 子目录：`examples/<device|host|dual>/<name>/src/`
  - `main.c` — 入口与主循环
  - `tusb_config.h` — 栈配置（必须）
  - `usb_descriptors.c` — 设备侧描述符回调（设备栈必须）
  - 类后端文件，如 `msc_disk.c`、`tusb_config.h`
- BSP：`hw/bsp/<FAMILY>/boards/<BOARD>/`
- MCU 驱动：`hw/mcu/<vendor>/`（需 `get_deps.py` 拉取）
- 栈源码：`src/`（`tusb.h` 总入口，`src/class/<class>/*_device.h` / `*_host.h` 类 API）

### Include 模式

```c
// 应用统一只 include 一个总头（已包含配置与类驱动）
#include "tusb.h"

// 板级 API（board_init、board_led_write、board_millis、board_button_read）
#include "bsp/board_api.h"
```

注意：
- `tusb.h` 内部会按 `tusb_config.h` 使能的宏拉入对应类的 `*_device.h` / `*_host.h`
- 应用代码 **只调用 `tud_*` / `tuh_*` / `tusb_*`**，绝不直接调 `dcd_*` / `hcd_*` / `tu_*`

### 标准设备项目结构

```
my_usb_device/
├── src/
│   ├── main.c              # board_init → tusb_init → while(1){tud_task(); app();}
│   ├── tusb_config.h       # CFG_TUSB_MCU / CFG_TUSB_OS / CFG_TUD_*
│   └── usb_descriptors.c   # tud_descriptor_device_cb / _configuration_cb / _string_cb
├── CMakeLists.txt          # 或 Makefile（参考 examples/device/cdc_msc/）
└── CMakePresets.json        # 预设 BOARD
```

### 标准主循环模式（裸机）

```c
#include "bsp/board_api.h"
#include "tusb.h"

int main(void) {
  board_init();

  tusb_rhport_init_t dev_init = {.role = TUSB_ROLE_DEVICE, .speed = TUSB_SPEED_AUTO};
  tusb_init(BOARD_TUD_RHPORT, &dev_init);

  board_init_after_tusb();

  while (1) {
    tud_task();          // 处理设备事件（必须频繁调用）
    // app_task();
  }
}

// USB 中断转发（命名与芯片向量表一致）
void USB0_IRQHandler(void) { tusb_int_handler(0, true); }
```

### 回调实现约定

TinyUSB 回调无需注册，按“规定函数名”实现即可被栈调用：

```c
// 设备核心回调
void tud_mount_cb(void);                  // 配置被主机选中
void tud_umount_cb(void);
void tud_suspend_cb(bool remote_wakeup_en);
void tud_resume_cb(void);

// 描述符回调（设备必须实现前三个）
uint8_t const* tud_descriptor_device_cb(void);
uint8_t const* tud_descriptor_configuration_cb(uint8_t index);
uint16_t const* tud_descriptor_string_cb(uint8_t index, uint16_t langid);
```

### 中断处理模板

```c
// 在芯片 USB 中断向量对应函数里转发（in_isr 参数传 true）
void USB0_IRQHandler(void) { tusb_int_handler(0, true); }

// STM32 OTG：在 stm32xxx_it.c 里
void OTG_FS_IRQHandler(void) { tusb_int_handler(0, true); }
// 旧文档写法 tud_int_handler(0) 仍兼容，推荐 tusb_int_handler(rhport, in_isr)
```

### 内存对齐约定

部分 MCU 的 USB DMA 只能访问特定 SRAM 区/对齐，TinyUSB 提供：

```c
// tusb_config.h 可重定义
#ifndef CFG_TUSB_MEM_SECTION
#define CFG_TUSB_MEM_SECTION          // 如 __attribute__((section(".usb_ram")))
#endif
#ifndef CFG_TUSB_MEM_ALIGN
#define CFG_TUSB_MEM_ALIGN            __attribute__((aligned(4)))
#endif
```

## 构建工作流

### CMake（首选）

```bash
# 1. 首次：按 board 或 family 拉 MCU 依赖
python tools/get_deps.py -b stm32h743eval      # 按板
python tools/get_deps.py rp2040                 # 按家族

# 2. 配置 + 构建
cd examples/device/cdc_msc
cmake -DBOARD=stm32h743eval -B build            # 加 -G Ninja 用 Ninja
cmake --build build

# 3. 烧录/调试（target 因板而异，用 --target help 列全部）
cmake --build build --target cdc_msc-jlink      # 或 -stlink / -openocd
cmake --build build --target cdc_msc-uf2        # 生成 UF2（RP2040 等）
```

### Make（备选，部分家族不支持）

```bash
cd examples/device/cdc_msc
make BOARD=stm32h743eval all
make BOARD=stm32h743eval flash-jlink            # 或 flash-stlink / flash-openocd
make BOARD=stm32h743eval all uf2                # 生成 UF2
# 调试/日志：DEBUG=1 all；LOG=2 all；LOG=2 LOGGER=rtt all
```

### ESP-IDF 集成（Espressif 目标）

ESP-IDF 项目通过 IDF 自带 `tinyusb` 组件管理（非本仓库直接 make）；`CFG_TUSB_MCU` 等由 IDF Kconfig 注入。本 Skill 主要面向 TinyUSB 栈本身 API 与示例。

## 代码生成 Checklist

- [ ] `tusb_config.h` 定义了 `CFG_TUSB_MCU`、`CFG_TUSB_OS`、`CFG_TUD_ENABLED`（或 `CFG_TUH_ENABLED`）
- [ ] 设备类实例数 `CFG_TUD_<CLASS>` 与描述符里的接口数一一对应
- [ ] 端点 0 大小 `CFG_TUD_ENDPOINT0_SIZE`（默认 64）已设
- [ ] `tusb_init(BOARD_TUD_RHPORT, &dev_init)` 在 `board_init()` 之后调用
- [ ] 主循环 / 高优先级 RTOS 任务里持续调用 `tud_task()`（或 `tuh_task()`）
- [ ] USB ISR 转发 `tusb_int_handler(rhport, true)`
- [ ] 设备侧实现 `tud_descriptor_device_cb` / `_configuration_cb` / `_string_cb`
- [ ] 接口号 `ITF_NUM_TOTAL`、端点号 `EPNUM_*` 与描述符宏长度 `TUD_*_DESC_LEN` 一致
- [ ] CDC 写后 `tud_cdc_write_flush()`；HID 上报前查 `tud_hid_ready()`
- [ ] VID/PID 唯一（多类组合用 `PID_MAP()` 位图）
- [ ] 未直接调用 `dcd_*` / `hcd_*` / `tu_*` 内部函数

## Do Not Modify

- `src/` — TinyUSB 栈核心源码（如需自定义类驱动，用 `usbd_app_driver_get_cb()` / `usbh_app_driver_get_cb()` 注入，不改栈源码）
- `hw/mcu/` — 第三方 MCU 驱动（由 `get_deps.py` 管理）
- `SKILL.md` frontmatter — Skill 元数据
- 真实仓库源码路径 `D:/esp-skill/espressif-repos/tinyusb/` — 仅作只读引用来源
