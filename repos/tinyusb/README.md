# tinyusb-skill

面向 [TinyUSB](https://github.com/hathach/tinyusb)（开源跨平台 USB Host/Device 协议栈）的 AI Skill（Claude Code / Agent skill 格式）。TinyUSB 以“无动态内存分配 + 中断延迟到任务上下文处理”为核心，广泛用于嵌入式 USB 设备与主机开发，支持 50+ MCU 家族（Espressif ESP32-S2/S3/C3/C6/P4、STM32、RP2040、nRF、LPC、Kinetis、SAMD 等）。

本 Skill 让 AI Agent 能够**正确**地基于 TinyUSB 开发固件：所有 API、回调、宏、配置项、示例路径均来自 TinyUSB 真实源码（`src/`、`docs/`、`examples/`），绝不臆造。

## 功能特性

- **场景化配方（recipes）**：覆盖设备栈集成、CDC/HID/MSC/Vendor/WebUSB、Audio(UAC2)/MIDI/DFU/USBTMC、Type-C/PD、描述符编写、构建烧录、主机栈、`tusb_config.h` 配置等 14 个真实场景
- **API 速查**：按 `tud_*`（设备）/ `tuh_*`（主机）/ `tusb_*`（核心）分组的真实函数签名与回调声明
- **配置参考**：`CFG_TUSB_*` / `CFG_TUD_*` / `CFG_TUH_*` 宏与 `OPT_OS_*` / `OPT_MODE_*` / `OPT_MCU_*` 枚举
- **陷阱汇总**：枚举失败、CDC/HID/MSC 常见错误、描述符坑、并发模型、术语陷阱
- **示例索引**：设备/主机/双角色/Type-C 全部真实示例路径与说明

## 安装

将本目录放入 Claude Code 的 skills 目录之一即可被自动发现：

- 项目级：`.claude/skills/tinyusb-skill/`
- 用户级：`~/.claude/skills/tinyusb-skill/`

或直接 git clone：

```bash
# 项目级
mkdir -p .claude/skills
cp -r tinyusb-skill .claude/skills/

# 用户级
cp -r tinyusb-skill ~/.claude/skills/
```

随后在对话中提及 USB / TinyUSB / CDC / HID / MSC 等触发词即可触发本 Skill。

## 目录结构

```
tinyusb-skill/
├── SKILL.md                # 核心规则、配方索引、陷阱、执行工作流
├── AGENTS.md               # 补充工程约定（文件命名、include、构建、checklist）
├── recipes/                # 场景配方（含真实代码与常见错误表）
│   ├── device_getting_started.md
│   ├── cdc_virtual_serial.md
│   ├── hid_device.md
│   ├── msc_device.md
│   ├── descriptors_config.md
│   ├── vendor_and_webusb.md
│   ├── host_cdc_msc_hid.md
│   ├── build_and_flash.md
│   └── tusb_config_guide.md
├── resources/              # 速查文档
│   ├── api_reference.md
│   ├── config_reference.md
│   ├── pitfalls.md
│   └── example_list.md
├── README.md
└── CHANGELOG.md
```

## 支持范围

- **设备类**：CDC、HID（键盘/鼠标/手柄/触控）、MSC、MIDI、Audio（UAC2）、DFU、Vendor（含 WebUSB/WinUSB）、MTP、BTH HCI、Video（UVC，WIP）
- **主机类**：CDC-ACM、MSC、HID、Hub、Vendor 串口转换（FTDI/CP210x/CH34x/PL2303）
- **目标 MCU**：Espressif ESP32-S2/S3/P4/C3/C6（dwc2）、STM32（dwc2/fsdev）、RP2040、nRF、LPC、Kinetis、SAMD、RA 等 50+ 家族
- **RTOS**：裸机、FreeRTOS、RT-Thread、Mynewt、Pico SDK、Zephyr
- **构建**：CMake（首选）/ Make
- **不支持**：USB 3.0 SuperSpeed（5Gbps）

## 许可

本 Skill 文档遵循 MIT 许可（与 TinyUSB 一致）。代码片段均改编自 TinyUSB 官方示例，版权归原作者所有。

## 数据来源

所有内容基于 `D:/esp-skill/espressif-repos/tinyusb/` 真实仓库，主要来源：
- `docs/`（getting_started.rst、integration.rst、reference/architecture.rst、usb_concepts.rst、concurrency.rst、glossary.rst、dependencies.rst、boards.rst）
- `src/tusb.h`、`src/tusb_option.h`、`src/device/usbd.h`
- `src/class/{cdc,hid,msc,vendor}/{_device,_host}.h`
- `examples/{device,host,dual,typec}/` 真实示例源码
