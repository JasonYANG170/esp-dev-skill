# esp-nimble-skill

面向 Apache NimBLE BLE 协议栈（Espressif 的 esp-nimble 分支）的 AI Skill。本 Skill 让 AI 代理（如 Claude Code）能够基于仓库真实文档与源码，正确开发、修改、调试 NimBLE Host 固件——涵盖 GAP 广播/扫描/连接、GATT 服务端/客户端、SMP 配对绑定、扩展/周期广播、Bluetooth Mesh 等场景。

所有 API、结构体、宏、配置项、代码片段均取自仓库 `D:/esp-skill/espressif-repos/esp-nimble` 的 `docs/`、`nimble/host/include/host/` 头文件与 `apps/` 示例，不臆造任何不存在的接口。

## 功能特性

- **场景化配方（recipes）**：11 个配方覆盖 Host 初始化、地址配置、外设广播、Beacon、通知、扫描、中心连接、自定义 GATT 服务、SMP 配对、扩展广播、Mesh 节点。
- **真实 API 参考**：`resources/api_reference.md` 按 GAP / GATT / SMP / 地址 / mbuf / FreeRTOS 端口分组，列出来自头文件的真实签名与常量。
- **配置项参考**：区分 Mynewt syscfg（`BLE_*`）与 ESP-IDF Kconfig（`CONFIG_BT_NIMBLE_*`）。
- **陷阱汇总**：15 条高频陷阱，每条给出错误与正确写法。
- **示例索引**：列出仓库 `apps/` 下全部真实示例及其用途。

## 支持范围

- **芯片**：ESP32、ESP32-C3、ESP32-S3、ESP32-C2、ESP32-H2、ESP32-H4，以及其它 NimBLE 移植平台（Mynewt / NuttOS / RIOT / Linux）。
- **协议**：BLE 5.x（Host + Controller），包括扩展广播、周期广播、2M/Coded PHY、Secure Connections、Bluetooth Mesh。
- **构建**：ESP-IDF（`idf.py`），NimBLE 作为 IDF 组件或 managed component。

## 安装

将本 Skill 克隆 / 复制到 Claude Code 的 skills 目录之一：

- **项目级**：`<project>/.claude/skills/esp-nimble-skill`
- **用户级**：`~/.claude/skills/esp-nimble-skill`

```bash
# 项目级安装示例
mkdir -p .claude/skills
cp -r esp-nimble-skill .claude/skills/
```

安装后，在涉及 NimBLE / BLE / 蓝牙的对话中，Claude 会自动加载本 Skill。

## 目录结构

```
esp-nimble-skill/
├── SKILL.md                      # 主入口：原则、配方索引、陷阱、执行流程
├── AGENTS.md                     # 工程约定：include 模式、初始化模板、构建流程
├── README.md                     # 本文件
├── CHANGELOG.md                  # 变更记录
├── recipes/                      # 场景配方（11 个）
│   ├── host_init.md
│   ├── address_setup.md
│   ├── peripheral_adv.md
│   ├── beacon.md
│   ├── notify.md
│   ├── scanner.md
│   ├── central_connect.md
│   ├── gatt_server.md
│   ├── security_pairing.md
│   ├── ext_adv.md
│   └── mesh_node.md
└── resources/                    # 快速参考文档
    ├── api_reference.md
    ├── config_reference.md
    ├── pitfalls.md
    └── example_list.md
```

## 许可

本 Skill 文档遵循仓库 LICENSE（Apache-2.0）。
