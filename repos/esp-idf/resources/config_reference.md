# Kconfig / sdkconfig 配置速查

> 符号取自 ESP-IDF 仓库 `components/*/Kconfig*` 与 `Kconfig`（根）。改配置通过 `idf.py menuconfig`，或写入 `sdkconfig.defaults` 让构建自动应用；勿手改 `sdkconfig`。

## 工程与目标

| 符号 | 说明 |
|---|---|
| `CONFIG_IDF_TARGET` | 目标芯片名（`esp32`/`esp32s3`/`esp32c3`/...），由 `idf.py set-target` 设定 |
| `CONFIG_IDF_TARGET_ARCH` | `riscv` / `xtensa` |
| `CONFIG_IDF_TARGET_ESP32` / `_ESP32S3` / `_ESP32C3` ... | 各目标布尔开关 |

> 切目标必须 `idf.py set-target <chip>`，不要手改这些符号。

## 分区表（`components/partition_table/Kconfig.projbuild`）

| 符号 | 说明 |
|---|---|
| `CONFIG_PARTITION_TABLE_TYPE` | 单选：默认单 app / Factory+两 OTA / 自定义 CSV |
| `CONFIG_PARTITION_TABLE_FILENAME` | 自定义 CSV 文件名（如 `partitions.csv`） |
| `CONFIG_PARTITION_TABLE_OFFSET` | 分区表偏移（默认 `0x8000`） |
| `CONFIG_PARTITION_TABLE_MD5` | 校验分区表 MD5 |

CSV 字段：`# Name, Type, SubType, Offset, Size, Flags`，`Type` = `app`/`data`/`bootloader`/`partition`；OTA 需 `factory` + `ota_0` + `ota_1` + `otadata`(`data,ota`)。

## 日志（`components/log/Kconfig.level`）

| 符号 | 说明 |
|---|---|
| `CONFIG_LOG_DEFAULT_LEVEL` | 编译期默认级别：`NONE`/`ERROR`/`WARN`/`INFO`/`DEBUG`/`VERBOSE` |
| `CONFIG_LOG_MAXIMUM_LEVEL` | 允许的运行时最高级别（高于 DEFAULT 时需 `esp_log_level_set` 提级） |
| `CONFIG_LOG_COLORS` | 终端彩色 |
| `CONFIG_LOG_TIMESTAMP_SOURCE` | 时间戳来源（CPU tick / RTC） |

> 运行时 `esp_log_level_set(tag, ESP_LOG_DEBUG)` 可对单个 tag 提级。

## FreeRTOS（`components/freertos/Kconfig`）

| 符号 | 说明 |
|---|---|
| `CONFIG_FREERTOS_UNICORE` | 单核运行（esp32 双核时可强制单核） |
| `CONFIG_FREERTOS_HZ` | tick 频率（默认 100，常用 1000） |
| `CONFIG_FREERTOS_USE_TICKLESS_IDLE` | tickless 空闲（配合低功耗） |
| `CONFIG_FREERTOS_THREAD_LOCAL_STORAGE_POINTERS` | TLS 指针数量 |

## Wi-Fi / 蓝牙（`menuconfig` 中 Component config）

| 符号 | 说明 |
|---|---|
| `CONFIG_ESP_WIFI_STATIC_RX_BUFFER_NUM` / `_DYNAMIC` | Wi-Fi RX 缓冲数量 |
| `CONFIG_ESP_WIFI_TX_BUFFER` | TX 缓冲类型/数量 |
| `CONFIG_ESP_WIFI_NVS_ENABLED` | 是否用 NVS 存 Wi-Fi 配置 |
| `CONFIG_BT_ENABLED` | 启用蓝牙（总开关） |
| `CONFIG_BT_BLUEDROID_ENABLED` | Bluedroid 双模 Host（支持 Classic BT；用 `esp_ble_*`/`esp_spp_*`/`esp_a2d_*` API） |
| `CONFIG_BT_NIMBLE_ENABLED` | NimBLE Host（仅 BLE，体积小；用 `ble_*` API，与 Bluedroid 互斥） |
| `CONFIG_BT_CLASSIC_ENABLED` | 经典蓝牙（**仅 esp32**，依赖 `BT_BLUEDROID_ENABLED` + `SOC_BT_CLASSIC_SUPPORTED`） |
| `CONFIG_BT_SPP_ENABLED` | SPP（经典蓝牙串口，依赖 Classic BT） |
| `CONFIG_BT_A2DP_ENABLE` | A2DP 音频（依赖 Classic BT）；`CONFIG_BT_A2DP_CODEC_AAC_ENABLED` 启 AAC |
| `CONFIG_BT_HFP_ENABLED` | HFP 免提（依赖 Classic BT） |
| `CONFIG_BLE_MESH` | ESP-BLE-MESH（蓝牙 Mesh，可配合 Bluedroid 或 NimBLE） |
| `CONFIG_MESH_TOPOLOGY` | ESP-WIFI-MESH 拓扑（`TREE` / `CHAIN`） |
| `CONFIG_MESH_MAX_LAYER` | Wi-Fi Mesh 最大层级 |
| `CONFIG_MESH_ROUTE_TABLE_SIZE` | Wi-Fi Mesh 路由表大小（按节点数调） |
| `CONFIG_ETH_ENABLED` | 以太网 MAC 驱动（esp32/p4 默认启用） |
| `CONFIG_BTDM_CTRL_HCI_MODE_VHCI` / `..._UART_H4` | HCI 传输模式 |

> Wi-Fi 与蓝牙共存/优先级受 `CONFIG_ESP_COEX_SW_COEXIST_ENABLE` 等控制。Classic BT（SPP/A2DP/HFP）仅 esp32 双模支持；其余 SoC（c3/s3/c6/h2 等）仅 BLE。Bluedroid 与 NimBLE 二选一（`choice BT_HOST`）。

## Flash / 分区存储

| 符号 | 说明 |
|---|---|
| `CONFIG_SPIFLASH_ENABLE_ENCRYPTED_READ_WRITE` | 启用加密读写 |
| `CONFIG_FATFS_LFN_ENABLED` | FATFS 长文件名 |
| `CONFIG_WL_SECTOR_SIZE` | wear_levelling 扇区（512/4096） |
| `CONFIG_SPIFFS_*` | SPIFFS 参数 |

## HTTP / TLS

| 符号 | 说明 |
|---|---|
| `CONFIG_ESP_HTTP_CLIENT_ENABLE_HTTPS` | HTTPS 客户端 |
| `CONFIG_ESP_HTTPS_OTA_DECRYPT_CB` | OTA 解密回调（加密镜像） |
| `CONFIG_MBEDTLS_CERTIFICATE_BUNDLE` | 内置根证书包 |

## 电源管理

| 符号 | 说明 |
|---|---|
| `CONFIG_PM_ENABLE` | 电源管理框架 |
| `CONFIG_FREERTOS_USE_TICKLESS_IDLE` | tickless（节电） |
| `CONFIG_ESP_DEFAULT_CPU_FREQ_MHZ` | 默认主频 |
| `CONFIG_PM_SLP_DEFAULT_PARAMS_OPTIMIZE` | 默认睡眠参数优化 |

## 构建/优化（项目级常见）

| 符号 | 说明 |
|---|---|
| `CONFIG_COMPILER_OPTIMIZATION_*` | `NONE`/`SIZE`/`PERFORMANCE`/`ASSERTIONS` |
| `CONFIG_COMPILER_WARN_WRITE_STRINGS` | 字符串常量写警告 |
| `CONFIG_BOOTLOADER_LOG_LEVEL` | bootloader 日志级别 |
| `CONFIG_ESPTOOLPY_FLASHSIZE` | Flash 大小（`2MB`/`4MB`/`8MB`/...） |
| `CONFIG_ESPTOOLPY_FLASHMODE` | Flash 模式（`dio`/`qio`/`dout`/`qout`） |
| `CONFIG_ESPTOOLPY_FLASHFREQ` | Flash 频率（`40m`/`80m`） |

## sdkconfig.defaults 示例

```
# 默认应用 OTA 分区表
CONFIG_PARTITION_TABLE_TYPE=y
CONFIG_PARTITION_TABLE_CUSTOM=y
CONFIG_PARTITION_TABLE_CUSTOM_FILENAME="partitions.csv"

# 日志级别
CONFIG_LOG_DEFAULT_LEVEL_INFO=y

# Flash
CONFIG_ESPTOOLPY_FLASHSIZE_4MB=y
CONFIG_ESPTOOLPY_FLASHMODE_DIO=y

# 蓝牙
CONFIG_BT_ENABLED=y
CONFIG_BT_NIMBLE_ENABLED=y
```

> `sdkconfig.defaults` 仅在首次配置（`set-target`/`build`）时生效；改后用 `idf.py reconfigure` 或 `rm sdkconfig && idf.py build`。
