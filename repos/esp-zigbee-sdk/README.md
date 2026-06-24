# esp-zigbee-sdk-skill

面向 [Espressif ESP Zigbee SDK](https://github.com/espressif/esp-zigbee-sdk) **v2.x**（基于 ESP-IDF 与乐鑫自研 Zigbee 协议栈，预编译库 `espressif/esp-zigbee-lib`）的 Claude Code / Agent 技能。帮助 AI 在 ESP32-H2 / ESP32-C6 等芯片上正确、可靠地开发 Zigbee 3.0 固件——从协调器/路由器/终端设备、ZHA 设备模型、ZCL cluster 与命令、BDB commissioning，到网关/RCP、休眠终端、OTA 与自定义 cluster。

本技能的全部内容（API 签名、结构体、宏、Kconfig、文件路径、代码片段）均**严格来自仓库真实文档与源码**（`docs/en/`、`components/esp-zigbee-lib/include/ezbee/`、`examples/`），不存在则不写入。

> 本技能面向 v2.x 主线（`ezb_` API）。v1.x（ZBOSS、`esp_zb_` API）为 LTS 维护分支，请参考仓库 `release/v1.0` 与 `docs/en/migration-guide/v2.x/`。

## 功能特性

- **场景 Recipes（10 篇）**：ZC 灯、ZR/ZED 开关、休眠终端、ZHA 数据模型、属性上报、ZCL 命令、ZCL Core Action、网关/RCP、Touchlink、OTA、自定义 cluster
- **API 速查**：`resources/api_reference.md`，按平台层 / Core / BDB / AF / ZHA / NWK / APS / Security / ZDO / ZCL / Touchlink 分组，真实签名
- **配置参考**：`resources/config_reference.md`，Kconfig、sdkconfig、`partitions.csv`、`idf_component.yml`、示例私有宏
- **陷阱清单**：SKILL.md 中 14 条带"错误/正确"代码对照 + `resources/pitfalls.md` 汇总版
- **示例索引**：`resources/example_list.md`，覆盖仓库 `examples/` 全部子目录
- **执行工作流**：从规划 → 选 recipe → 校验 → 复制示例 → 编译烧录 → 调试

## 适用范围

- 芯片：ESP32-H2、ESP32-C6（native 802.15.4）；ESP32-C3/S3/P4 + H2/C6 RCP 网关
- 角色：Coordinator / Router / End Device / Sleepy End Device
- 协议：Zigbee 3.0 / Zigbee Pro R23 / ZCL v8
- 构建链：ESP-IDF v5.2+（示例对齐 v5.5.4）、`idf.py`、`espressif/esp-zigbee-lib >=2.0.0`

## 安装

将本技能目录放入 Claude Code 的 skills 目录之一：

```bash
# 项目级（仅当前项目可用）
mkdir -p .claude/skills
cp -r esp-zigbee-sdk-skill .claude/skills/

# 用户级（所有项目可用）
mkdir -p ~/.claude/skills
cp -r esp-zigbee-sdk-skill ~/.claude/skills/
```

或直接 clone 仓库后软链 `skills/esp-zigbee-sdk-skill` 到上述目录。Claude Code 会自动发现并在触发词命中时加载。

## 目录结构

```
esp-zigbee-sdk-skill/
├── SKILL.md                  # 主入口：原则、When to Use、Recipes 索引、陷阱、工作流
├── AGENTS.md                 # 补充约定：API 分层、项目结构、main 模式、checklist
├── recipes/                  # 10 个场景 recipe
│   ├── coordinator_light.md
│   ├── router_switch.md
│   ├── sleepy_end_device.md
│   ├── zha_device_model.md
│   ├── zcl_attribute_report.md
│   ├── zcl_command_send.md
│   ├── zcl_core_action.md
│   ├── zigbee_gateway_rcp.md
│   ├── touchlink.md
│   ├── ota_upgrade.md
│   └── custom_cluster.md
├── resources/                # 速查文档
│   ├── api_reference.md
│   ├── config_reference.md
│   ├── pitfalls.md
│   └── example_list.md
├── README.md
└── CHANGELOG.md
```

## 许可

本技能内容基于 ESP Zigbee SDK（Apache-2.0）整理，引用代码片段遵循各源文件头部声明（多为 CC0-1.0 示例 / Apache-2.0 头文件）。技能本身按 Apache-2.0 提供。
