# Core Dump 捕获与上报

> **适用摘要**: 让设备在崩溃时把 core dump 写入 flash，并在下次启动时把“摘要”（PC、异常 cause/vaddr、通用寄存器、backtrace）上报到云端。涉及 Kconfig、分区表与 ELF 固件包上传。

> Version: selected repo/component version; align with the user project and dependency manifest.
> Evidence: `repos/esp-insights/resources/`, source/examples in `repos/esp-insights/`, and this recipe path `repos/esp-insights/recipes/core_dump.md`.
> Validation: draft metadata added from repository routing; verify APIs, Kconfig symbols, and component dependencies against the selected version.

## 触发意图

- "崩溃日志上报"
- "core dump 配置"
- "崩溃 backtrace 在仪表盘查看"
- "coredump 分区"
- "异常重启原因分析"

## 前置条件

| 条件 | 要求 |
|---|---|
| Kconfig | 启用 coredump-to-flash + ELF 格式（见下） |
| 分区表 | 含 `coredump` 分区 |
| 固件包 | 已上传 `build/<project>-<ver>.zip` 到仪表盘（用于 backtrace 行号交叉引用） |
| 参考 | `examples/minimal_diagnostics`（sdkconfig.defaults + partitions.csv） |

## 分步说明

### 1. Kconfig（来自 `examples/minimal_diagnostics/sdkconfig.defaults`）

```
CONFIG_ESP_INSIGHTS_ENABLED=y
CONFIG_ESP32_ENABLE_COREDUMP=y
CONFIG_ESP32_ENABLE_COREDUMP_TO_FLASH=y
CONFIG_ESP32_COREDUMP_DATA_FORMAT_ELF=y
CONFIG_ESP32_COREDUMP_CHECKSUM_CRC32=y
CONFIG_ESP32_CORE_DUMP_MAX_TASKS_NUM=64
CONFIG_ESP32_CORE_DUMP_STACK_SIZE=1024
```
> 新版 IDF 符号名为 `CONFIG_ESP_COREDUMP_ENABLE_TO_FLASH` / `CONFIG_ESP_COREDUMP_DATA_FORMAT_ELF`；`CONFIG_ESP_INSIGHTS_COREDUMP_ENABLE`（默认 y）依赖“to_flash **且** ELF”同时成立（见 `components/esp_insights/Kconfig`）。

### 2. 分区表加 coredump（来自 `examples/minimal_diagnostics/partitions.csv`）

```csv
# Name,   Type, SubType,  Offset,   Size,    Flags
nvs,      data, nvs,      0x9000,   24K,
phy_init, data, phy,      0xf000,   4K,
factory,  app,  factory,  0x10000,  0x1E0000,
coredump, data, coredump, 0x330000, 64K,
fctry,    data, nvs,      0x340000, 0x6000,
```
并设 `CONFIG_PARTITION_TABLE_CUSTOM=y`、`CONFIG_PARTITION_TABLE_CUSTOM_FILENAME="partitions.csv"`。

### 3. 触发崩溃以验证（来自 `examples/diagnostics_smoke_test/main/app_main.c`）

```c
/* 制造一次非法内存写，触发崩溃；崩溃前先记一条 error 用于验证“跨重启保留” */
ESP_LOGE(TAG, "[count][%d] [crash_count][%" PRIu32 "] [excvaddr][0x0f] Crashing...",
         count, s_reset_count);
*(int *)0x0F = 0x10;   /* 写非法地址 → 崩溃 */
```
设备会重启，下次启动 Insights 上次启动的 error（因落 RTC）+ core dump 摘要。

### 4. 跨重启保留机制说明（来自 README/FEATURES）

- Critical 日志（error/warning/event）写 RTC memory，软复位后保留 → 上次崩溃前的 error 会随摘要一起上报。
- core dump 写 flash 的 coredump 分区 → 下次启动解析为摘要上报。
- **摘要上报成功后，core dump 会从 flash 分区擦除**。如需本地 `espcoredump` 调试，应先关掉 core dump 上报。

### 5. 上传固件包以拿到 backtrace 行号

```bash
idf.py build
# 上传 build/minimal_diagnostics-v1.0.zip 到 Dashboard → Firmware Images
```
云端用 ELF 把 backtrace 里的 PC 地址翻译为文件名:行号。不上传则只看到裸地址。

### 6. 想本地调试时禁用上报

```
# 在 menuconfig 把 ESP Insights core dump support 关闭，或不在 init insights
# 这样 coredump 留在 flash 供本地：idf.py 的 coredump 工具 / espcoredump.py
```

## 常见错误

| 错误 | 原因 | 解决方法 |
|---|---|---|
| 崩溃后仪表盘无 core dump | 三件套缺一：to_flash / ELF / 分区 | 三项 Kconfig + `coredump` 分区全部到位后 `erase_flash` 重烧 |
| `ESP_INSIGHTS_COREDUMP_ENABLE` 灰显 | 依赖未满足（非 to_flash 或非 ELF） | 同时启用 to_flash + ELF |
| backtrace 只见裸地址 | 未上传固件包 | 上传 `build/<project>-<ver>.zip` |
| 本地 espcoredump 取不到 | 摘要已上报后被擦除 | 调试期先关 insights core dump 上报 |
| RISC-V 板 backtrace 信息少 | on-device 解析限制 | `esp_diag_task_snapshot_get` 的 `bt_info` 仅 Xtensa 存在 |

## 参考

- `examples/minimal_diagnostics/sdkconfig.defaults` — coredump 配置
- `examples/minimal_diagnostics/partitions.csv` — `coredump` 分区
- `examples/diagnostics_smoke_test/main/app_main.c` — 崩溃触发与跨重启验证
- `components/esp_insights/Kconfig` — `ESP_INSIGHTS_COREDUMP_ENABLE` 依赖条件
- `FEATURES.md` “Core Dump” 段 — 摘要内容与擦除说明
