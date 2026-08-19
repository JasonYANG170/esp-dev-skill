# RAM 与 Flash 优化

> **适用摘要**: Matter 固件体积大、内存吃紧（尤其 esp32c2 / esp32h2）时，按收益从高到低逐项打开 Kconfig 优化项，并用 measured before/after 表预估节省量。所有数字取自 `docs/en/optimizations.rst`（基于 esp32c3 / esp32h2 + light 示例）。

> Version: selected repo/component version; align with the user project and dependency manifest.
> Validation: draft metadata added from repository routing; verify APIs, Kconfig symbols, and component dependencies against the selected version.

## 触发意图

- "Matter 固件太大 / RAM 不够"
- "esp32c2 / esp32h2 上跑 Matter 怎么省内存"
- "怎么关掉用不到的 cluster 省 flash"
- "newlib nano / LTO / IRAM to flash 怎么开"
- "BLE NimBLE 怎么省 DRAM"
- "spi_ram 怎么用（controller 示例）"

## 前置条件

| 条件 | 要求 |
|---|---|
| 参考工程 | `examples/light/`（基准测量对象）、`examples/controller/sdkconfig.defaults.ram_optimization`（SPIRAM 实战配置）、`examples/multiple_on_off_plugin_units/`（多 endpoint 内存） |
| 文档 | `docs/en/optimizations.rst`（所有 measured 表） |
| 工具 | `idf.py size-components` / `idf.py size-files` 看实测分布 |

## 分步说明

> 下表把各优化项的 **典型收益** 集中列出（来自 `optimizations.rst`，esp32h2 / esp32c3 上 light 示例）。"D/IRAM 节省"指 Used D/IRAM 减少；"Heap 增"指 On Bootup Free Heap 增加。叠加时收益非严格线性，建议逐项测量。

| 优化项 | 主要 Kconfig | D/IRAM 节省 | Flash 节省 | Heap 增 |
|---|---|---|---|---|
| 关 chip-shell | `CONFIG_ENABLE_CHIP_SHELL=n` | ~0.8–2K | ~55–67K | ~10K |
| 调小 dynamic endpoint | `CONFIG_ESP_MATTER_MAX_DYNAMIC_ENDPOINT_COUNT` / `MAX_DEVICE_TYPE_COUNT` | ~6.6K | ~0.1–0.4K | ~6K |
| newlib nano | `CONFIG_NEWLIB_NANO_FORMAT=y` | 0 | ~47K | ~2K |
| BLE NimBLE 调优 | 见步骤 4 | ~1.7–2.2K | ~23–24K | ~10K（仅 bootup） |
| 事件日志 buffer | `CONFIG_EVENT_LOGGING_*_BUFFER_SIZE` | ~5.4K | ~0 | ~6.5K |
| BLE 进 flash | `CONFIG_BT_CTRL_RUN_IN_FLASH_ONLY=y` | ~19–20K | -13～-142K（变大） | ~18–23K |
| FreeRTOS 进 flash | `CONFIG_FREERTOS_PLACE_FUNCTIONS_INTO_FLASH=y` | ~9–10K | -9～-10K | ~9K |
| Ringbuf 进 flash | `CONFIG_RINGBUF_PLACE_FUNCTIONS_INTO_FLASH=y` | ~4.7K | -5K | ~4K |
| SPI flash ROM impl | `CONFIG_SPI_FLASH_ROM_IMPL=y` | ~9.5–12.7K | ~3K | ~8–13K |
| Heap 进 flash | `CONFIG_HEAP_PLACE_FUNCTION_INTO_FLASH=y` | 0–7K | -7K | -0.4～6.5K |
| 任务栈调小 | `CONFIG_CHIP_TASK_STACK_SIZE=6144` 等 | 0 | 0 | ~3.3–4K |
| 关未用 cluster | `CONFIG_SUPPORT_*_CLUSTER=n` | ~3.7K | ~37K | ~4K |
| LTO | `CONFIG_COMPILER_OPTIMIZATION_ASSERTIONS_ENABLE_SILENT` 等（见步骤 9） | — | ~90K | -1.7K stack |

### 1. 关闭 chip-shell（生产）

shell 只在调试用，生产关掉省 55K+ flash + 10K heap：

```text
CONFIG_ENABLE_CHIP_SHELL=n
```

### 2. 调小 dynamic endpoint / device type 计数

默认都是 16，但 light 只需 2 个 endpoint。每减少一个约省 550 字节 DRAM：

```text
CONFIG_ESP_MATTER_MAX_DYNAMIC_ENDPOINT_COUNT=2
CONFIG_ESP_MATTER_MAX_DEVICE_TYPE_COUNT=2
```

### 3. newlib nano formatting

省 ~47K flash，并降低调用 printf 类函数任务的栈高水位：

```text
CONFIG_NEWLIB_NANO_FORMAT=y
```

### 4. BLE NimBLE 调优（Matter 设备一般只需 1 条 BLE 连接）

Matter 设备只把 BLE 用于 commissioning，且通常只接受 1 条连接，可关 central / observer 角色：

```text
CONFIG_NIMBLE_MAX_CONNECTIONS=1
CONFIG_BTDM_CTRL_BLE_MAX_CONN=1
CONFIG_BT_NIMBLE_MAX_CONNECTIONS=1
CONFIG_BT_NIMBLE_ROLE_CENTRAL=n
CONFIG_BT_NIMBLE_ROLE_OBSERVER=n
CONFIG_BT_NIMBLE_MAX_BONDS=2
CONFIG_BT_NIMBLE_MAX_CCCDS=2
CONFIG_BT_NIMBLE_SECURITY_ENABLE=n
CONFIG_BT_NIMBLE_50_FEATURE_SUPPORT=n
CONFIG_BT_NIMBLE_WHITELIST_SIZE=1
CONFIG_BT_NIMBLE_GATT_MAX_PROCES=1
CONFIG_BT_NIMBLE_MSYS_1_BLOCK_COUNT=10
CONFIG_BT_NIMBLE_MSYS_1_BLOCK_SIZE=100
CONFIG_BT_NIMBLE_MSYS_2_BLOCK_COUNT=4
CONFIG_BT_NIMBLE_MSYS_2_BLOCK_SIZE=320
CONFIG_BT_NIMBLE_ACL_BUF_COUNT=5
CONFIG_BT_NIMBLE_HCI_EVT_HI_BUF_COUNT=5
CONFIG_BT_NIMBLE_HCI_EVT_LO_BUF_COUNT=3
CONFIG_BT_NIMBLE_ENABLE_CONN_REATTEMPT=n
```

> 由于 `CONFIG_USE_BLE_ONLY_FOR_COMMISSIONING=y` 时 BLE 在 commissioning 后被释放，这些项主要提升 bootup heap，对 post-commissioning heap 贡献为 0 或略负。

### 5. 缩小事件日志 buffer

```text
CONFIG_EVENT_LOGGING_CRIT_BUFFER_SIZE=256
CONFIG_EVENT_LOGGING_INFO_BUFFER_SIZE=256
CONFIG_EVENT_LOGGING_DEBUG_BUFFER_SIZE=256
CONFIG_MAX_EVENT_QUEUE_SIZE=20
CONFIG_ESP_SYSTEM_EVENT_QUEUE_SIZE=16
CONFIG_ESP_SYSTEM_EVENT_TASK_STACK_SIZE=2048
```

> 缩得太小会丢事件，关键日志场景慎用。

### 6. 把代码从 IRAM 挪到 flash（省 IRAM，增 flash、降速）

> 这些项可能影响性能，生产前要充分测试。

```text
# BLE controller 代码进 flash（建议同时开 SPI_FLASH_AUTO_SUSPEND 减缓性能影响）
CONFIG_BT_CTRL_RUN_IN_FLASH_ONLY=y

# FreeRTOS 非 ISR 函数进 flash（最多省 8K IRAM）
CONFIG_FREERTOS_PLACE_FUNCTIONS_INTO_FLASH=y

# 非 ISR ringbuf 函数进 flash
CONFIG_RINGBUF_PLACE_FUNCTIONS_INTO_FLASH=y

# 用 ROM 里的 SPI flash 驱动
CONFIG_SPI_FLASH_ROM_IMPL=y
CONFIG_SPI_MASTER_ISR_IN_IRAM=n
CONFIG_SPI_SLAVE_ISR_IN_IRAM=n

# heap 组件进 flash（仅当无 ISR 在 cache 关闭时调 heap_caps.h 才安全）
CONFIG_HEAP_PLACE_FUNCTION_INTO_FLASH=y
```

### 7. 调小任务栈

```text
CONFIG_ESP_MAIN_TASK_STACK_SIZE=3072
CONFIG_ESP_TIMER_TASK_STACK_SIZE=2048
CONFIG_CHIP_TASK_STACK_SIZE=6144
```

### 8. 关闭未使用的 Matter cluster

菜单：`Component config → ESP Matter → Select Supported Matter Clusters`。默认已经把未用 cluster 关掉；若开了多余 cluster，按需关。完整列表见 `optimizations.rst`（约 80 个 `CONFIG_SUPPORT_*_CLUSTER`），常用项举例：

```text
CONFIG_SUPPORT_DOOR_LOCK_CLUSTER=n
CONFIG_SUPPORT_THERMOSTAT_CLUSTER=n
CONFIG_SUPPORT_WINDOW_COVERING_CLUSTER=n
CONFIG_SUPPORT_FAN_CONTROL_CLUSTER=n
CONFIG_SUPPORT_OCCUPANCY_SENSING_CLUSTER=n
CONFIG_SUPPORT_MEDIA_PLAYBACK_CLUSTER=n
CONFIG_SUPPORT_APPLICATION_BASIC_CLUSTER=n
# ...（仅保留你 device type 用到的）
```

### 9. LTO（仅 esp32c2/c3/c5/c6/h2 等）

LTO 可省 ~90K flash，但任务栈会多 ~1700 字节，启用方式见 ESP-IoT-Solution 的 `cmake_utilities` gcc 文档：

```text
# 通过 idf 对应的 LTO 选项开启（具体 Kconfig 名见 ESP-IDF / cmake_utilities 文档）
```

### 10. SPIRAM：把 BSS 放到外部 RAM（controller 示例）

带 PSRAM 的模组（如 esp32s3）可把 controller 库的 BSS 移到 SPIRAM。来自 `examples/controller/sdkconfig.defaults.ram_optimization` + `main/linker.lf`：

```text
CONFIG_SPIRAM=y
CONFIG_SPIRAM_ALLOW_BSS_SEG_EXTERNAL_MEMORY=y
# 2MB PSRAM 用 QUAD，>2MB 用 OCT
# CONFIG_SPIRAM_MODE_QUAD=y   # 2MB
# CONFIG_SPIRAM_MODE_OCT=y    # >2MB
```

```bash
idf.py -D SDKCONFIG_DEFAULTS="sdkconfig.defaults;sdkconfig.defaults.ram_optimization" set-target esp32s3 build
```

若报 "PSRAM chip not found or not supported, or wrong PSRAM line mode"：检查模组是否真有 PSRAM、`CONFIG_SPIRAM_MODE_QUAD`（2MB）/ `CONFIG_SPIRAM_MODE_OCT`（>2MB）是否选对。`sdkconfig.defaults.otbr` 默认已经启用 RAM 优化。

## 常见错误

| 错误 | 原因 | 解决方法 |
|---|---|---|
| 开 `BT_CTRL_RUN_IN_FLASH_ONLY` 后 BLE 不稳 | 没开 flash suspend | 同时开 `SPI_FLASH_AUTO_SUSPEND` |
| `HEAP_PLACE_FUNCTION_INTO_FLASH` 后随机崩 | 有 ISR 在 cache 关闭时调了 heap | 确认无 IRAM ISR 调 `esp_heap_caps.h` 再启用 |
| 关 cluster 后编译失败 | 关掉了 device type 必需的 cluster | 重新打开该 device type 需要的 `CONFIG_SUPPORT_*_CLUSTER` |
| PSRAM 报错 | mode 选错 | 2MB→`SPIRAM_MODE_QUAD`，>2MB→`SPIRAM_MODE_OCT` |
| LTO 后栈溢出 | LTO 增加栈耗 ~1.7K | 调大任务栈或不开 LTO |
| 测量值与文档不符 | IDF / esp-matter 版本不同 | 以本机构建实测为准，文档数字仅参考 |
| post-commissioning heap 没涨 | BLE 优化只影响 bootup（BLE 已释放） | 正常现象；post-commissioning BLE 内存已回收 |

## 参考

- `D:/esp-skill/espressif-repos/esp-matter/docs/en/optimizations.rst` — 每项优化的 measured before/after 表（esp32h2 + esp32c3 / light）
- `D:/esp-skill/espressif-repos/esp-matter/examples/controller/sdkconfig.defaults.ram_optimization` + `examples/controller/main/linker.lf` — SPIRAM + BSS 外置实战
- `D:/esp-skill/espressif-repos/esp-matter/examples/controller/sdkconfig.defaults.otbr` — OTBR 默认启用 RAM 优化
- `D:/esp-skill/espressif-repos/esp-matter/examples/light/` — 基准测量工程
- `D:/esp-skill/espressif-repos/esp-matter/examples/multiple_on_off_plugin_units/` — 多 endpoint 内存参考
- ESP-IDF 性能指南：`ram-usage` / `size` / `speed`（链接见 `optimizations.rst` 末尾）
- ESP-IoT-Solution `cmake_utilities` gcc 文档 — LTO 启用与示例
