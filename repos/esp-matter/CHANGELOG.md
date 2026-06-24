# Changelog

本文件记录 esp-matter-skill 的发布历史。格式参考 [Keep a Changelog](https://keepachangelog.com/)，版本号遵循语义化版本。

## [1.1.0] - 2026-06-18

补齐 5 个经审计确认的高价值 recipe 缺口。所有内容基于 esp-matter 仓库真实文档与示例源码（controller.rst、optimizations.rst、examples/controller、examples/thread_border_router、examples/icd_app、examples/refrigerator、examples/room_air_conditioner、examples/all_device_types_app、esp_matter_endpoint_impl.h、esp_matter_cluster_impl.h、esp_matter_controller_client.h、Kconfig）。

### 新增

- **recipes/controller_ondevice.md** — 设备端 Matter Controller / Commissioner：`matter_controller_client::init(node_id, fabric_id, port)` + `setup_commissioner()`、client-only Kconfig、`matter esp controller pairing/invoke-cmd/read-attr/write-attr/subs-attr/group-settings`、Attestation Trust Store（Test/Spiffs/DCL/Custom）、NOC Issuer（Test/Custom）、JSON `<TagNumber>:<DataType>` 格式。源：`examples/controller/`、`docs/en/controller.rst`、`components/esp_matter_controller/Kconfig` + `core/esp_matter_controller_client.h`。
- **recipes/optimizations.md** — RAM/Flash 优化：12+ 项 Kconfig 的 measured before/after 表（esp32h2 / esp32c3 / light 基准）、chip-shell / dynamic endpoint / newlib nano / BLE NimBLE / 事件日志 buffer / IRAM→flash（`BT_CTRL_RUN_IN_FLASH_ONLY`、`FREERTOS_PLACE_FUNCTIONS_INTO_FLASH`、`RINGBUF_PLACE_FUNCTIONS_INTO_FLASH`、`SPI_FLASH_ROM_IMPL`、`HEAP_PLACE_FUNCTION_INTO_FLASH`）/ 任务栈 / 关未用 `CONFIG_SUPPORT_*_CLUSTER` / LTO / SPIRAM BSS 外置。源：`docs/en/optimizations.rst`、`examples/controller/sdkconfig.defaults.ram_optimization`、`examples/light`、`examples/multiple_on_off_plugin_units`。
- **recipes/thread_border_router.md** — Thread Border Router（ESP32-S3 + ESP32-H2）：烧 ot_rcp / `CONFIG_AUTO_UPDATE_RCP` 自动更新、BR sdkconfig（`OPENTHREAD_BORDER_ROUTER` / LwIP 转发 / route hook 关闭）、`esp_openthread_border_router_init()`、commission BR 后 ThreadBorderRouterManagement 三步（arm-fail-safe → set-active-dataset-request → commissioning-complete）、controller console `ot_cli dataset`、commission Thread 终端、RIO/TC-SC-4.9。源：`examples/thread_border_router/`、`examples/controller/sdkconfig.defaults.otbr` + `app_main.cpp`、`docs/en/controller.rst`、`docs/en/certification.rst`。
- **recipes/icd_device.md** — ICD（SIT/LIT）：参数对比表、`CONFIG_ENABLE_ICD_SERVER` + `ENABLE_ICD_LIT`、配套 PM/tickless/15.4 sleep/BLE sleep Kconfig、`esp_pm_configure()`、`ENABLE_USER_ACTIVE_MODE_TRIGGER_BUTTON` 按键唤醒、Matter 1.4 LIT-需-client-注册 规则、H2/C6 电流波形图索引。源：`examples/icd_app/`（`app_main.cpp`、`Kconfig.projbuild`、`sdkconfig.defaults*`、`image/`）。
- **recipes/hvac_appliances.md** — 暖通/家电 device type：Room Air Conditioner（OnOff + Thermostat）、Thermostat cluster 的 heating/cooling O.a+ feature conformance（`feature::heating::get_id()`）、Refrigerator + TemperatureControlledCabinet 父子 endpoint（`set_parent_endpoint`）、Pump `config_t(max_pressure, max_speed, max_flow)` 构造、TemperatureControl / ModeBase 等 delegate cluster 列表。源：`examples/refrigerator/`、`examples/room_air_conditioner/`、`examples/all_device_types_app/`、`esp_matter_endpoint_impl.h`、`esp_matter_cluster_impl.h`、`docs/en/developing.rst`、`docs/en/app_guide.rst`。

### 更新

- **SKILL.md**：版本 `1.0.0 → 1.1.0`；新增 Core Principle 13（controller/commissioner 是独立 client-only 固件形态）；在 Scenario Quick Reference 表的 "Commissioning & Control" 加 `controller_ondevice.md`、"Device Types" 加 `hvac_appliances.md`，并新增 "Optimization, Thread & Low-Power" 分组（`optimizations.md` / `thread_border_router.md` / `icd_device.md`）。
- **resources/example_list.md**：给 `examples/controller`、`thread_border_router`、`icd_app`、`refrigerator`、`room_air_conditioner`、`all_device_types_app`、`light`、`multiple_on_off_plugin_units` 八项加上对应新 recipe 的交叉引用。
- **resources/api_reference.md**：新增 "On-device Controller / Commissioner"（`matter_controller_client` 签名 + Kconfig + 命令行）、"Thermostat / Pump / 家电 cluster config（feature flag）"、"家电 endpoint config（refrigerator / cabinet / room_ac / pump）"、`set_parent_endpoint`、"ICD Kconfig" 五节。

### Grounding

新增内容均取自：
- `docs/en/controller.rst`、`docs/en/optimizations.rst`、`docs/en/developing.rst`、`docs/en/app_guide.rst`、`docs/en/certification.rst`
- `examples/controller/`（`main/app_main.cpp`、`sdkconfig.defaults*`、`partitions.csv`、`README.md`）、`examples/thread_border_router/`、`examples/icd_app/`、`examples/refrigerator/`、`examples/room_air_conditioner/`
- `components/esp_matter_controller/`（`Kconfig`、`core/esp_matter_controller_client.h`）
- `components/esp_matter/data_model/legacy/esp_matter_endpoint_impl.h`、`esp_matter_cluster_impl.h`、`data_model/esp_matter_data_model.h`

## [1.0.0] - 2026-06-18

首个发布版本。面向 Espressif esp-matter（Matter / CHIP）固件开发的 AI Skill，全部内容基于 esp-matter 仓库的真实文档与源码。

### 新增

- **SKILL.md**：12 条核心原则、何时用、12 个 recipe 的快速索引、芯片/规范支持表、默认入网凭据表、节点生命周期与回调签名、20 条关键陷阱（每条含错误/正确代码对照）、执行流程表与失败策略。
- **AGENTS.md**：项目上下文、文件命名、include 模式、标准工程结构、`app_main` 范式、属性更新回调范式、构建流程、代码生成 checklist、Do-Not-Modify 说明、host 工具说明。
- **recipes/**（12 个场景，中文叙事 + 真实代码 + 常见错误表 + 真实参考路径）：
  - `setup_and_build.md` — 环境搭建与首次构建
  - `device_data_model.md` — Node/Endpoint/Cluster/Attribute/Command 数据模型
  - `custom_cluster.md` — 自定义厂商 cluster（XML + codegen）
  - `commissioning_chiptool.md` — chip-tool 入网与控制
  - `device_console.md` — 设备端 `matter` shell
  - `lighting.md` — On/Off / Dimmable / Color Temperature / Extended Color 灯具
  - `sensors.md` — 温/湿/占用传感器与 `ScheduleLambda` 异步上报
  - `switches_binding.md` — On/Off Light Switch 与 Binding
  - `door_lock_window_covering.md` — 门锁与窗帘（含 EndProductType）
  - `bridge_zigbee.md` — Aggregator + 动态 bridged endpoint 桥接
  - `factory_data_attestation.md` — esp-matter-mfg-tool 工厂分区与四类 Provider
  - `matter_ota.md` — OTA Requestor（含加密 OTA）
- **resources/**（5 个速查文档）：
  - `api_reference.md` — 真实函数签名（core/node/endpoint/cluster/attribute/command/event/identify/ota/client）
  - `config_reference.md` — 全部 esp-matter Kconfig 选项
  - `pitfalls.md` — 20 条陷阱汇总
  - `example_list.md` — `examples/` 真实工程索引
  - `state_machine.md` — Matter 节点事件流与回调
- **README.md**：中文介绍、功能特性、安装方式、目录结构、使用方式。
- **CHANGELOG.md**：本文件。

### 支持范围

- ESP32 / ESP32-C2/C3/C5/C6/C61 / ESP32-H2 / ESP32-S3 / ESP32-P4
- ESP-IDF v5.5.4 + connectedhomeip 子模块（pin `107f97a9813`）
- Matter 规范 v1.0–v1.5（release 分支）/ v1.6（main）
- License：Apache-2.0

### Grounding

所有函数名、结构体、宏、Kconfig 选项、文件路径、代码片段均取自：
- `D:/esp-skill/espressif-repos/esp-matter/docs/en/`（`index.rst`、`developing.rst`、`controller.rst`、`app_guide.rst`、`production.rst`、`api-reference/`）
- `D:/esp-skill/espressif-repos/esp-matter/components/esp_matter/`（`esp_matter_core.h`、`esp_matter_ota.h`、`esp_matter_identify.h`、`Kconfig`，及 `data_model/` 下的 `esp_matter_data_model.h` 与 `legacy/*_impl.h`）
- `D:/esp-skill/espressif-repos/esp-matter/examples/`（`light`、`light_switch`、`sensors`、`door_lock`、`bridge_apps`、`ota_provider` 等的 `app_main.cpp` / `README.md`）
