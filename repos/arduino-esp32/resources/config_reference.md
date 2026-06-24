# 配置 / 分区表 / Kconfig 参考

> 内容来源：`docs/en/tutorials/partition_table.rst`、`docs/en/migration_guides/2.x_to_3.0.rst`、`Kconfig.projbuild`、`cores/esp32/main.cpp`、`idf_component.yml`、`platform.txt`。

## 1. 编译宏（菜单 / `-D`）

来自 `Kconfig.projbuild` 与 core 源码常用的 Arduino 构建宏：

| 宏 | 说明 |
|---|---|
| `ARDUINO_USB_CDC_ON_BOOT` | 启动时 USB CDC 作 Serial（影响默认 `Serial` 走向） |
| `ARDUINO_USB_MSC_ON_BOOT` | 启动时启用 USB MSC |
| `ARDUINO_USB_DFU_ON_BOOT` | 启动时进入 USB DFU |
| `ARDUINO_USB_MODE` | USB 模式（硬件 CDC vs USB-OTG） |
| `ARDUINO_LOOP_STACK_SIZE` | 覆盖 loopTask 栈大小（默认 8192） |
| `CONFIG_ARDUINO_LOOP_STACK_SIZE` | Kconfig 对应项 |
| `CONFIG_AUTOSTART_ARDUINO` | 自动启动 Arduino loopTask |
| `CONFIG_FREERTOS_UNICORE` | 单核运行（影响 yield 行为） |
| `NO_GLOBAL_INSTANCES` | 不创建全局实例（Serial 等） |
| `NO_GLOBAL_SERIAL` | 不创建全局 Serial |
| `ARDUHAL_LOG_LEVEL` | 日志级别（0=NONE ... 5=VERBOSE），影响 `log_i/log_e` |

> 日志宏：`log_v/log_d/log_i/log_w/log_e`（级别受 `ARDUHAL_LOG_LEVEL` 控制）。

## 2. 分区表（`docs/en/tutorials/partition_table.rst`）

CSV 格式：`# Name, Type, SubType, Offset, Size, Flags`
- Type: `app` / `data`
- SubType(`data`): `nvs` / `ota` / `coredump` / `nvs_keys` / `fat` / `spiffs`
- SubType(`app`): `factory` / `ota_0`..`ota_15` / `test`
- Offset: app 分区须 64 KB 对齐，首项数据分区固定 `0x9000`，首个 app `0x10000`
- Flags: `encrypted`（启用 flash 加密）

**自定义分区表**：在 sketch 同目录创建 `partitions.csv`，构建系统自动拾取。

常用尺寸建议：`nvs` ≥12 KB（推荐 12~64 KB）；`otadata` 固定 8 KB（0x2000）；OTA 至少 `ota_0`+`ota_1` 两个；`coredump` 64 KB；`nvs_keys` 4 KB。

### 典型 4 MB（带 OTA）
```csv
# Name,   Type, SubType, Offset, Size, Flags
nvs,      data, nvs,     36K,    20K,
otadata,  data, ota,     56K,    8K,
app0,     app,  ota_0,   64K,    1900K,
app1,     app,  ota_1,   ,       1900K,
```

### 典型 8 MB（OTA + 存储，来自 `default_8MB.csv`）
```csv
# Name,   Type, SubType, Offset,  Size,    Flags
nvs,      data, nvs,     0x9000,  0x5000,
otadata,  data, ota,     0xe000,  0x2000,
app0,     app,  ota_0,   0x10000, 0x330000,
app1,     app,  ota_1,   0x340000,0x330000,
spiffs,   data, spiffs,  0x670000,0x190000,
```

### 无 OTA（factory 单 app）
```csv
# Name,   Type, SubType, Offset, Size, Flags
nvs,      data, nvs,     36K,    20K,
factory,  app,  factory, 64K,    4000K,
```

## 3. 版本与依赖（`idf_component.yml`）

- 描述：`Arduino core for ESP32, ESP32-C, ESP32-H, ESP32-P, ESP32-S series of SoCs`
- 许可：`LGPL-2.1`
- targets: `esp32 esp32c2 esp32c3 esp32c5 esp32c6 esp32c61 esp32h2 esp32p4 esp32s2 esp32s3`
- ESP-IDF 依赖：`>=5.3,<6.2`（core 3.x）
- 平台版本（`platform.txt`）：`version=3.3.10`

C2/C61：不在 boards manager 预编译库覆盖范围，必须以 ESP-IDF 组件方式或 lib_builder 重建。

## 4. 2.x → 3.0 迁移要点（`docs/en/migration_guides/2.x_to_3.0.rst`）

已删除 API（编译失败）：
- `ledcSetup` / `ledcAttachPin`（→ `ledcAttach`）
- `ledcAttachPin` 改名 `ledcDetach`，channel 参数改为 pin
- `analogSetClockDiv` / `adcAttachPin` / `analogSetVRefPin`
- `hallRead`（Hall sensor 不再支持）
- RMT: `_rmtDumpStatus` / `rmtSetTick` / `rmtWriteBlocking` / `rmtEnd` / `rmtBeginReceive` / `rmtReadData`

新增 API：
- LEDC: `ledcAttach` / `ledcOutputInvert` / `ledcFade` / `ledcFadeWithInterrupt(Arg)`
- RMT: `rmtSetEOT` / `rmtWriteAsync` / `rmtTransmitCompleted` / `rmtSetRxMinThreshold`
- I2S 完全重构（见 `docs/en/api/i2s`）

行为变更：
- `WiFiClient/WiFiUDP` 的 `flush()` 不再清接收缓冲，改用 `clear()`
- `WiFiServer::available()` 弃用，用 `accept()`
- BLE 返回类型由 `std::string` 改为 Arduino `String`；UUID 改为 `BLEUUID`；`BLEScan::start/getResults` 返回 `BLEScanResults*`

## 5. 工具链路径（`platform.txt`）

| 项 | 路径 |
|---|---|
| xtensa gcc | `tools/xtensa-esp-elf/bin/` |
| riscv32 gcc | `tools/riscv32-esp-elf/bin/` |
| 静态库 | `tools/esp32-arduino-libs/<chip_variant>/` |
| esptool | `tools/esptool` |
| OTA 工具 | `tools/espota.py`（`tools/espota.exe`） |
| 分区表工具 | `tools/gen_esp32part.py` |
| 预置分区表 | `tools/partitions/*.csv` |
