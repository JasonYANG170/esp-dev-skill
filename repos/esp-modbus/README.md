# esp-modbus-skill

面向 [Espressif esp-modbus](https://github.com/espressif/esp-modbus) 库（Modbus RTU / ASCII / TCP，组件版本 v2.x.x）的 AI 编程助手技能（Claude Code / Agent skill 格式）。本技能为 AI 代理提供**完全基于仓库真实文档与源码**的场景化指南、API 速查、配置参考与排错清单，使其能够正确地创建、修改、调试运行在 ESP-IDF 上的 Modbus 主站 / 从站固件，**绝不杜撰 API**。

## 功能特性

- **场景化配方（recipes）**：覆盖串行主/从站、TCP 主/从站、数据字典编写、从站寄存器区域映射、自定义功能码、扩展数据类型与字节序转换、从站事件循环，共 8 个配��。
- **API 速查（resources/api_reference.md）**：从 `modbus/mb_controller/common/include/` 头文件逐字提取的函数签名、结构体、枚举。
- **配置参考（resources/config_reference.md）**：`Kconfig` 全部 `CONFIG_FMB_*` 选项 + 示例级 `CONFIG_MB_*` 选项。
- **陷阱清单（resources/pitfalls.md）**：15 条最常见的错误及修复方法（含 `WRONG` / `CORRECT` 代码对照）。
- **示例索引（resources/example_list.md）**：`examples/` 下真实存在的工程路径及一句话说明。

## 支持范围

- 目标芯片：ESP32 / ESP32-S2 / ESP32-S3 / ESP32-C2 / C3 / C5 / C6 / C61 / H2 / P4
- 框架：ESP-IDF v5.0 及以上（CMake 构建）
- 协议：Modbus RTU、ASCII（串行 / RS485）、Modbus TCP（Wi-Fi / 以太网 / IPv4 / IPv6）
- API 风格：v2 实例化（`mbc_*_create_*` 返回句柄，句柄作为后续每个调用的首参）

## 安装方式

将本技能目录放入 Claude Code 的技能目录即可（项目级或用户级二选一）：

```bash
# 项目级（随仓库走）
mkdir -p .claude/skills
cp -r esp-modbus-skill .claude/skills/

# 用户级（对所有项目生效）
mkdir -p ~/.claude/skills
cp -r esp-modbus-skill ~/.claude/skills/
```

Claude Code 会在用户提及 Modbus / RTU / ASCII / Modbus TCP / RS485 / esp-modbus / Modbus 主站 / 从站 等触发词时自动加载本技能。

## 目录结构

```
esp-modbus-skill/
├── SKILL.md                       # 主入口：原理、配方索引、陷阱、执行流程
├── AGENTS.md                      # 补充约定：项目结构、include 模式、初始化模板、构建流程
├── recipes/                       # 场景化配方（中文说明 + 真实代码 + 常见错误表）
│   ├── serial_slave.md
│   ├── serial_master.md
│   ├── tcp_slave.md
│   ├── tcp_master.md
│   ├── data_dictionary.md
│   ├── slave_register_areas.md
│   ├── custom_handlers.md
│   ├── extended_types.md
│   └── slave_events.md
├── resources/                     # 速查文档（全部来自仓库真实头文件/文档/示例）
│   ├── api_reference.md
│   ├── config_reference.md
│   ├── pitfalls.md
│   └── example_list.md
├── README.md                      # 本文件
└── CHANGELOG.md
```

## 数据来源声明

本技能所有函数名、结构体、宏、Kconfig 选项、文件路径与代码片段均直接取自 `espressif-repos/esp-modbus` 仓库的 `docs/en/`、`modbus/mb_controller/common/include/`、`Kconfig`、`examples/`。仓库文档较薄处（如部分内部端口层）未做臆测补充。

## 许可证

esp-modbus 库本身为 Apache-2.0。本技能文档按同样口径提供。
