# esp-lowcode-matter-skill

面向 Espressif **[esp-lowcode-matter](https://github.com/espressif/esp-lowcode-matter)** 框架的 Claude Code / Agent AI Skill。该框架在 ESP32-C6 上以非对称双核（HP Core + LP Core）方式构建 Matter 连接产品：HP Core 跑预编译的 Wi-Fi/BLE/Matter 协议栈镜像，开发者只需在 LP Core 上用 Arduino 风格的 `setup()/loop()` + `low_code` 事件/特性模型编写产品固件。

本 Skill 让 AI 代理**正确**地进行 esp-lowcode-matter 固件开发——所有 API、结构体、配置项、引脚、代码片段均来自仓库源码与文档，绝不臆造。

## 功能特性

- **场景化 recipes（16 篇）**：环境搭建、创建产品、setup/loop 骨架、特性收发、事件处理、GPIO、按键、灯光（LED/WS2812）、继电器插座、system_timer、SHT30、LD2420、SSD1306、产品配置、调试。
- **真实 API 速查**：`low_code`、`system`、`button`、`relay`、`light`、`temperature_sensor_sht30`、`occupancy_sensor_ld2420`、`display_ssd1306` 等组件的函数签名与类型。
- **配置参考**：`product_info.json` 字段、Kconfig 选项、CMake REQUIRES、默认引脚、烧录地址。
- **陷阱清单**：30 条高频陷阱（生命周期、路由、内存、构建）。
- **示例索引**：仓库全部产品/组件/驱动/文档的真实路径。

## 适用范围

- 创建或定制 Matter 产品（灯、插座、传感器、温控、占用检测）
- 在 LP Core 编写 `app_main.cpp` / `app_driver.cpp`
- 使用官方组件与外设驱动
- 自定义 Matter 数据模型（`data_model.zap`）
- 本地终端/Codespaces/VS Code 的构建烧录与调试

**仅支持 ESP32-C6**。需要直接用 ESP Matter SDK 或 Connectedhomeip 的场景请用对应方案。

## 安装

将该 Skill 目录放入 Claude Code 的 skills 目录之一，并按需启用：

- 项目级：`<项目根>/.claude/skills/esp-lowcode-matter-skill/`
- 用户级：`~/.claude/skills/esp-lowcode-matter-skill/`

克隆：

```sh
git clone <this-skill-repo> ~/.claude/skills/esp-lowcode-matter-skill
# 或复制到项目：
cp -r esp-lowcode-matter-skill  <项目根>/.claude/skills/
```

安装后 Claude Code 会自动发现 `SKILL.md` 的 front matter，并在用户提到 esp-lowcode-matter / Matter / ESP32-C6 等触发词时加载本 Skill。

## 目录结构

```
esp-lowcode-matter-skill/
├── SKILL.md              # 入口：核心原则、场景速查、陷阱、执行流程
├── AGENTS.md             # 约定：命名/包含/骨架/构建/checklist
├── recipes/              # 16 篇场景化操作指南
├── resources/            # api_reference / config_reference / pitfalls / example_list
├── README.md             # 本文��
└── CHANGELOG.md          # 版本历史
```

## 许可

Apache-2.0（与 esp-lowcode-matter 仓库一致）。
