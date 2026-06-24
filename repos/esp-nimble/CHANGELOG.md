# Changelog

本文件记录 esp-nimble-skill 的版本变更。

格式基于 [Keep a Changelog](https://keepachangelog.com/)，版本号遵循 [语义化版本](https://semver.org/)。

## [1.1.0] — 2026-06-18

补齐两个经审计确认的 GAP 连接管理配方缺口。

### 新增
- `recipes/conn_param_update.md` — 连接参数更新：L2CAP（外设）与 Link-Layer（中心）两个过程、`ble_gap_update_params` 调用与返回码、`struct ble_gap_upd_params` 字段单位、`BLE_GAP_EVENT_CONN_UPDATE` 结果读取、对端 `L2CAP_UPDATE_REQ`/`CONN_UPDATE_REQ` 的 accept/reject 回调、连接时就下发参数（中心）。来源：`apps/bleprph/src/main.c`（`BLE_GAP_EVENT_CONN_UPDATE` 处理 line 206）、`apps/blecent/src/main.c`、`docs/btshell/btshell_GAP.rst`（conn-update-params line 228、l2cap-update line 272）、`docs/ble_hs/ble_gap.rst`。
- `recipes/phy_update.md` — LE PHY 选择（1M/2M/Coded S=2/S=8）：`BLE_GAP_LE_PHY_*_MASK` 位掩码与 `BLE_GAP_LE_PHY_CODED_*` 编码选项、`ble_gap_set_prefered_default_le_phy`（连接前默认）、`ble_gap_set_prefered_le_phy`（已建立连接）、`ble_gap_read_le_phy`、`BLE_GAP_EVENT_PHY_UPDATE_COMPLETE`。来源：`apps/bleprph/src/phy.c`（按键映射 + 切换 + LED 指示）、`apps/bleprph/src/main.c`（事件集成 line 268）、`docs/btshell/btshell_GAP.rst`（phy-set line 252、phy-set-default line 262、phy-read line 268）、`docs/index.rst`（2M/Coded PHY 特性）。

### 变更
- `SKILL.md`：Scenario Quick Reference 表新增两行；`metadata.version` 1.0.0 → 1.1.0。
- `resources/api_reference.md`：修正 `ble_gap_update_params` 的参数类型（`ble_gap_upd_params` 而非 `ble_gap_conn_params`）；新增 `struct ble_gap_upd_params` 字段表、PHY 掩码 / 枚举 / Coded 选项常量、`conn_update` / `conn_update_req` / `phy_updated` 事件 union 说明。
- `resources/example_list.md`：`apps/bleprph` 条目补充 `src/phy.c` 与 `CONN_UPDATE` 事件说明。

## [1.0.0] — 2026-06-18

首个正式发布版本。

### 新增
- `SKILL.md`：核心原则（12 条）、适用场景、配方索引表、GAP 角色/模式/地址类型/GATT 权限速查表、15 条关键陷阱（含错误/正确代码对照）、执行流程与失败策略表。
- `AGENTS.md`：工程约定——include 模式（区分 Mynewt 与 ESP-IDF）、ESP-IDF 标准工程结构、`app_main` 初始化模板、GAP/access_cb 回调签名、ESP-IDF 构建流程、代码生成清单、不可修改源码说明。
- `recipes/`：11 个场景配方——Host 初始化、地址配置、可连接外设、Beacon、通知/指示、扫描、中心连接、自定义 GATT 服务、SMP 配对、扩展/周期广播、Mesh 节点。每个配方含适用摘要、触发意图、前置条件、分步说明（真实代码）、常见错误表、参考示例路径。
- `resources/api_reference.md`：来自头文件的真实 API 签名，按 ble_hs / ble_hs_id / ble_gap / ble_hs_adv / ble_gatt / ble_sm / ble_store / ble_uuid / FreeRTOS NPL 分组。
- `resources/config_reference.md`：Mynewt syscfg（`BLE_*`）与 ESP-IDF Kconfig（`CONFIG_BT_NIMBLE_*`）配置项，区分来源、不臆造默认值。
- `resources/pitfalls.md`：15 条陷阱汇总。
- `resources/example_list.md`：仓库 `apps/` 全部示例 + `porting/` 移植示例与端口 + `targets/` 构建目标的索引表。
- `README.md`：中文介绍、功能特性、安装方式、目录结构。
- `CHANGELOG.md`：本文件。

### 来源
- 全部内容基于 `D:/esp-skill/espressif-repos/esp-nimble` 的 `docs/`（index.rst、ble_setup/、ble_hs/、ble_sec.rst、mesh/）、`nimble/host/include/host/` 头文件与 `apps/` 示例（bleprph、blecent、blehr、scanner、advertiser、ext_advertiser、blemesh 等）。
