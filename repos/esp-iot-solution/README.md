# esp-iot-solution-skill

面向 **Espressif ESP-IoT-Solution** 的 AI 技能（Claude Code / Agent Skill 格式）。

ESP-IoT-Solution 是构建在 ESP-IDF 之上的 IoT 组件库与方案集合，涵盖传感器、显示、USB、音频、输入设备、电机、低功耗等大量设备驱动与框架组件。本技能让 AI 代理能够**正确地**基于 ESP-IoT-Solution 开发固件——所有 API、结构体、配置项均来自仓库真实文档（`docs/`）与组件源码（`components/`、`examples/`），绝不臆造。

## 功能特性

- **场景化 recipe**：按键、旋钮、LED 指示灯、I2C/SPI 总线、sensor_hub、功率计量、USB 主机（UVC/UAC/CDC）、舵机、触摸按键等 13 个常见场景，每个 recipe 含适用摘要、触发意图、前置条件、分步说明（可复制代码）、常见错误表、参考示例路径。
- **API 速查**：`resources/api_reference.md` 按模块分组列出真实函数签名与结构体。
- **配置参考**：`resources/config_reference.md` 列出各组件真实 Kconfig 符号与默认值。
- **坑点合集**：`resources/pitfalls.md` 汇总最易踩的错（工厂函数签名、回调禁止阻塞、IDF 版本 backend 等）。
- **示例索引**：`resources/example_list.md` 列出仓库 `examples/` 下真实示例路径与一句话描述，便于复制起步。
- **执行工作流**：从需求 → 选 recipe → 查 API → 加依赖 → menuconfig → 编译烧录 → 验证，全流程指引。

## 适用范围

- ESP32 / ESP32-S2 / ESP32-S3 / ESP32-C2 / ESP32-C3 / ESP32-C6 / ESP32-H2 / ESP32-P4
- ESP-IDF v5.3+（master 分支）或 v4.4–v5.3（release/v2.0 分支）
- 组件独立迭代，具体依赖以各组件 `idf_component.yml` 为准

## 安装

将本技能目录克隆/复制到 Claude Code 的技能目录之一：

- 项目级：`<project>/.claude/skills/esp-iot-solution-skill/`
- 用户级：`~/.claude/skills/esp-iot-solution-skill/`

目录结构需保持：

```
esp-iot-solution-skill/
├── SKILL.md
├── AGENTS.md
├── README.md
├── CHANGELOG.md
├── recipes/
│   └── *.md
└── resources/
    └── *.md
```

安装后，当用户提及 ESP-IoT-Solution 组件（iot_button、led_indicator、i2c_bus、spi_bus、iot_knob、sensor_hub、usb_stream、iot_usbh_cdc、iot_servo 等）或相关场景时，代理会自动加载本技能。

## 目录索引

- 技能主体与原则 → `SKILL.md`
- 工程约定与工具链 → `AGENTS.md`
- 场景 recipe → `recipes/`
- 速查参考 → `resources/`

## 参考来源

- 仓库：https://github.com/espressif/esp-iot-solution
- 中文文档：https://docs.espressif.com/projects/esp-iot-solution/zh_CN
- 英文文档：https://docs.espressif.com/projects/esp-iot-solution/en
- ESP Component Registry：https://components.espressif.com/
- ESP-IDF 编程指南：https://docs.espressif.com/projects/esp-idf/

## 许可证

Apache-2.0（与上游 ESP-IoT-Solution 仓库一致）。
