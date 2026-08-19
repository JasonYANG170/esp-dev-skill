# 自定义 Metrics（指标）

> **适用摘要**: 注册并上报自定义 metrics（如室温、CPU 负载），用于在仪表盘绘制随时间变化的曲线。覆盖 metadata 1.0 与 2.0 两套 API，以及各数据类型的上报函数。

> Version: selected repo/component version; align with the user project and dependency manifest.
> Evidence: `repos/esp-insights/resources/`, source/examples in `repos/esp-insights/`, and this recipe path `repos/esp-insights/recipes/custom_metrics.md`.
> Validation: draft metadata added from repository routing; verify APIs, Kconfig symbols, and component dependencies against the selected version.

## 触发意图

- "上报自定义指标"
- "esp_diag_metrics_register"
- "记录温度/负载曲线"
- "metrics 1.0 vs 2.0 API"
- "metrics tag 和 key"

## 前置条件

| 条件 | 要求 |
|---|---|
| Kconfig | `CONFIG_DIAG_ENABLE_METRICS=y`（默认 y） |
| 运行 | 已 `esp_insights_init(&config)` |
| metadata 版本 | 确认 `CONFIG_ESP_INSIGHTS_META_VERSION_10`：y=1.0（`add_*`），n=2.0（`report_*`） |
| 参考 | `FEATURES.md` “Custom Metrics” 段、头文件 `esp_diagnostics_metrics.h` |

## 分步说明

### 1. 元数据字段含义（来自头文件 `esp_diag_metrics_meta_t`）

| 字段 | 含义 |
|---|---|
| `tag` | 分组标签（2.0 用，如 `"temp"`） |
| `key` | 全局唯一标识（1.0 仅用它；2.0 用 tag+key 组合） |
| `label` | 仪表盘显示名 |
| `path` | 层级路径，用 `.` 分隔，如 `"room"`、`"heap.internal"` |
| `unit` | 单位字符串（可 NULL）；2.0 可用 `esp_diag_metrics_add_unit` 单独设 |
| `type` | `esp_diag_data_type_t`：BOOL/INT/UINT/FLOAT/STR/IPv4/MAC/NULL |

### 2. metadata 2.0（推荐，examples 默认）用法

```c
#include "esp_diagnostics.h"
#include "esp_diagnostics_metrics.h"

void metrics_setup(void)
{
    /* 注册：tag, key, label, path, type */
    esp_diag_metrics_register("temp", "room1", "Room temperature", "home.room1",
                              ESP_DIAG_DATA_TYPE_UINT);
}

void metrics_sample(void)
{
    uint32_t room_temp = read_room_temperature();   /* 应用自实现 */
    esp_diag_metrics_report_uint("temp", "room1", room_temp);
}
```
其他类型（来自 `esp_diagnostics_metrics.h`，2.0 分支）：
```c
esp_diag_metrics_report_bool ("temp", "heater",  on);
esp_diag_metrics_report_int  ("load", "cpu",     cpu_pct);
esp_diag_metrics_report_float("env",  "rh",      humidity);
esp_diag_metrics_report_ipv4 ("net",  "rssi_ip", ip_u32);
esp_diag_metrics_report_mac  ("net",  "ap_mac",  mac6);
esp_diag_metrics_report_str  ("sys",  "ver",     "1.0.0");
/* 通用：esp_diag_metrics_report(type, tag, key, val, val_sz, ts); */
/* 元数据/卸载：esp_diag_metrics_unregister(tag,key)、esp_diag_metrics_add_unit(tag,key,unit) */
```

### 3. metadata 1.0（旧，仅 key）用法

```c
/* 仅当 CONFIG_ESP_INSIGHTS_META_VERSION_10=y */
esp_diag_metrics_register("temp", "room1", "Room temperature", "home.room1",
                          ESP_DIAG_DATA_TYPE_UINT);  /* 注册时仍传 tag，但上报只用 key */
esp_diag_metrics_add_uint("room1", room_temp);

/* 其他 1.0 接口：
 * esp_diag_metrics_add_bool/int/float/ipv4/mac/str(key, val)
 * esp_diag_metrics_add(type, key, val, val_sz, ts)
 * esp_diag_metrics_add_unit(key, unit)、esp_diag_metrics_unregister(key)
 */
```

### 4. 上限与运行时查询（来自 Kconfig 与头文件）

- 默认最多注册 `CONFIG_DIAG_METRICS_MAX_COUNT`（默认 20）个 metric。
- `esp_diag_metrics_meta_get_all(&len)` 取全部元数据数组；`esp_diag_metrics_meta_print_all()` 打印。
- 卸载全部：`esp_diag_metrics_unregister_all()`。

### 5. timestamps

```c
uint64_t ts = esp_diag_timestamp_get();   /* NTP 同步则为 epoch，否则相对启动 µs */
esp_diag_metrics_report(ESP_DIAG_DATA_TYPE_UINT, "temp", "room1", &val, sizeof(val), ts);
```

## 常见错误

| 错误 | 原因 | 解决方法 |
|---|---|---|
| 1.0 下调用 `report_*` 链接失败 | 1.0 分支只有 `add_*` | 按当前 `META_VERSION_10` 选对 API |
| 2.0 下 `add_*` 找不到 | 2.0 只有 `report_*`（带 tag） | 改用 `report_*` |
| 注册返回 `ESP_ERR_NO_MEM` | 超过 `DIAG_METRICS_MAX_COUNT` | 调大该 Kconfig 或先 unregister |
| 仪表盘旧 metric 数据消失 | 迁移到 2.0 是破坏性变更 | 见 CHANGELOG 2024-02 说明；新面板重新累积 |
| 节点上报但曲线不刷新 | 上报间隔动态自适应 + Non-critical 区可能满 | 手动 `esp_insights_send_data()`；检查低内存事件 |

## 参考

- `components/esp_diagnostics/include/esp_diagnostics_metrics.h` — 全部 metrics API（1.0/2.0 双分支）
- `components/esp_diagnostics/include/esp_diagnostics.h` — `esp_diag_data_type_t`、`esp_diag_timestamp_get`
- `components/esp_diagnostics/Kconfig` — `DIAG_ENABLE_METRICS`、`DIAG_METRICS_MAX_COUNT`
- `FEATURES.md` “Custom Metrics” 段 — 字段示例
- `CHANGELOG.md` 2024-02 条目 — metadata 2.0 破坏性说明
