# connectedhomeip-skill

面向 Matter / CHIP（Connected Home over IP）固件开发的 AI Skill，基于 Espressif 对上游 `project-chip/connectedhomeip`（CSA Matter 参考实现）的分叉。本 Skill 让 AI Agent 能够基于仓库真实文档与代码，正确地在 ESP32 / ESP32-C3 上构建、配网、定制 Matter 设备示例，而不会臆造 API。

## 功能特性

- **场景驱动 recipes**：构建烧录、BLE/Wi-Fi/Bypass 配网、python controller、chip-tool CLI、OnOff 集群绑 GPIO、自定义属性回调、设备事件处理等真实流程。
- **真实 API 参考**：`CHIPDeviceManager`、`InitServer`、`PlatformMgr/ConnectivityMgr`、`emberAfWriteAttribute`、ZCL 集群宏等签名均来自仓库 `examples/` 与 `src/`。
- **配置速查**：Rendezvous 模式（Bypass=0/Wi-Fi=1/BLE=2/Thread=4/Ethernet=8）、设备类型、sdkconfig.defaults、分区表。
- **关键陷阱清单**：初始化顺序、`idf.py` 构建、NimBLE、回调校验、mDNS 重启、加锁查询、子模块初始化等。
- **示例索引**：标注哪些示例含 `esp32/` 子目录可真正烧录。

## 支持范围

- 芯片：ESP32（Xtensa）、ESP32-C3（RISC-V）
- 设备类型：ESP32-DevKitC、ESP32-WROVER-KIT_V4.1、M5Stack、ESP32C3-DevKitM
- 工具链：ESP-IDF v4.3（设备固件）+ GN/ninja（chip-tool 主机侧）
- 集群：以 OnOff 为核心模板，可扩展到 DoorLock / Temperature 等

> 本仓库是较早的 CHIP 时代分叉，部分示例（如 lighting-app）在本版本中不含 `esp32/` 子目录；请以 `resources/example_list.md` 为准。

## 安装方法

将本 Skill 目录克隆/复制到 Claude Code 的 skills 目录：

- 项目级：`<project>/.claude/skills/connectedhomeip-skill/`
- 用户级：`~/.claude/skills/connectedhomeip-skill/`

或直接指向本目录：

```
D:/esp-skill/skills/connectedhomeip-skill/
```

安装后，Agent 在遇到 "Matter"、"CHIP"、"connectedhomeip"、"chip-tool"、"配网"、"cluster" 等触发词时会自动加载本 Skill。

## 目录结构

```
connectedhomeip-skill/
├── SKILL.md                 # 核心规则、recipes 索引、陷阱、执行流程
├── AGENTS.md                # 项目约定、命名、include、构建工作流
├── recipes/                 # 场景驱动步骤（中文）
│   ├── build_esp32_example.md
│   ├── new_esp32_example.md
│   ├── commissioning_ble.md
│   ├── commissioning_wifi_bypass.md
│   ├── python_controller.md
│   ├── chip_tool_cli.md
│   ├── onoff_cluster_hardware.md
│   ├── custom_attribute_callback.md
│   └── device_event_handling.md
├── resources/               # 速查文档
│   ├── api_reference.md
│   ├── config_reference.md
│   ├── pitfalls.md
│   └── example_list.md
├── README.md                # 本文件
└── CHANGELOG.md
```

## 许可证

跟随上游仓库：Apache-2.0。
