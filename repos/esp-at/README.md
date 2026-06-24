# esp-at-skill

面向 Espressif **ESP-AT** AT 指令固件的 AI 开发技能（Claude Code / Agent Skill 格式）。

ESP-AT 是基于 ESP-IDF 构建的官方 AT 固件平台，通过 UART/SPI/SDIO/socket 接口向外部 MCU 提供易于解析的 AT 指令集，快速集成 Wi-Fi、BLE、TCP-IP、HTTP、MQTT、WebSocket 等无线连接能力。本技能帮助 AI 代理在不臆造 API 的前提下，正确地构建、定制与调试 ESP-AT 固件——所有函数签名、结构体、Kconfig 选项、引脚、分区均来自仓库真实文档与头文件。

## 功能特性

- **场景化配方（recipes）**：覆盖本地编译烧录、AT 端口引脚修改、分区自定义、模块配置覆盖、自定义 AT 指令、BLE 服务定制、OTA 升级、SPI/SDIO 承载 AT 等核心场景，每个配方含真实代码与常见错误表。
- **真实 API 参考**：`resources/api_reference.md` 完整收录 `components/at/include/` 公开头文件的函数签名、结构体、枚举与宏。
- **配置选项速查**：`resources/config_reference.md` 汇总 `main/Kconfig` 的全部 AT 相关选项。
- **陷阱汇总**：`resources/pitfalls.md` 按类别归类实际开发中最常踩的 27 个坑。
- **示例索引**：`resources/example_list.md` 列出 `examples/` 下全部真实示例及关键源码位置。

## 支持范围

- **芯片**：ESP32、ESP32-C2、ESP32-C3、ESP32-C5、ESP32-C6、ESP32-C61、ESP32-S2
  - ESP32-S3 与 ESP32-H2 **不支持**
- **ESP-IDF 版本**：v5.4（ESP32/C2/C3/C6/S2）、v5.5（C5/C61）
- **通信接口**：UART / SPI / SDIO / socket
- **功能集**：Wi-Fi、BLE/BluFi/Classic BT(仅ESP32)、TCP-IP/SSL、HTTP、MQTT、WebSocket、文件系统(LittleFS/FatFS)、Ethernet(仅ESP32)、OTA、Web Server、RainMaker 等（按需通过 Kconfig 开启）

## 安装

将本技能目录克隆或复制到 Claude Code 的技能目录即可：

```bash
# 项目级技能（仅当前项目可用）
mkdir -p .claude/skills
cp -r esp-at-skill .claude/skills/

# 或用户级技能（所有项目可用）
mkdir -p ~/.claude/skills
cp -r esp-at-skill ~/.claude/skills/
```

目录结构：

```
esp-at-skill/
├── SKILL.md              # 主技能文件（原则、配方索引、陷阱、工作流）
├── AGENTS.md             # 补充约定（命名、include、构建、检查清单）
├── README.md             # 本文件
├── CHANGELOG.md          # 变更记录
├── recipes/              # 场景配方（8 个）
│   ├── build_and_flash.md
│   ├── set_port_pin.md
│   ├── customize_partitions.md
│   ├── override_module_config.md
│   ├── add_custom_command.md
│   ├── customize_ble_service.md
│   ├── ota_upgrade.md
│   └── at_over_spi_sdio.md
└── resources/            # 速查文档
    ├── api_reference.md
    ├── config_reference.md
    ├── pitfalls.md
    └── example_list.md
```

## 使用方式

1. AI 代理读取 `SKILL.md`，根据用户意图在 "Scenario Quick Reference" 中匹配配方。
2. 读取对应 `recipes/*.md`，按其分步说明与真实代码执行。
3. 配方未覆盖的 API/配置，查 `resources/api_reference.md` 与 `resources/config_reference.md`。
4. 遇到问题查 `resources/pitfalls.md`。

## 授权

ESP-AT 仓库本身采用 Apache-2.0 许可。本技能为社区编写的开发辅助资料。
