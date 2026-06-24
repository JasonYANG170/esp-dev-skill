# Changelog

本 Skill 遵循 esp-lowcode-matter 仓库的真实 API 与文档。所有变更仅涉及本 Skill 自身。

## [1.1.0] — 2026-06-18

补齐温控器（Thermostat）产品场景——`products/README.md` 中此前唯一无专属 recipe 的产品。

### 新增

- **recipes/thermostat.md**：温控器产品（MA-thermostat，`device_type_id` 769）。覆盖**单 endpoint 多 feature** 模式（对照 `recipes/relay_socket.md` 的多 endpoint 单 feature）：同一 `endpoint_id == 1` 内按 `feature_id` 三分发 `LOW_CODE_FEATURE_ID_TEMPERATURE`(4004) / `LOW_CODE_FEATURE_ID_COOLING_SETPOINT`(4005) / `LOW_CODE_FEATURE_ID_HEATING_SETPOINT`(4006)，均为带符号 `int16_t`（°C×100，必须用 `LOW_CODE_VALUE_TYPE_INTEGER`）；含 Thermostat cluster（code 513）属性表（`LocalTemperature`/`OccupiedCoolingSetpoint`/`OccupiedHeatingSetpoint` 及 AbsMin/AbsMax 限值）、`product_info.json` 的 `device_type_id: 769`、当前温度主动上报模式、事件 switch 指引、常见错误表（含 AbsMin/AbsMax 越限、负温解析错误等）。

### 更新

- **SKILL.md**：版本 `1.0.0` → `1.1.0`；在「传感器与显示（周期上报）」速查表新增 `recipes/thermostat.md` 行。
- **resources/example_list.md**：在 `products/thermostat/` 条目下补「温控器关键源文件」小节，索引 `app_main.cpp`（三 feature 分发）、`app_driver.cpp`（占位驱动 + 完整事件 switch）、`app_priv.h`、`data_model_wifi.zap`（endpoint 1 = MA-thermostat 769 / Thermostat cluster 513 / FeatureMap=3）、`data_model_thread.zap`、`product_info.json`（`device_type_id: 769`）、`product_config.json`、`README.md`。
- **resources/api_reference.md**：新增 §11「温控器（Thermostat）应用层模式」，列出 `app_priv.h` 的 `app_driver_set_temperature/set_cooling_setpoint/set_heating_setpoint(int16_t)` 原型（仓库占位实现）及数据模型要点（endpoint 1 / device type 769 / cluster 513 / FeatureMap=3）。

### Grounding

- 全部函数名、feature_id、枚举、cluster/attribute code、`device_type_id` 均取自 `products/thermostat/`（`app_main.cpp`、`app_driver.cpp`、`app_priv.h`、`data_model_wifi.zap`、`product_info.json`、`README.md`）与 `components/low_code/low_code.h`。
- `products/thermostat` 为骨架产品，`app_driver.cpp` 中 `app_driver_set_*` 仅为 printf 占位（仓库 README 已声明），recipe 如实标注「占位驱动」，未杜撰温控硬件实现。

## [1.0.0] — 2026-06-18

首个正式版本，内容完全基于 esp-lowcode-matter 仓库源码（`components/*`、`drivers/*`、`products/*`）与官方文档（`docs/*`、`README.md`）。

### 新增

- **SKILL.md**：YAML front matter（name/description/trigger words/tags/license/compatibility/version）+ 12 条核心原则、When to Use、16 篇 recipes 速查表、组件/驱动清单、feature_id 与 event 速查、产品默认引脚映射、数据流图、14 条 Critical Pitfalls（WRONG/CORRECT 对照）、Execution Workflow、Failure Strategies、References。
- **AGENTS.md**：项目上下文（C/C++、ESP32-C6、ESP-IDF v5.3 + ESP-AMP）、命名/包含模式、标准产品结构、setup/loop/main 骨架、事件处理模板、本地构建/烧录工作流、代码生成 checklist、Do Not Modify 清单。
- **recipes/（16 篇）**：
  - `getting_started.md`（本地环境与首次烧录）
  - `create_product.md`（创建新产品）
  - `setup_loop_model.md`（编程骨架）
  - `feature_update.md`（特性收发，feature_id 与 matter 低层两种方式）
  - `event_handling.md`（系统事件与工厂复位上报）
  - `gpio_system.md`（system_* GPIO + Arduino 映射）
  - `button_driver.md`（按键单击/长按）
  - `light_driver.md`（LED/WS2812 + 特效）
  - `relay_socket.md`（继电器/单/双通道插座）
  - `system_timer.md`（周期上报定时器）
  - `sht30_sensor.md`（SHT30 温度）
  - `ld2420_occupancy.md`（LD2420 雷达占用）
  - `ssd1306_display.md`（OLED 显示）
  - `product_configuration.md`（product_info / zap）
  - `debugging.md`（panic 定位与日志）
- **resources/**：
  - `api_reference.md`（10 个模块的真实 API 签名）
  - `config_reference.md`（product_info 字段、Kconfig、CMake、引脚、烧录地址）
  - `pitfalls.md`（30 条归类陷阱）
  - `example_list.md`（全部产品/组件/驱动/文档真实路径索引）
- **README.md** / **CHANGELOG.md**。

### Grounding

- 所有函数名、结构体、枚举、宏、配置项、引脚、烧录地址均来自仓库源码或文档。
- 代码片段改编自 `products/socket`、`light_cw_pwm`、`light_rgbcw_ws2812`、`temperature_sensor`、`temperature_sensor_with_display`、`occupancy_sensor`、`socket_2_channel`、`thermostat`、`template` 等真实示例。
- 仓库文档较薄处（如 `docs/production_considerations.md` 仍为 TODO）如实标注，未杜撰内容。
