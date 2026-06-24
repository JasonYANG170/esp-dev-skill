# esp-thread-br-skill

针对乐鑫 (Espressif) 官方 **ESP Thread Border Router SDK** ([esp-thread-br](https://github.com/espressif/esp-thread-br)) 的 Claude Code / Agent Skill。

该 SDK 基于 ESP-IDF 与 OpenThread，用于在 ESP32 系列主控 SoC（ESP32-S3/P4/C5/C61）+ ESP32-H2/C6 RCP 双芯片上构建通过 Thread 1.4 认证的 Thread Border Router，支持双向 IPv6、服务发现 (SRP/mDNS)、组播转发、NAT64、TREL、RCP 更新、HTTPS OTA、Web GUI 与 Home Assistant 集成。

本 Skill 让 AI agent 完全基于仓库真实文档与代码（`docs/`、`examples/`、`components/`）进行固件开发：构建、配置、起网、网络特性验证、维护升级，不臆造任何 API。

## 主要特性

- **场景化 Recipes**：构建烧录、板型/接口、自动起网、双向 IPv6、组播转发、服务发现、NAT64、TREL、Web GUI、RCP 更新、HTTPS OTA 等 12 个真实场景。
- **扩展 API 速查**：覆盖 `esp_rcp_update` / `esp_br_http_ota` / `esp_ot_br_server` / `esp_ot_cli_extension` 等组件的真实函数签名。
- **配置参考**：从 `sdkconfig.defaults` 与 `Kconfig.projbuild` 提取的真实 Kconfig、引脚、分区表。
- **陷阱合集**：基于 `docs/en/qa.rst`、codelab 与源码注释的常见错误与正确写法对照。
- **示例与组件清单**：仓库内每个 example/component 的真实路径与一句话说明。

## 安装

将本目录放入 Claude Code 的 skills 目录之一即可被自动发现：

- **项目级**：`<repo>/.claude/skills/esp-thread-br-skill`
- **用户级**：`~/.claude/skills/esp-thread-br-skill`

```bash
git clone <this-skill-repo> ~/.claude/skills/esp-thread-br-skill
# 或拷贝本目录到 .claude/skills/
```

随后在 Claude Code 中触发（命中任一关键词即可）：`esp-thread-br`、`Thread Border Router`、`OpenThread`、`border router`、`线程边界路由器`、`Thread 边界路由器` 等。

## 目录结构

```
esp-thread-br-skill/
├── SKILL.md              # 主入口：原则、场景索引、陷阱、执行工作流
├── AGENTS.md             # 工程约定（命名、include、结构、构建、checklist）
├── recipes/              # 12 个场景化分步指南
│   ├── build_and_run.md
│   ├── board_and_interface.md
│   ├── auto_start_mode.md
│   ├── bidirectional_ipv6.md
│   ├── multicast_forwarding.md
│   ├── service_discovery.md
│   ├── nat64.md
│   ├── trel.md
│   ├── web_gui.md
│   ├── rcp_update.md
│   └── http_ota.md
├── resources/            # 速查文档
│   ├── api_reference.md
│   ├── config_reference.md
│   ├── pitfalls.md
│   └── example_list.md
├── README.md             # 本文件
└── CHANGELOG.md
```

## 支持范围

- **主控 SoC**：ESP32-S3（官方 BR 板默认）、ESP32-P4、ESP32-C5、ESP32-C61
- **RCP**：ESP32-H2（默认）、ESP32-C6
- **通信接口**：UART（默认 460800）、SPI
- **Backbone**：Wi-Fi（含 SoftAP 配网）、Ethernet（W5500 Sub-Ethernet）
- **ESP-IDF**：>= 5.1.0，推荐 v5.5.4
- **特性**：双向 IPv6、mDNS/SRP、组播转发、NAT64/DNS64、TREL、DHCPv6 PD、RCP 自动/手动更新、HTTPS OTA、Web GUI、Home Assistant、外部 RF 共存

## 许可

本 Skill 内容基于 Apache-2.0 许可的 esp-thread-br 仓库整理；Skill 本身按相同方式提供。
