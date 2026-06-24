# ESP-Insights Consolidated Pitfalls

> 与 `SKILL.md` 的 “Critical Pitfalls” 互补，按主题汇总，便于快速定位。所有结论均来自仓库 README/FEATURES/CHANGELOG/头文件注释。

## 1. 初始化与传输

- **HTTPS 必须 `auth_key`，MQTT 不用**：`esp_insights_config_t.auth_key` 仅 HTTPS 有效；MQTT 靠 RainMaker Claiming 证书。来源：`esp_insights.h` 字段注释。
- **Auth Key 文件要链接进固件**：`main/CMakeLists.txt` 须含 `target_add_binary_data(${COMPONENT_TARGET} "insights_auth_key.txt" TEXT)`，源码用 `asm("_binary_insights_auth_key_txt_start")` 取址。来源：`examples/*/main/CMakeLists.txt`。
- **自定义 transport 用 enable 不用 init**：`esp_insights_init()` 会内部注册默认 transport；覆盖时用 `esp_insights_transport_register()` + `esp_insights_enable()`。来源：`esp_insights.h` 函数注释。
- **关闭顺序**：`esp_insights_disable()` 不注销 transport；`esp_insights_transport_unregister()` 才移除回调。来源：头文件 `@note`。

## 2. 日志采集

- **Default log verbosity = No output 会吞掉 error/warn**：`ESP_LOGE/W` 在预处理期被移除，诊断钩子也消失。改 >= Warning，或把 Console channel 设 None（日志进诊断、串口静默）。`ESP_DIAG_EVENT` 不受影响。来源：FEATURES.md “Insights with Logging disabled”。
- **`ESP_DIAG_EVENT` 是事件通道，不是 INFO**：走 `ESP_DIAG_LOG_TYPE_EVENT`，`log_type` 必须含该位。来源：`esp_diagnostics.h` 宏展开。
- **Wi-Fi 日志默认被丢**：`CONFIG_DIAG_LOG_DROP_WIFI_LOGS=y`（默认），每条 Wi-Fi 日志会衍生 3 条诊断日志故默认丢弃；要记录则置 n。来源：`components/esp_diagnostics/Kconfig`。
- **外部包装日志共存**：另一组件也要 `--wrap` 日志时，启用 `CONFIG_DIAG_USE_EXTERNAL_LOG_WRAP` 并在 `__wrap_esp_log_writev` 内调 `esp_diag_log_writev()`。来源：`esp_diagnostics.h` 注释。

## 3. metadata 1.0 vs 2.0（破坏性）

- **1.0（`META_VERSION_10=y`）API 无 `tag` 形参**：`esp_diag_metrics_add_uint(key, val)`、`esp_diag_variable_add_uint(key, val)`。
- **2.0（`META_VERSION_10=n`）API 带 `tag`**：`esp_diag_metrics_report_uint(tag, key, val)`、`esp_diag_variable_report_uint(tag, key, val)`。
- **迁移到 2.0 是破坏性变更**：旧 metric/variable 数据在新面板不再显示。examples 默认开 2.0（`sdkconfig.defaults` 里 `CONFIG_ESP_INSIGHTS_META_VERSION_10=n`）。来源：CHANGELOG 2024-02。

## 4. Core dump

- **三件套缺一不可**：① coredump-to-flash（`ESP*_ENABLE_COREDUMP_TO_FLASH`）② ELF 格式（`*_COREDUMP_DATA_FORMAT_ELF`）③ `coredump` 分区。`ESP_INSIGHTS_COREDUMP_ENABLE` 依赖 ①② 同时成立。来源：`components/esp_insights/Kconfig`。
- **摘要上报后被擦除**：上报成功后 core dump 从 flash 分区擦除；本地调试前需禁用上报。来源：README Behind the Scenes、FEATURES.md。
- **RISC-V 无 on-device backtrace**：`esp_diag_task_snapshot_get` 返回结构里 `bt_info` 仅 Xtensa 存在（`#ifndef CONFIG_IDF_TARGET_ARCH_RISCV`）。来源：`esp_diagnostics.h`。

## 5. 分区表

- **自定义分区表必须启用**：`CONFIG_PARTITION_TABLE_CUSTOM=y` + `CONFIG_PARTITION_TABLE_CUSTOM_FILENAME="partitions.csv"`。
- **coredump 分区**：`coredump, data, coredump, , 64K`。
- **MQTT claiming 需 fctry**：`fctry, data, nvs, 0x340000, 0x6000,`。来源：`examples/minimal_diagnostics/partitions.csv`。
- **换分区表后 `erase_flash`**：否则旧分区表残留导致异常。

## 6. 数据存储

- **RTC 默认且跨软重启保留**：硬重启/断电不保留（当前 FLASH 存储未支持）。
- **容量估算**：critical 每条 ~121 字节，non-critical 每条 ~49 字节。例：2048B critical ≈ 16 条。来源：README “RTC data store”。
- **满载会丢**：监听 `ESP_DIAG_DATA_STORE_EVENT_*_LOW_MEM`，触发 `esp_insights_send_data()` 或调大 `CONFIG_RTC_STORE_CRITICAL_DATA_SIZE`。
- **ESP32 RTC 上限**：旧 IDF 限 4K；IDF v4.3+ 可达 8K（Kconfig max 7168）。

## 7. 上报节奏

- **动态间隔**：发数据→间隔翻倍，不发→减半，夹在 [MIN=60, MAX=240]（默认）。来源：`esp_insights/Kconfig`。
- **手动 flush**：`esp_insights_send_data()` 异步，可能需时间。
- **`reporting_disable()` 不彻底停**：meta/boot 消息仍发；彻底停用 `esp_insights_disable()`。来源：头文件 `@note`。

## 8. 仪表盘 / 云端

- **必须上传固件包**：`build/<project>-<ver>.zip`（含 bin/elf/map）。不上传 → “Firmware Image missing”，rodata 字符串无法交叉引用，backtrace 只见裸地址。来源：examples/minimal_diagnostics/README.md。
- **包与板上二进制必须一致**：`idf.py build` 即使代码没变也会重新生成 zip，务必同步上传。
- **Node ID**：未设 `config.node_id` 则用 MAC。启动日志 `Insights enabled for Node ID ...`。

## 9. command-response

- **仅 RainMaker MQTT**：`ESP_INSIGHTS_CMD_RESP_ENABLED` 依赖 `ENABLED && TRANSPORT_MQTT`。HTTPS 不可用。
- **用 `enable()` 启动时需手动 enable**：`esp_insights_cmd_resp_enable()`（`init()` 会自动做）。来源：头文件注释。

## 10. 兼容性

- **ESP-IDF 4.x 不在 main 分支**：用 `idf_4_x_compat` 分支或 registry 的 esp_insights 1.2.x。
- **v5.0 用 1.2.x**：来自 examples/README.md。
- **v6.0 已兼容**：CHANGELOG 2026-04（coredump 头文件守卫、CMake target 顺序、WIFI_BW 重命名兼容）。
