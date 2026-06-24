# CHANGELOG

## [1.1.0] — 2026-06-18

补齐审计确认的 5 个高价值功能缺口，新增对应 recipes 与 API 参考。全部基于仓库真实文档与头文件，未臆造。

### 新增 recipes（5）
- `recipes/host_ext_coex.md` — 外部共存 EXT_COEX：从 host 配置 slave PTA（1/2/3/4-wire、leader/follower、grant_delay/validate_high、host↔slave API 映射、BT 共存限制）。源：`examples/host_manage_copro_ext_coex/`、`host/api/include/esp_hosted_cp_ext_coex.h`。
- `recipes/network_split.md` — Network Split：host 与 slave 共享 IP，按端口分流，路由决策矩阵，MQTT `wakeup-host` 唤醒，`nw_split_router.c` 自定义。仅 C5/C6/S2/S3。源：`docs/feature_network_split.md`、`examples/host_network_split__power_save/`。
- `recipes/wifi_itwt.md` — Wi-Fi 6 iTWT 省电：`esp_wifi_sta_itwt_setup` 协商、Kconfig 符号与取值范围、iTWT 事件、`itwt>`/`probe` 控制台命令、RPC 列表。仅 C5/C6。源：`examples/host_wifi_itwt/`（main.c / wifi_itwt_cmd.c / Kconfig.projbuild）。
- `recipes/wifi_dpp_enrollee.md` — Wi-Fi Easy Connect (DPP) Enrollee：QR 码扫码入网、5 个 `esp_supp_dpp_*` API 与 RPC、v5.5/v6.0 事件分发差异。源：`examples/host_wifi_easy_connect_dpp_enrollee/`、`docs/features.md`、`docs/implemented_rpcs.md`。
- `recipes/peer_custom_data.md` — host↔协处理器自定义二进制通道：`esp_hosted_send_custom_data` / `register_custom_callback`、msg_id 配对、v2.12.4 回调签名变更、8166 字节上限。源：`examples/host_peer_data_transfer/`、`docs/migration_guide.md`。

### SKILL.md 变更
- `metadata.version`：1.0.0 → 1.1.0。
- Scenario Quick Reference：Host Application 表新增 `wifi_dpp_enrollee`；Advanced Features 表新增 `network_split`、`wifi_itwt`、`host_ext_coex`、`peer_custom_data`。
- When to Use：补充 Network Split、iTWT、DPP、peer/custom data、EXT_COEX 适用场景。
- Core Principles：新增 #13（EXT_COEX + BT 共存取决于 slave 芯片与 IDF 版本）。

### resources 变更
- `api_reference.md`：新增 §11 Wi-Fi iTWT API（setup/teardown/suspend/probe 事件 + RPC）、§12 Wi-Fi Easy Connect DPP API（5 个 `esp_supp_dpp_*` + RPC + 事件并入 `WIFI_EVENT_DPP_*` 说明）。
- `example_list.md`：改进 `host_peer_data_transfer` 描述（host↔协处理器二进制通道）。

### 接地说明
所有新增函数/结构/宏/Kconfig 符号/RPC ID 均取自 `host/api/include/*.h`、各例程 `main.c` 与 `Kconfig.projbuild`、`docs/feature_*.md`、`docs/implemented_rpcs.md`、`docs/migration_guide.md`。

## [1.0.0] — 2026-06-18

首个版本。基于 `esp-hosted-mcu` 仓库（idf_component 版本 2.12.9）的真实文档（`docs/`）、host API 头文件（`host/*.h`、`host/api/include/*.h`）、根 `Kconfig` 与 `examples/` 整理而成。

### 新增
- `SKILL.md`：核心原则、传输对比、协处理器芯片支持、SPI/SDIO 引脚映射、ESP-Hosted 帧头接口类型、12 条带 WRONG/CORRECT 代码的 Critical Pitfalls、执行工作流、失败策略。
- `AGENTS.md`：仓库布局、文件/头文件约定、include 模式、标准 host 初始化范式、构建工作流、codegen 清单、Do Not Modify 说明。
- `recipes/`（11 个）：
  - `bringup_spi_fd.md` — SPI 全双工搭建
  - `bringup_sdio.md` — SDIO 1/4-bit 搭建与 Stream/Packet 模式
  - `bringup_uart.md` — UART 传输搭建（Wi-Fi+BT）
  - `transport_config_code.md` — 运行期 `esp_hosted_*_set_config()`
  - `host_wifi_sta.md` — host Wi-Fi STA 应用
  - `host_events_recovery.md` — ESP_HOSTED 事件、心跳与传输故障恢复
  - `host_bt_controller.md` — 协处理器 BT 控制器初始化（v2.5.2+）
  - `host_gpio_expander.md` — GPIO expander
  - `slave_ota.md` — 协处理器 OTA
  - `host_power_save.md` — host 省电（deep sleep）
  - `openthread_zigbee_rcp.md` — OpenThread/Zigbee RCP
- `resources/`：
  - `api_reference.md` — host API 真实签名（按模块）
  - `config_reference.md` — 真实 Kconfig 符号
  - `pitfalls.md` — 24 条陷阱汇总
  - `example_list.md` — `examples/` 真实例程索引
  - `state_machine.md` — host/slave 状态、SPI 握手、OTA、省电流
- `README.md`、`CHANGELOG.md`。

### 接地说明
所有函数名、结构/枚举、宏、Kconfig 符号、文件路径、引脚表与代码片段均取自仓库文档或头文件；文档薄弱处采用更少的配方与更短的参考，未臆造任何 API。
