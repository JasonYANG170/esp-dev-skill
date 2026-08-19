# 日志采集与自定义事件

> **适用摘要**: 配置 ESP-Insights 采集 error/warning 日志与自定义事件，包括 `log_type` 位掩码、按 tag 调级别、`ESP_DIAG_EVENT` 宏，以及“日志不显示”的诊断思路。

> Version: selected repo/component version; align with the user project and dependency manifest.
> Evidence: `repos/esp-insights/resources/`, source/examples in `repos/esp-insights/`, and this recipe path `repos/esp-insights/recipes/log_capture.md`.
> Validation: draft metadata added from repository routing; verify APIs, Kconfig symbols, and component dependencies against the selected version.

## 触发意图

- "采集 ESP_LOGE / ESP_LOGW"
- "上报自定义事件"
- "按 tag 控制日志级别"
- "ESP_DIAG_EVENT 怎么用"
- "日志在仪表盘不显示"

## 前置条件

| 条件 | 要求 |
|---|---|
| Kconfig | `CONFIG_ESP_INSIGHTS_ENABLED=y`（自动选中 `DIAG_ENABLE_WRAP_LOG_FUNCTIONS`） |
| 运行 | 已 `esp_insights_init(&config)` |
| 参考 | `examples/diagnostics_smoke_test/main/app_main.c`（error/warning/event 用法） |

## 分步说明

### 1. log_type 位掩码（来自 `esp_diagnostics.h` 枚举）

```c
ESP_DIAG_LOG_TYPE_ERROR   = 1 << 0
ESP_DIAG_LOG_TYPE_WARNING = 1 << 1
ESP_DIAG_LOG_TYPE_EVENT   = 1 << 2
```
`esp_insights_config_t.log_type` 是这些的位或。诊断数据在 RTC store 中分两区：
- Critical（error/warning/event）
- Non-critical（metrics/variables）

```c
esp_insights_config_t config = {
    .log_type = ESP_DIAG_LOG_TYPE_ERROR | ESP_DIAG_LOG_TYPE_WARNING | ESP_DIAG_LOG_TYPE_EVENT,
};
esp_insights_init(&config);
```

### 2. 普通 error/warning 直接用 ESP_LOG 宏即可（来自 smoke_test）

```c
static const char *TAG = "diag_smoke";

ESP_LOGE(TAG, "[count][%d] read sensor failed", count);
ESP_LOGW(TAG, "[count][%d] retrying", count);
```
启用日志包装后，这些会被诊断钩子自动捕获，无需手动调用。

### 3. 自定义事件用 ESP_DIAG_EVENT 宏（来自 `esp_diagnostics.h`）

```c
ESP_DIAG_EVENT(TAG, "[count][%d] user_button_pressed", count);
```
该宏展开为同时调用 `esp_diag_log_event(tag, "EV (...) %s: " format, ...)` 与 `ESP_LOGI(tag, format, ...)`。
> 注意：`ESP_DIAG_EVENT` 走的是 `ESP_DIAG_LOG_TYPE_EVENT` 通道，不受 `esp_log_level_set()` 影响；但 `log_type` 必须包含 `EVENT` 位才采集。

### 4. 按 tag 动态调级别（来自 FEATURES.md）

```c
/* 关闭某 tag 的所有日志 */
esp_log_level_set("noisy_tag", ESP_LOG_NONE);

/* 只保留 error */
esp_log_level_set("wifi_drv", ESP_LOG_ERROR);

/* error + warning */
esp_log_level_set("sensor", ESP_LOG_WARN);
```
同名及更低 verbosity 才会既上串口又上报。`ESP_DIAG_EVENT` 不受此影响。

### 5. 运行时启停诊断日志钩子（来自 `esp_diagnostics.h`）

```c
/* 只采集 error */
esp_diag_log_hook_enable(ESP_DIAG_LOG_TYPE_ERROR);

/* 采集 error + warning + event */
esp_diag_log_hook_enable(ESP_DIAG_LOG_TYPE_ERROR | ESP_DIAG_LOG_TYPE_WARNING | ESP_DIAG_LOG_TYPE_EVENT);

/* 临时停掉事件采集 */
esp_diag_log_hook_disable(ESP_DIAG_LOG_TYPE_EVENT);
```

### 6. 日志不显示的排查清单

- `Default log verbosity` 被设为 No output → ESP_LOGE/W 在预处理期被移除，钩子也消失。改为 >= Warning，或把 Console channel 设 None（日志仍进诊断、串口静默）。
- `log_type` 没包含对应位。
- RTC 存储满 → 监听 `ESP_DIAG_DATA_STORE_EVENT_CRITICAL_DATA_LOW_MEM`。
- 未上传固件包 → 显示 Firmware Image missing（字符串无法交叉引用）。

## 常见错误

| 错误 | 原因 | 解决方法 |
|---|---|---|
| error/warn 在仪表盘消失 | Default log verbosity = No output | 改为 Warning 及以上，或 Console channel = None |
| `ESP_DIAG_EVENT` 不上报 | log_type 缺 EVENT 位 | config 加 `\| ESP_DIAG_LOG_TYPE_EVENT` |
| 日志量大被丢 | RTC 满载 | 调级别/减量，或调大 `CONFIG_RTC_STORE_CRITICAL_DATA_SIZE` |
| 多设备日志洪流 | 全量 error/warn 过多 | 按 tag 用 `esp_log_level_set` 收敛 |
| 想外部包装日志与 insights 共存 | 二者都用 `--wrap` | 启用 `CONFIG_DIAG_USE_EXTERNAL_LOG_WRAP`，在 `__wrap_esp_log_writev` 内调 `esp_diag_log_writev()` |

## 参考

- `examples/diagnostics_smoke_test/main/app_main.c` — error/warning/event 实战
- `components/esp_diagnostics/include/esp_diagnostics.h` — `ESP_DIAG_LOG_TYPE_*`、`ESP_DIAG_EVENT`、`esp_diag_log_hook_enable/disable`、`esp_diag_log_writev/write`
- `FEATURES.md` “Logs” 段 — 级别配置与 console channel 说明
- `components/esp_diagnostics/Kconfig` — `DIAG_LOG_DROP_WIFI_LOGS`、`DIAG_LOG_MSG_ARG_FORMAT_*`
