# esp-usb-skill

面向 **ESP-USB**（Espressif USB 设备栈 `esp_tinyusb` + USB 主机库 `usb` 及主机 class 驱动 CDC/MSC/HID/UAC/UVC）的 AI 开发技能。所有 API、结构体、宏、Kconfig 选项、文件路径与代码片段均取自 `esp-usb` 仓库的真实头文件、Kconfig 与测试应用，杜绝臆造。

本技能让 AI 代理能够：

- 正确选择设备栈（`esp_tinyusb`）或主机栈（`usb` + class 驱动）并完成初始化
- 编写 CDC-ACM 串口、MSC 存储、HID、复合设备、console/VFS 重定向等设备固件
- 编写主机端 CDC-ACM（含 CP210x/FTDI/CH34x）、MSC、HID 应用
- 正确处理外部 PHY、VBUS 监测、挂起/恢复、远程唤醒、安装/卸载生命周期
- 避开 12+ 条最常见陷阱（含 WRONG/CORRECT 代码对照）

## 包含内容

```
esp-usb-skill/
├── SKILL.md                 # 核心原则、场景速查、参考表、关键陷阱、执行流程
├── AGENTS.md                # 项目约定、依赖添加、include 模式、构建流程、检查清单
├── recipes/                 # 10 个场景化方案（中文）
│   ├── device_cdc_serial.md        # USB CDC-ACM 串口设备
│   ├── device_msc_storage.md       # USB MSC 存储（SPI-Flash / SD 卡）
│   ├── device_console_vfs.md       # 控制台重定向与 VFS
│   ├── device_composite.md         # 复合设备（CDC + MSC）
│   ├── device_external_phy.md      # ESP32-S3 外部 PHY
│   ├── device_install_uninstall.md # 安装/卸载与事件、VBUS 监测
│   ├── host_library_basic.md       # USB Host Library 基本用法
│   ├── host_cdc_acm.md             # CDC-ACM 主机驱动
│   ├── host_msc.md                 # MSC 主机驱动（U 盘）
│   └── host_hid.md                 # HID 主机驱动（键盘/鼠标）
├── resources/               # 速查文档（英文 API 名）
│   ├── api_reference.md            # 真实函数签名（按模块分组）
│   ├── config_reference.md         # 真实 Kconfig / menuconfig 选项
│   ├── pitfalls.md                 # 汇总陷阱
│   └── example_list.md            # 真实示例/测试应用路径索引
├── README.md                # 本文��
└── CHANGELOG.md             # 变更记录
```

## 安装

将本技能目录放进 Claude Code / Agent 的 skills 目录之一（项目级或用户级），重启会话即可被自动发现。

- **项目级**：复制到 `<project>/.claude/skills/esp-usb-skill/`
- **用户级**：复制到 `~/.claude/skills/esp-usb-skill/`

```bash
# 示例：克隆/复制到用户级 skills 目录
mkdir -p ~/.claude/skills
cp -r ./esp-usb-skill ~/.claude/skills/
```

无需额外依赖；技能本身只是 Markdown 文档，配合你的 ESP-IDF 工程使用。

## 支持范围

- **目标芯片**：ESP32-S2、ESP32-S3、ESP32-S31、ESP32-P4、ESP32-H4（须带 USB-OTG 外设）
- **ESP-IDF**：主机组件 `>= 5.5.3`；设备组件 `>= 5.0`
- **设备类**：CDC-ACM、MSC、HID、MIDI、复合、Vendor、DFU、BTH、NCM/ECM-RNDIS
- **主机类**：CDC-ACM（含 vendor VCP）、MSC、HID、UVC、UAC，及 USB Host Library 低层 API
- **不在范围**：USB Type-C / USB-PD / TCPM（`usb_tcpm` 组件），PCB 设计，其他厂商 USB 栈

## 许可证

Apache-2.0（与 esp-usb 仓库头文件 SPDX 一致）。
