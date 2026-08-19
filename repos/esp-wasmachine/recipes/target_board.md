# 选择目标板与 sdkconfig.defaults

> **适用摘要**: 为 ESP32 / ESP32-S3（含 BOX）/ ESP32-C6 / ESP32-P4 选择正确的 `set-target` 与板级 `sdkconfig.defaults` 覆盖。

> Evidence: `repos/esp-wasmachine/resources/`, source/examples in `repos/esp-wasmachine/`, and this recipe path `repos/esp-wasmachine/recipes/target_board.md`.
> Validation: draft metadata added from repository routing; verify APIs, Kconfig symbols, and component dependencies against the selected version.

## 触发意图

- "S3-BOX 怎么编译"
- "ESP32-P4 用哪个 sdkconfig"
- "4MB 板用哪个分区表"
- "target_board / sdkconfig.defaults 区别"

## 前置条件

| 条件 | 要求 |
|---|---|
| 硬件 | 确认板型号与 flash 容量 |
| 文件 | `examples/wasmachine/sdkconfig.defaults*` 与 `partitions.*.csv` 在工程内 |

## 分步说明

### 目标与板级映射

| 目标板 | `set-target` | 默认文件叠加 | 分区表 |
|---|---|---|---|
| ESP32-DevKitC (4MB) | `esp32` | `sdkconfig.defaults` + `sdkconfig.defaults.esp32` | `partitions.4mb.single_app.csv` |
| ESP32-S3-DevKitC | `esp32s3` | `sdkconfig.defaults` + `sdkconfig.defaults.esp32s3` | `partitions.8mb.csv`（典型） |
| ESP32-S3-BOX / S3-BOX-Lite | `esp32s3` | `sdkconfig.defaults` + `sdkconfig.esp-box` | 见 `sdkconfig.esp-box` |
| ESP32-C6-DevKitC (4MB) | `esp32c6` | `sdkconfig.defaults` + `sdkconfig.defaults.esp32c6` | `partitions.4mb.single_app.csv` |
| ESP32-P4-Function-EV-Board | `esp32p4` | `sdkconfig.defaults` + `sdkconfig.esp32_p4_function_ev_board` | 见该文件 |

### ESP32-DevKitC / C6（4MB）

```bash
idf.py set-target esp32        # 或 esp32c6
idf.py build
```

`README.md` §4.1 注：4MB 板使用 `partitions.4mb.singleapp.csv`（单 OTA 槽）。

### ESP32-S3-BOX / S3-BOX-Lite

```bash
idf.py -DSDKCONFIG_DEFAULTS="sdkconfig.defaults;sdkconfig.esp-box" set-target esp32s3
idf.py build
```

`examples/wasmachine/sdkconfig.defaults.esp32s3` 设定了：

```ini
CONFIG_ESPTOOLPY_FLASHMODE_QIO=y
CONFIG_ESPTOOLPY_FLASHSIZE_8MB=y
CONFIG_ESP_RMAKER_FACTORY_PARTITION_NAME="nvs"
CONFIG_PARTITION_TABLE_CUSTOM=y
CONFIG_PARTITION_TABLE_CUSTOM_FILENAME="partitions.8mb.csv"
CONFIG_SPIRAM=y
CONFIG_SPIRAM_MALLOC_ALWAYSINTERNAL=0
CONFIG_SPIRAM_MODE_OCT=y
CONFIG_SPIRAM_TRY_ALLOCATE_WIFI_LWIP=y
CONFIG_MBEDTLS_EXTERNAL_MEM_ALLOC=y
```

### ESP32-P4-Function-EV-Board

```bash
idf.py -DSDKCONFIG_DEFAULTS="sdkconfig.defaults;sdkconfig.esp32_p4_function_ev_board" set-target esp32p4
idf.py build
```

P4 注意点（来自 `examples/wasmachine/main/idf_component.yml`）：
- P4 不支持 RainMaker（`wasmachine_ext_wasm_native_rainmaker` 的 `rules` 把 P4 排除）。
- P4 用 `esp_wifi_remote`（非 `esp_wifi`），依赖 `esp_wifi_remote == 0.14.0`。
- LVGL/BSP 通过 `esp32_p4_function_ev_board == 5.0.*` 引入。

## 常见错误

| 错误 | 原因 | 解决方法 |
|---|---|---|
| BOX 屏幕不显示 | 未叠加 `sdkconfig.esp-box` | 用 `-DSDKCONFIG_DEFAULTS=...` 重新 `set-target` |
| P4 链接 RainMaker 失败 | P4 不支持 RainMaker | 删除 `CONFIG_WASMACHINE_WASM_EXT_NATIVE_RMAKER=y`，依赖 yml 已排除 |
| P4 编译报 `esp_wifi` 缺失 | P4 用 `esp_wifi_remote` | 确认 `idf_component.yml` 中 P4 规则拉了 `esp_wifi_remote` |
| 4MB 板烧录失败/重叠 | 用了多 OTA 分区表 | 改用 `partitions.4mb.single_app.csv` |

## 参考

- `README.md` §2、§4.1 — 支持的开发板与编译命令
- `examples/wasmachine/sdkconfig.defaults{.esp32,.esp32c6,.esp32s3,.esp32p4,.esp-box,.esp32_p4_function_ev_board}`
- `examples/wasmachine/main/idf_component.yml` — 目标相关的托管依赖规则
- `resources/config_reference.md` — 分区表对照
