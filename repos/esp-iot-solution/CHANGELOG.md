# Changelog

## [1.1.0] - 2026-06-18

### Added

- 新增 5 个场景 recipe，填补 BLE / FOC / 灯带 三大缺口（此前 SKILL.md 描述提及但无 recipe）：
  - `recipes/ble_conn_mgr.md` — `esp_ble_conn_mgr` 简化 BLE API：外设/中心角色、周期广播、周期同步、SPP（服务端/客户端）、L2CAP CoC（外设/中心）。来源示例：`examples/bluetooth/ble_conn_mgr/{ble_periodic_adv,ble_periodic_sync,ble_spp}`、`examples/bluetooth/ble_l2cap_coc/{l2cap_coc_peripheral,l2cap_coc_central}`。
  - `recipes/ble_profiles_ota.md` — `ble_profiles` GATT profile：`esp_ble_ota_raw`（服务 UUID `0x8018`，扇区 CRC OTA 固件升级，含 partitions.csv/sdkconfig）与 `esp_ble_htp`（健康体温计，服务 `0x1809`）。来源：`examples/bluetooth/ble_profiles/{ble_ota,ble_htp,ble_hrp}`。
  - `recipes/foc_motor.md` — `esp_simplefoc` FOC 无刷电机控制：开环 `velocity_openloop` 与闭环 `velocity`（AS5600 + PID），MCPWM/LEDC 自动选择。来源：`examples/motor/{foc_openloop_control,foc_velocity_control,foc_knob_example}`。
  - `recipes/bthome.md` — BTHome V2 协议对接 Home Assistant：加密广播端（`bthome_make_adv_data`）与解析端（`bthome_parse_adv_data`），含 NVS 计数器持久化防重放。来源：`examples/bluetooth/ble_adv/bthome/{bulb,dimmer}`。
  - `recipes/led_indicator_strips.md` — RGB LED 与 WS2812 灯带：`LED_BLINK_RGB/HSV`、`LED_BLINK_*_RING` 渐变、`SET_IHSV/SET_IRGB/INSERT_INDEX` 灯带 index 控制。来源：`examples/indicator/{rgb,ws2812_strips}`。
- `SKILL.md`：新增「蓝牙 (BLE)」场景分组表（3 条）；输出与指示组补 `led_indicator_strips`；电机组补 `foc_motor`；版本 `1.0.0` → `1.1.0`；关键组件/API 速览表补 7 行（ble_conn_mgr、ble_profiles ota_raw/htp、bthome_v2、esp_simplefoc、led_indicator rgb/strips）。
- `resources/api_reference.md`：新增 6 个组件章节的真实签名（ble_conn_mgr 完整事件/GATT/L2CAP API、esp_ble_ota_raw、esp_ble_htp、bthome_v2、esp_simplefoc C++ 类、led_indicator RGB/Strips 后端 + led_convert.h 颜色宏）。
- `resources/example_list.md`：bluetooth 小节补 `ble_adv/bthome/{bulb,dimmer}`、`ble_profiles/{ble_ota,ble_htp,ble_hrp}` 等细粒度示例路径。

### Grounding

- 所有新增 API / 结构体 / 宏 / Kconfig / 示例路径均取自仓库 `components/bluetooth/{ble_conn_mgr,ble_profiles,ble_adv/bthome}`、`components/led/led_indicator/include/`、`examples/{bluetooth,motor,indicator}/`、`docs/en/{bluetooth,motor,display}/` 真实内容。

## [1.0.0] - 2026-06-18

### Added

- 初始版本。
- `SKILL.md`：技能元数据、核心原则（11 条）、适用/不适用、12 个 recipe 索引表、组件版本/IDF 兼容表、关键组件 API 速览、13 条 Critical Pitfalls（每条含 WRONG/CORRECT 代码块）、执行工作流、失败策略。
- `AGENTS.md`：项目上下文、文件命名、include 模式、标准工程结构、`app_main` 入口约定、构建工作流、代码生成清单、日志约定、勿改说明。
- `recipes/`：13 个场景 recipe（i2c_bus、spi_bus、button_gpio、button_power_save、knob、led_indicator_gpio、led_indicator_ledc、sensor_hub、power_measure、usb_stream、usbh_cdc、servo、touch_button）。
- `resources/api_reference.md`：按模块分组的真实 API 速查（button、knob、led_indicator、i2c_bus、spi_bus、sensor_hub、power_measure、usb_stream、iot_usbh_cdc、servo、touch_button）。
- `resources/config_reference.md`：真实 Kconfig 符号（i2c_bus、button 等）与依赖管理说明。
- `resources/pitfalls.md`：常见坑点合集。
- `resources/example_list.md`：仓库 `examples/` 真实示例路径索引。
- `README.md`：中文技能介绍与安装说明。
- 所有 API、结构体、宏、配置项、示例路径均源自仓库 `docs/` 与 `components/`、`examples/` 真实内容。

### Grounding

- 上游仓库：esp-iot-solution（master 分支）
- 依赖 ESP-IDF v5.3+
- 许可证：Apache-2.0
