# Changelog

本项目（esp-idf-skill）所有重要变更记录于此文件。

格式参考 [Keep a Changelog](https://keepachangelog.com/)，版本号遵循 [Semantic Versioning](https://semver.org/)。

## [1.1.0] - 2026-06-18

### Added

- 补齐蓝牙与组网场景：新增 7 篇 recipe，全部 API/结构体/宏/Kconfig 符号/代码片段均取自 ESP-IDF 仓库 `components/*/include`、`docs/en/`、`examples/` 真实源码。
  - `recipes/ble_peripheral.md` — Bluedroid GATT Server：controller/Host 启动固定顺序、属性表建服务（`esp_ble_gatts_create_attr_tab`）、广播、notify/indicate。来源 `examples/bluetooth/bluedroid/ble/gatt_server_service_table`。
  - `recipes/ble_central.md` — Bluedroid GATT Client：扫描、连接（`esp_ble_gattc_enh_open`）、服务/特征值发现、读写、订阅 notify。来源 `examples/bluetooth/bluedroid/ble/gatt_client`。
  - `recipes/classic_bt_spp.md` — 经典蓝牙 SPP 串口透传（`esp_spp_enhanced_init`/`start_srv`/`write`，CB 与 VFS 模式）。来源 `examples/bluetooth/bluedroid/classic_bt/bt_spp_acceptor`。
  - `recipes/classic_bt_a2dp.md` — A2DP Sink 音频接收 + AVRCP CT 控制（`esp_a2d_sink_init`、PCM 回调、`esp_avrc_ct_*`）。来源 `a2dp_sink_stream`。
  - `recipes/ble_mesh.md` — ESP-BLE-MESH 节点（Composition Data + Generic OnOff Server + 配网承载 + publish）。来源 `examples/bluetooth/esp_ble_mesh/onoff_models/onoff_server`。
  - `recipes/wifi_mesh.md` — ESP-WIFI-MESH（`esp_mesh_init`/`set_config`/`start`，P2P 收发，根节点选举与事件）。来源 `examples/mesh/internal_communication`。
  - `recipes/ethernet.md` — 内部以太网 MAC+PHY（`esp_eth_mac_new_esp32`/`phy_new_generic`/`driver_install`/`start`，netif 挂载）。来源 `examples/ethernet/basic`。
- `SKILL.md`：新增 "蓝牙 (Bluedroid)" 与扩展 "网络/无线" recipe 分组；新增第 13、14 条核心原则（蓝牙启动固定顺序与 Classic BT 模式互斥；Mesh/以太网先建 netif 再启协议栈）；`metadata.version` 提升至 `1.1.0`。
- `resources/api_reference.md`：新增蓝牙控制器/Host、BLE GAP、GATT Server/Client、经典 SPP、A2DP/AVRCP、ESP-BLE-MESH、Wi-Fi Mesh、以太网各模块的真实函数签名；netif 部分补 `create_default_wifi_mesh_netifs`、`ESP_NETIF_DEFAULT_ETH`、`esp_netif_attach`。
- `resources/example_list.md`：新增蓝牙（Bluedroid/经典/BLE Mesh）与组网/以太网（mesh/ethernet）真实样例路径索引。
- `resources/config_reference.md`：蓝牙 Kconfig 速查补 `CONFIG_BT_CLASSIC_ENABLED`、`CONFIG_BT_A2DP_ENABLE`、`CONFIG_BT_BLUEDROID_ENABLED`/`CONFIG_BT_NIMBLE_ENABLED` 区分。

### Grounding

- 全部新 recipe 的函数、结构体、宏、Kconfig 符号、事件名均对照 ESP-IDF 仓库 `D:/esp-skill/espressif-repos/esp-idf` 的 `components/bt/host/bluedroid/api/include/api/*.h`、`components/bt/include/esp32/include/esp_bt.h`、`components/bt/esp_ble_mesh/api/*/include/*.h`、`components/esp_eth/include/*.h`、`examples/` 真实源码核对。
- 经典蓝牙（SPP/A2DP/HFP）明确标注仅 esp32 双模支持（`SOC_BT_CLASSIC_SUPPORTED`）；BLE/BLE Mesh/Wi-Fi Mesh/以太网标注适用 SoC 范围。

## [1.0.0] - 2026-06-18

### Added

- 初始版本：面向 Espressif ESP-IDF 的 AI 技能。
- `SKILL.md`：12 条核心原则、目标芯片支持矩阵（esp32/esp32s2/s3/esp32c2/c3/c5/c6/c61/esp32h2/h4/h21/esp32p4）、关键 `idf.py` 命令表、13 条关键陷阱（WRONG/CORRECT）、执行流程与失败策略。
- `AGENTS.md`：工程目录结构、顶层 `CMakeLists.txt` 固定顺序、组件 `idf_component_register` 用法、include 模式、`app_main` 模板、ISR 模板、构建工作流与代码生成 checklist。
- `recipes/`（13 篇）：`new_project`、`freertos_task`、`gpio_control`、`uart_comm`、`i2c_master`（新总线 API）、`spi_master`、`ledc_pwm`、`gptimer`、`wifi_sta`、`softap`、`nvs_storage`、`partition_table`、`ota_update`、`deep_sleep`。每篇含适用摘要、触发意图、前置条件、分步真实代码、常见错误表、参考路径。
- `resources/api_reference.md`：按模块分组的真实函数签名（系统/FreeRTOS/GPIO/UART/I2C/SPI/LEDC/GPTimer/ADC/事件循环/Netif/Wi-Fi/NVS/分区表/Flash/OTA/睡眠）。
- `resources/config_reference.md`：Kconfig 速查（目标/分区表/日志/FreeRTOS/Wi-Fi/蓝牙/Flash/HTTP/电源/构建）+ `sdkconfig.defaults` 示例。
- `resources/pitfalls.md`：分模块的陷阱汇总。
- `resources/example_list.md`：ESP-IDF `examples/` 下真实样例路径索引表。
- `README.md`：技能介绍、安装（用户/项目目录）、目录结构。
- `CHANGELOG.md`：本文件。

### Grounding

- 全部 API、结构体、宏、Kconfig 选项、文件路径取自 ESP-IDF 仓库 `D:/esp-skill/espressif-repos/esp-idf` 的 `components/*/include`、`docs/en/`、`examples/` 与根 `Kconfig`。
- 所有示例代码改编自仓库 `examples/` 真实工程（hello_world、blink、wifi/getting_started、peripherals/、storage/、system/ota、system/deep_sleep）。
