# Changelog

## [1.1.0] - 2026-06-18

### Added

- 4 个新场景 recipe（填补 audit 确认的覆盖空白）：
  - `recipes/controller_node.md` — 控制器节点 + User Helper API：`esp_rmaker_controller_enable` + `CONFIG_ENABLE_RM_USER_HELPER_API`，以用户身份经云端操作其它节点（list/get/set params、schedules、config、status、removenode）；含 auth service refresh token 登录与 CLI 命令表。来源 `examples/rainmaker_controller/`、`examples/common/rmaker_user_api/`、`components/esp_rainmaker/include/esp_rmaker_controller.h`。
  - `recipes/ethernet_connectivity.md` — 以太网主传输 + 可选双网络（`EXAMPLE_ENABLE_WIFI`）+ on-network challenge-response 配网（独立 HTTP 或复用 local control 两种实现，互斥）；PHY/RMII GPIO 配置、`app_ethernet_init/start`、No Claim。来源 `examples/ethernet_switch/`（含 `components/app_ethernet/`、`main/app_on_network_test.c`）。
  - `recipes/thread_border_router.md` — Thread 边界路由器（TBR）：`esp_rmaker_thread_br_enable(platform_config)` 启用 BR 服务，RainMaker-over-Thread 设备经 NAT64 上云；ESP32-S3 主控 + ESP32-H2 RCP 分体、RCP 自动更新、`TBRService.ThreadCmd`/`ActiveDataset` CLI、LwIP IPv6 与 mbedTLS 编译要求。来源 `examples/thread_br/`、`components/esp_rainmaker/include/esp_rmaker_thread_br.h`。
  - `recipes/zigbee_gateway.md` — Zigbee 网关：运行时把加入的 Zigbee 终端动态映射为 RainMaker 设备（按 HA device id 分派），两种加设备方式（预共享 key `ZigBeeAlliance09` / install code JSON + `ESP_RMAKER_UI_QR_SCAN`），NVS 持久化与重启恢复。来源 `examples/zigbee_gateway/`。

### Changed

- `SKILL.md`：metadata.version 1.0.0 → 1.1.0；Scenario Quick Reference 新增"网络与连接"分组（ethernet）与"进阶"扩展（controller_node / zigbee_gateway / thread_border_router）；新增 Core Principle #13（运行时增删设备必须 `esp_rmaker_report_node_details`）与 #14（节点角色三态：业务节点 / 控制器节点 / 网关或 BR）；When to Use 补充以太网/控制器/网关/BR。
- `resources/example_list.md`：扩充 `ethernet_switch` / `thread_br` / `zigbee_gateway` / `rainmaker_controller` / `rmaker_user_api` 五条目的真实细节（Kconfig、API、CLI 命令、硬件分体）。
- `resources/api_reference.md`：新增 Controller、Thread Border Router、Auth Service、User API（core+helper）、On-network Challenge-Response、网关/控制器/BR 相关标准类型宏 六个章节（仅仓库头文件/示例组件中验证过的签名）。

### Grounding

- 全部新增函数签名、结构体、宏、Kconfig、CLI 命令均取自 `esp-rainmaker` 仓库真实文件（`components/esp_rainmaker/include/*.h`、`examples/rainmaker_controller/`、`examples/ethernet_switch/`、`examples/thread_br/`、`examples/zigbee_gateway/`、`examples/common/rmaker_user_api/`）。`esp.device.controller` 设备类型字符串在仓库中为字面量，未定义为宏，已在 api_reference 中注明。

## [1.0.0] - 2026-06-18

### Added

- 初始版本，面向 ESP RainMaker（`esp-rainmaker` 仓库 `components/esp_rainmaker`，组件 version 1.15.0）。
- `SKILL.md`：核心原则、When to Use、场景快速索引、芯片/Claiming 支持矩阵、关键 Kconfig 速查、Agent 状态机、20 条 Critical Pitfalls（含错误/正确代码对照）、执行工作流、失败策略。
- `AGENTS.md`：项目上下文、文件命名与 include 模式、标准工程结构、canonical `app_main()` 与 write 回调模板、构建流程、代码生成 checklist、Do Not Modify 说明。
- `recipes/`：10 个场景化 recipe
  - `getting_started.md`（节点骨架）
  - `claiming_and_provisioning.md`（Claiming/配网/user mapping/chal_resp）
  - `custom_device.md`（自定义设备/参数/UI Type/bounds/valid_str/时间序列）
  - `multi_device.md`（多设备节点/动态增删）
  - `standard_devices.md`（标准设备/参数/服务 helper 与宏）
  - `ota_update.md`（默认/自定义 OTA、事件、回滚诊断、Kconfig）
  - `scheduling_scenes.md`（调度/场景/触发源/日光）
  - `services.md`（时区/系统/Connectivity/Groups 服务）
  - `local_control.md`（本地控制/PoP/sec0-2/chal_resp 端点）
  - `mqtt_topics.md`（直发/预算/Basic Ingest/合并上报）
- `resources/api_reference.md`：按模块分组的真实函数签名（Core/Standard Params/Devices/Services/OTA/MQTT/Schedule/Scenes/Connectivity/Groups/User Mapping/Console）。
- `resources/config_reference.md`：全部 `CONFIG_ESP_RMAKER_*` Kconfig 符号（Claiming/MQTT/安全/本地控制/OTA/时间调度场景/命令-响应/控制台/网络）。
- `resources/pitfalls.md`：20 条陷阱合集。
- `resources/example_list.md`：13 个官方示例 + 7 个共用组件的真实路径与说明。
- `README.md`：中文介绍、安装方式、支持范围、目录结构。

### Grounding

- 所有 API、结构体、宏、Kconfig 符号、示例路径均取自 `esp-rainmaker` 仓库真实文件（头文件、`Kconfig.projbuild`、`examples/*/main/app_main.c`、`idf_component.yml`）。
- 未发现的内容一律省略，未臆造任何 API。
