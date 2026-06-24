# 系统级 Heap / Wi-Fi Metrics 与 Network Variables

> **适用摘要**: 启用 ESP-Insights 内置的系统 metrics（free/最大块/历史最小空闲堆，内部 RAM 与 PSRAM；Wi-Fi RSSI 与历史最小 RSSI）与 network variables，并按需主动 dump。

## 触发意图

- "采集 free heap 指标"
- "Wi-Fi RSSI 上报"
- "heap metrics 轮询间隔"
- "esp_diag_heap_metrics_dump"
- "network variables 自动采集"

## 前置条件

| 条件 | 要求 |
|---|---|
| Kconfig | `CONFIG_DIAG_ENABLE_METRICS=y` + `CONFIG_DIAG_ENABLE_HEAP_METRICS=y` + `CONFIG_DIAG_ENABLE_WIFI_METRICS=y`；variables 相关 `CONFIG_DIAG_ENABLE_VARIABLES=y` + `CONFIG_DIAG_ENABLE_NETWORK_VARIABLES=y` |
| 运行 | 已 `esp_insights_init(&config)`（内部按 Kconfig 初始化系统 metrics/network vars） |
| 参考 | `examples/minimal_diagnostics/main/app_main.c`（dump 用法） |

## 分步说明

### 1. sdkconfig.defaults（来自 `examples/minimal_diagnostics/sdkconfig.defaults`）

```
CONFIG_DIAG_ENABLE_METRICS=y
CONFIG_DIAG_ENABLE_HEAP_METRICS=y
CONFIG_DIAG_ENABLE_WIFI_METRICS=y
CONFIG_DIAG_ENABLE_VARIABLES=y
CONFIG_DIAG_ENABLE_NETWORK_VARIABLES=y
```

### 2. Heap Metrics（来自 `esp_diagnostics_system_metrics.h`）

采集内容（内部 RAM，含 PSRAM 若有）：free heap、largest free block、历史最小 free。
- 周期：`CONFIG_DIAG_HEAP_POLLING_INTERVAL`（默认 30s，范围 30–86400）。
- 运行时改间隔：`esp_diag_heap_metrics_reset_interval(period)`，传 0 停采集。
- 主动 dump：`esp_diag_heap_metrics_dump()`（立即采集并打印 + 上报）。

```c
#include "esp_diagnostics_system_metrics.h"

/* 周期主动 dump（minimal_diagnostics 每 10 分钟一次） */
#define METRICS_DUMP_INTERVAL_TICKS  ((600 * 1000) / portTICK_PERIOD_MS)

while (true) {
    esp_diag_heap_metrics_dump();
    esp_diag_wifi_metrics_dump();
    vTaskDelay(METRICS_DUMP_INTERVAL_TICKS);
}
```

### 3. Wi-Fi Metrics（来自 `esp_diagnostics_system_metrics.h`）

采集：RSSI（每 30s 采样，跨步长如 5dB 才上报）+ 历史最小 RSSI。
- 周期：`CONFIG_DIAG_WIFI_POLLING_INTERVAL`（默认 30s，范围 30–86400）。
- 阈值（历史最小 RSSI 触发）：`esp_wifi_set_rssi_threshold()`（ESP-IDF API）。
- 运行时改间隔：`esp_diag_wifi_metrics_reset_interval(period)`。
- 主动 dump：`esp_diag_wifi_metrics_dump()`。

### 4. Network Variables（来自 `esp_diagnostics_network_variables.h`）

启用后自动采集并在变化时上报：
- Wi-Fi：SSID、BSSID、channel、auth mode、连接状态、断连原因
- IP：IPv4 地址、netmask、gateway

进阶：`CONFIG_DIAG_MORE_NETWORK_VARS=y`。手动 init/deinit（通常 `esp_insights_init` 已内部完成）：
```c
esp_diag_network_variables_init();
esp_diag_network_variables_deinit();
```

### 5. 冒烟测试参考（来自 `examples/diagnostics_smoke_test/main/app_main.c`）

```c
/* smoke_test 任务里循环 dump heap metrics，配合 malloc/free 制造曲线波动 */
esp_diag_heap_metrics_dump();
/* ... malloc / free 制造内存变化 ... */
```

## 常见错误

| 错误 | 原因 | 解决方法 |
|---|---|---|
| `esp_diag_heap_metrics_dump` 链接失败 | 未启 heap metrics | `CONFIG_DIAG_ENABLE_HEAP_METRICS=y` |
| `esp_diag_wifi_metrics_dump` 链接失败 | 未启 wifi metrics | `CONFIG_DIAG_ENABLE_WIFI_METRICS=y` |
| RSSI 曲线无数据 | Wi-Fi 未连 / 阈值过严 | 确认已联网；调 `esp_wifi_set_rssi_threshold` |
| 周期太长看不到数据 | 默认 30s + 动态上报间隔 | 主动 dump 或调小 polling interval |
| network vars 不出现 | 未启 network variables | `CONFIG_DIAG_ENABLE_NETWORK_VARIABLES=y` |

## 参考

- `examples/minimal_diagnostics/main/app_main.c` — `esp_diag_heap_metrics_dump` / `esp_diag_wifi_metrics_dump` 实战
- `examples/diagnostics_smoke_test/main/app_main.c` — 配合 malloc/free 的 heap 曲线
- `components/esp_diagnostics/include/esp_diagnostics_system_metrics.h` — heap/wifi metrics API
- `components/esp_diagnostics/include/esp_diagnostics_network_variables.h` — network variables
- `components/esp_diagnostics/Kconfig` — `DIAG_ENABLE_HEAP_METRICS`、`DIAG_HEAP_POLLING_INTERVAL`、`DIAG_ENABLE_WIFI_METRICS`、`DIAG_WIFI_POLLING_INTERVAL`、`DIAG_ENABLE_NETWORK_VARIABLES`、`DIAG_MORE_NETWORK_VARS`
- `FEATURES.md` “Heap Metrics”/“Wi-Fi Metrics”/“Network Variables” 段
