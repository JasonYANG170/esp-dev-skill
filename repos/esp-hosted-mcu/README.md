# esp-hosted-mcu-skill

面向 **ESP-Hosted-MCU** 的 AI 开发技能（Claude Code / Agent skill 格式）。ESP-Hosted-MCU 让任意 host MCU 通过 SDIO / SPI / UART 把一片 ESP 芯片当作 Wi-Fi / 蓝牙 / OpenThread / Zigbee **通信协处理器**使用，控制与数据以 protobuf 编码的 RPC 在 host 与 co-processor 间传输。

本技能把仓库的真实文档（`docs/`）、host API 头文件（`host/*.h`、`host/api/include/*.h`）、Kconfig（根 `Kconfig`）与 `examples/` 整理成场景驱动的配方、API/配置快速参考与陷阱清单，**全部内容均来自仓库本身**，不存在臆造 API。

## 功能特性

- **传输搭建配方**：SPI 全双工、SDIO（1/4-bit）、UART 三种传输的完整 host + slave 搭建步骤、引脚表、链路验证。
- **Host 应用配方**：Wi-Fi STA、`ESP_HOSTED_EVENT` 事件与传输故障恢复、协处理器 BT 控制器初始化（v2.5.2+）、GPIO expander。
- **高级功能配方**：协处理器 OTA（经传输链路）、host 省电（deep sleep）、OpenThread/Zigbee RCP。
- **运行期传输配置**：`esp_hosted_*_set_config()` + `INIT_DEFAULT_HOST_*` 宏的代码范式。
- **快速参考**：`resources/api_reference.md`（真实签名）、`resources/config_reference.md`（真实 Kconfig）、`resources/pitfalls.md`（24 条陷阱）、`resources/example_list.md`（真实例程索引）、`resources/state_machine.md`（状态与序列）。
- **关键表格**：传输对比、协处理器芯片支持、SPI/SDIO 引脚映射、ESP-Hosted 帧头接口类型。

## 安装

将本技能目录克隆/复制到 Claude Code 的 skills 目录即可：

- 项目级：`<project>/.claude/skills/esp-hosted-mcu-skill/`
- 用户级：`~/.claude/skills/esp-hosted-mcu-skill/`

目录结构：

```
esp-hosted-mcu-skill/
├── SKILL.md
├── AGENTS.md
├── README.md
├── CHANGELOG.md
├── recipes/            # 11 个场景配方
└── resources/          # API / 配置 / 陷阱 / 例程 / 状态机
```

## 支持范围

- **Host**：任意 ESP 芯片（或经 port 层移植的非 ESP MCU）。
- **Co-processor（slave）**：ESP32、ESP32-C2/C3/C5/C6/C61、ESP32-S2/S3、ESP32-H2/H4。
- **传输**：SDIO 1/4-bit、SPI Full-Duplex、SPI Half-Duplex（1/2/4 线）、UART。
- **构建**：ESP-IDF >= 5.3（`idf.py`）。
- **特性**：Wi-Fi STA/SoftAP/STA+AP、Network Split、iTWT、Wi-Fi Enterprise、Wi-Fi Easy Connect（DPP）、蓝牙（NimBLE/BlueDroid，Hosted HCI 与标准 HCI）、OpenThread/Zigbee RCP、host 省电、GPIO expander、外部共存、协处理器 OTA。

> Linux host 的 ESP-Hosted 属于另一个仓库（`esp-hosted`），不在本技能范围。

## 许可

本技能内容基于 `esp-hosted-mcu`（Apache-2.0）整理。代码示例来自该仓库的文档与头文件。
