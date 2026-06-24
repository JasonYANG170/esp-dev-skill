# esp-lowcode-matter 真实示例产品索引

> 全部路径来自仓库 `products/`（`products/README.md`）与 `components/`（`components/README.md`）、`drivers/`（`drivers/README.md`）。每个产品含 `configuration/`、`main/app_main.cpp`、`main/app_driver.cpp`、`main/app_priv.h`、`main/CMakeLists.txt`、`README.md`、`sdkconfig.defaults`。

## 产品（`products/`）

| 产品路径 | 设备类型 | 一句话描述 | 主要组件 |
|---|---|---|---|
| `products/light_cw_pwm/` | 灯（CW PWM） | PWM 调冷暖白光灯（2CH CW，cold=GPIO4/warm=GPIO6） | light(LED) |
| `products/light_rgbcw_ws2812/` | 灯（RGB WS2812） | WS2812 RGB（+CW）灯，支持通断/亮度/色温/色调/饱和度多 feature | light(WS2812) |
| `products/socket/` | 单通道插座 | 单通道智能插座，按键切换 + 继电器 + WS2812 指示 | relay, button, light |
| `products/socket_2_channel/` | 双通道插座 | 双 endpoint 独立控制的智能插座 | relay, button, light |
| `products/temperature_sensor/` | 温度传感器 | SHT30 I2C 温度，10s 周期上报 `TEMPERATURE_SENSOR_VALUE` | temperature_sensor_sht30, button, light |
| `products/temperature_sensor_with_display/` | 温度+显示 | SHT30 + SSD1315 OLED，配网状态显示 | temperature_sensor_sht30, display_ssd1306, button |
| `products/occupancy_sensor/` | 占用传感器 | LD2420 雷达（UART），2s 周期上报 `OCCUPANCY_SENSOR_VALUE` | occupancy_sensor_ld2420, button, light |
| `products/thermostat/` | 温控器 | 温控（温度/制冷/制热设定点）骨架，带占位驱动 | low_code, system（骨架，待填驱动） |

### 温控器关键源文件（`products/thermostat/`，单 endpoint 多 feature）

> 详见 `recipes/thermostat.md`。温控器是**单 endpoint 多 feature** 模式（对照插座的多 endpoint 单 feature）。

| 文件 | 说明 |
|---|---|
| `products/thermostat/main/app_main.cpp` | 单 endpoint（id=1）内按 `feature_id` 三分发：`LOW_CODE_FEATURE_ID_TEMPERATURE`(4004) / `LOW_CODE_FEATURE_ID_COOLING_SETPOINT`(4005) / `LOW_CODE_FEATURE_ID_HEATING_SETPOINT`(4006)，均 `int16_t` |
| `products/thermostat/main/app_driver.cpp` | 占位驱动（`app_driver_set_temperature/set_cooling_setpoint/set_heating_setpoint` 仅 printf）+ 仓库内**覆盖全部 `LOW_CODE_EVENT_*` 最完整**的事件 switch |
| `products/thermostat/main/app_priv.h` | 驱动函数原型 |
| `products/thermostat/configuration/data_model_wifi.zap` | endpoint 1 = endpointType `deviceTypeRef.code=769`（MA-thermostat）；Thermostat cluster（code 513，`LocalTemperature`=0 / `OccupiedCoolingSetpoint`=17 / `OccupiedHeatingSetpoint`=18）；`FeatureMap` 默认 3（Heating+Cooling） |
| `products/thermostat/configuration/data_model_thread.zap` | Thread 版同构数据模型 |
| `products/thermostat/configuration/product_info.json` | `"device_type_id": 769`、`"product_type": "thermostat"` |
| `products/thermostat/configuration/product_config.json` | `test_mode[]`（common/ble/sniffer/low_code）+ `device_management: true` |
| `products/thermostat/README.md` | 温控器模板说明（Heating + Cooling 特性） |
| `products/template/` | 通用模板 | 仅 root node，最小 setup/loop 骨架，作为新产品起点 | low_code, system |

## 组件（`components/`，来自 `components/README.md`）

| 组件 | 头文件 | 说明 |
|---|---|---|
| `components/button/` | `button_driver.h` | 按键（HP/LP GPIO），单击/长按回调 |
| `components/display_ssd1306/` | `display_ssd1306.h` | SSD1306 OLED I2C 显示 |
| `components/light/` | `light_driver.h` + `color_format.h` | LED(PWM)/WS2812 灯光，多通道与特效 |
| `components/low_code/` | `low_code.h` | 核心事件/特性收发 |
| `components/low_code_transport/` | — | HP/LP 核间通信传输层 |
| `components/sw_timer/` | `sw_timer.h` | LP Core 软件定时器 |
| `components/occupancy_sensor_ld2420/` | `occupancy_sensor_ld2420.h` | LD2420 UART 占用传感器 |
| `components/relay/` | `relay_driver.h` | GPIO 继电器 |
| `components/system/` | `system.h` | LP Core 系统工具（GPIO/延时/定时器） |
| `components/temperature_sensor_sht30/` | `temperature_sensor_sht30.h` | SHT30 I2C 温度传感器 |

## 外设驱动（`drivers/`，来自 `drivers/README.md`）

| 驱动 | 说明 |
|---|---|
| `drivers/i2c/` | HP/LP I2C master（SHT30、SSD1306） |
| `drivers/rmt/` | LP Core RMT 发送（WS2812） |
| `drivers/uart/` | HP/LP UART RX/TX（LD2420） |

## HP 预编译镜像与工具

| 路径 | 说明 |
|---|---|
| `pre_built_binaries/` | HP Core 预编译镜像 + `flash_args` |
| `tools/mfg/` | 证书/QR 码生成（`mfg_low_code.sh`） |

## 文档（`docs/`）

| 文档 | 说明 |
|---|---|
| `docs/getting_started_terminal.md` | 本地终端全流程 |
| `docs/getting_started_vscode.md` | VS Code 扩展流程 |
| `docs/hardware_setup.md` | 硬件与串口权限 |
| `docs/programmer_model.md` | 编程模型（HP/LP 分核、消息） |
| `docs/create_product.md` | 创建/定制产品 + Arduino 映射表 |
| `docs/product_configuration.md` | 产品配置与数据模型 |
| `docs/device_setup.md` | 设备配网与控制 |
| `docs/debugging.md` | LP Core panic 调试 |
| `docs/production_considerations.md` | 生产考量（仓库内多为 TODO 占位） |
| `docs/matter_solutions.md` | ZeroCode/LowCode/ESP-Matter/Connectedhomeip 对比 |
| `docs/all_documents.md` | 文档索引 |
