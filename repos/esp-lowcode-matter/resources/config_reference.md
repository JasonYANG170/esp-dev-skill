# esp-lowcode-matter 配置速查

> 全部来自仓库 `products/*/configuration/`、`products/*/sdkconfig.defaults`、`components/*/Kconfig` 与 `docs/`。

## 1. `product_info.json` 字段（见 `docs/product_configuration.md`，示例 `products/socket`）

| 字段 | 示例值 | 说明 |
|---|---|---|
| `config_version` | 3 | 配置版本 |
| `vendor_id` | 65521 | Matter Vendor ID（测试范围） |
| `product_id` | 32768 | Matter Product ID |
| `origin_vendor_id` | 65521 | 原始 Vendor ID |
| `origin_product_id` | 32768 | 原始 Product ID |
| `device_type_id` | 266 | Matter 设备类型 ID（插座示例；灯/温控/传感器各有不同） |
| `vendor_name` | "Espressif" | 厂商名（生态 App 展示） |
| `product_name` | "Matter Product" | 产品名 |
| `hw_ver` / `hw_ver_str` | 1 / "1" | 硬件版本 |
| `chip` | "esp32c6" | **当前必须 esp32c6** |
| `connection_type` | "wifi" | "wifi" 或 "thread"（对应 `data_model_wifi.zap` / `data_model_thread.zap`） |
| `module` | "ESP32-C6-MINI-1" | 模组 |
| `flash_size` | "4MB" | Flash 大小 |
| `secure_boot` | "enabled" | 安全启动 |
| `product_type` | "socket" | 产品类型 |
| `solution_type` | "low_code" | 方案类型 |

> data model 文档（`docs/product_configuration.md`）给出的示例值：`device_type_id: 268`，`vendor_name: "Espressif"`, `product_name: "Matter Product"`。实际以你的产品 zap 为准。

## 2. `product_config.json`（示例 `products/socket`）

| 键 | 说明 |
|---|---|
| `config_version` | 3 |
| `test_mode[]` | 测试模式数组，每项 `{ "type", "subtype", ... }`；`type` 形如 `ezc.test_mode.common` / `.ble` / `.sniffer` / `.low_code`；`low_code` 可带 `ssid`，`sniffer` 可带 `trigger` |
| `device_management` | bool |

`test_mode[].subtype` 会经 `LOW_CODE_EVENT_TEST_MODE_*` 事件的 `event_data` 传给应用。

## 3. Kconfig 选项

### button（`components/button/Kconfig`）

| 选项 | 默认 | 说明 |
|---|---|---|
| `BUTTON_DRIVER_USE_HP_GPIO` | y | HP GPIO 作按键 |
| `HP_BUTTON_LOOP_INTERVAL` | 50 | HP GPIO 轮询间隔(ms) |
| `BUTTON_DRIVER_USE_LP_GPIO` | n | LP GPIO 作按键 |
| `MAX_BUTTON_NUM` | 4 | 最大按键数 |

### light（`components/light/Kconfig`）

| 选项 | 默认 | 说明 |
|---|---|---|
| `USE_LIGHT_DEVICE_TYPE_WS2812` | y | 选 WS2812 |
| `USE_LIGHT_DEVICE_TYPE_LED` | n | 选 LED(PWM) |

### occupancy_sensor_ld2420（`components/occupancy_sensor_ld2420/Kconfig`）

存在 Kconfig（具体项以仓库为准；引用时核对 `components/occupancy_sensor_ld2420/Kconfig`）。

### display_ssd1306（`components/display_ssd1306/Kconfig`）

存在 Kconfig（具体项以仓库为准；引用时核对 `components/display_ssd1306/Kconfig`）。

## 4. `sdkconfig.defaults`（示例）

`products/socket/sdkconfig.defaults`：

```text
# Button
CONFIG_BUTTON_DRIVER_USE_HP_GPIO=y
```

各产品按需追加 `CONFIG_*` 默认值。

## 5. CMake 依赖（`main/CMakeLists.txt` REQUIRES）

| 组件名 | REQUIRES 关键字 |
|---|---|
| 核心 | `low_code` `system` |
| 按键 | `button` |
| 继电器 | `relay` |
| 灯光 | `light` |
| 温度传感器 | `temperature_sensor_sht30` |
| 占用传感器 | `occupancy_sensor_ld2420` |
| OLED 显示 | `display_ssd1306` |

示例（`products/temperature_sensor/main/CMakeLists.txt`）：

```cmake
idf_component_register(SRC_DIRS .
                        INCLUDE_DIRS .
                        REQUIRES low_code system button light temperature_sensor_sht30)
```

## 6. 默认引脚（取自各 product 源码）

| 用途 | 产品 | 引脚 |
|---|---|---|
| Button | socket / temperature_sensor / occupancy_sensor / temperature_sensor_with_display | GPIO9 |
| Relay | socket | GPIO2 |
| WS2812 指示 | socket / temperature_sensor / occupancy_sensor | GPIO8 |
| PWM 冷/暖白 | light_cw_pwm | cold=GPIO4, warm=GPIO6 |
| I2C SCL/SDA | temperature_sensor(_with_display) | GPIO1 / GPIO2（`I2C_NUM_0`） |
| LD2420 UART TX/RX | occupancy_sensor | TX=GPIO3, RX=GPIO2（`UART_NUM_1`） |
| SSD1306 I2C 地址 | temperature_sensor_with_display | 0x3C（`SSD1306_I2C_ADDRESS`） |

## 7. 烧录地址（见 `docs/getting_started_terminal.md`）

| 内容 | 烧录地址 |
|---|---|
| HP 预编译镜像 | 由 `pre_built_binaries/flash_args` 指定 |
| `esp_secure_cert.bin` | `0xD000` |
| `fctry.bin`（含数据模型/工厂分区） | `0x1F2000` |
| LP 应用固件 `build/<product>.bin` | `0x20C000` |
