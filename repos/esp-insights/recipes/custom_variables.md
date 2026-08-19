# 自定义 Variables（变量）

> **适用摘要**: 注册并上报自定义 variables（如当前关联的 station 数、���备状态机当前态），区别于 metrics：variables 强调“当前值”而非“随时间序列”。覆盖 1.0 与 2.0 API 及各数据类型。

> Version: selected repo/component version; align with the user project and dependency manifest.
> Evidence: `repos/esp-insights/resources/`, source/examples in `repos/esp-insights/`, and this recipe path `repos/esp-insights/recipes/custom_variables.md`.
> Validation: draft metadata added from repository routing; verify APIs, Kconfig symbols, and component dependencies against the selected version.

## 触发意图

- "上报当前状态变量"
- "esp_diag_variable_register"
- "记录 IP / 关联站点数"
- "variables 与 metrics 区别"
- "variable tag 和 key"

## 前置条件

| 条件 | 要求 |
|---|---|
| Kconfig | `CONFIG_DIAG_ENABLE_VARIABLES=y`（默认 y） |
| 运行 | 已 `esp_insights_init(&config)` |
| metadata 版本 | `CONFIG_ESP_INSIGHTS_META_VERSION_10`：y=1.0（`add_*`），n=2.0（`report_*`） |
| 参考 | `FEATURES.md` “Variables”/“Custom Variables”、头文件 `esp_diagnostics_variables.h` |

## 分步说明

### 1. metrics vs variables（来自 README/FEATURES）

- **metrics**：时变序列，仪表盘画曲线（如 RSSI、free heap）。
- **variables**：当前值更重要（如当前 IP、状态机当前态、关联站点数）。

### 2. metadata 2.0 用法（来自 `esp_diagnostics_variables.h`）

```c
#include "esp_diagnostics.h"
#include "esp_diagnostics_variables.h"

void vars_setup(void)
{
    esp_diag_variable_register("wifi", "sta_cnt", "STAs associated", "wifi.sta",
                               ESP_DIAG_DATA_TYPE_UINT);
}

/* 在 WIFI_EVENT_AP_STACONNECTED / DISCONNECTED 里维护 sta_cnt 后上报当前值 */
void on_sta_changed(uint32_t sta_cnt)
{
    esp_diag_variable_report_uint("wifi", "sta_cnt", sta_cnt);
}
```
其他类型（2.0 分支）：
```c
esp_diag_variable_report_bool ("pwr",  "charger", charging);
esp_diag_variable_report_int  ("fsm",  "state",   cur_state);
esp_diag_variable_report_float("env",  "rh",      humidity);
esp_diag_variable_report_ipv4 ("net",  "ip",      ip_u32);
esp_diag_variable_report_mac  ("net",  "ap_mac",  mac6);
esp_diag_variable_report_str  ("sys",  "fw",      "1.0.0");
/* 通用：esp_diag_variable_report(type, tag, key, val, val_sz, ts); */
/* 元数据：esp_diag_variable_unregister(tag,key)、esp_diag_variable_add_unit(tag,key,unit) */
```

### 3. metadata 1.0 用法（仅 key）

```c
/* 仅当 CONFIG_ESP_INSIGHTS_META_VERSION_10=y */
esp_diag_variable_register("wifi", "sta_cnt", "STAs associated", "wifi.sta",
                           ESP_DIAG_DATA_TYPE_UINT);
esp_diag_variable_add_uint("sta_cnt", sta_cnt);

/* 其他 1.0 接口：
 * esp_diag_variable_add_bool/int/float/ipv4/mac/str(key, val)
 * esp_diag_variable_add(type, key, val, val_sz, ts)
 * esp_diag_variable_add_unit(key, unit)、esp_diag_variable_unregister(key)
 */
```

### 4. 内置 network variables（来自 `esp_diagnostics_network_variables.h`）

启用 `CONFIG_DIAG_ENABLE_NETWORK_VARIABLES=y` 后，`esp_insights_init` 会自动注册并按变化上报：
- Wi-Fi：SSID、BSSID、channel、auth mode、连接状态、断连原因
- IP：IPv4 地址、netmask、gateway

进阶变量用 `CONFIG_DIAG_MORE_NETWORK_VARS=y`。初始化/反初始化：
```c
esp_diag_network_variables_init();    /* 通常由 esp_insights_init 内部完成 */
esp_diag_network_variables_deinit();
```

### 5. 上限与元数据查询（来自 Kconfig/头文件）

- 最多 `CONFIG_DIAG_VARIABLES_MAX_COUNT`（默认 20）个 variable。
- `esp_diag_variable_meta_get_all(&len)` / `esp_diag_variable_meta_print_all()`。
- `esp_diag_variables_deinit()` / `esp_diag_variable_unregister_all()`。

## 常见错误

| 错误 | 原因 | 解决方法 |
|---|---|---|
| 1.0 下 `report_*` 找不到 | 1.0 用 `add_*` | 按 `META_VERSION_10` 选 API |
| 2.0 下 `add_*` 找不到 | 2.0 用 `report_*`（带 tag） | 改用 `report_*` |
| 网络变量不出现 | 未启用 network variables | `CONFIG_DIAG_ENABLE_NETWORK_VARIABLES=y` |
| 注册超限 | 超 `DIAG_VARIABLES_MAX_COUNT` | 调大或 unregister |
| 仪表盘旧数据消失 | 迁移 2.0 破坏性 | 见 CHANGELOG，新面板重新累积 |

## 参考

- `components/esp_diagnostics/include/esp_diagnostics_variables.h` — 全部 variables API（1.0/2.0）
- `components/esp_diagnostics/include/esp_diagnostics_network_variables.h` — network variables
- `components/esp_diagnostics/include/esp_diagnostics.h` — `esp_diag_data_type_t`、`esp_diag_timestamp_get`
- `components/esp_diagnostics/Kconfig` — `DIAG_ENABLE_VARIABLES`、`DIAG_ENABLE_NETWORK_VARIABLES`、`DIAG_MORE_NETWORK_VARS`、`DIAG_VARIABLES_MAX_COUNT`
- `FEATURES.md` “Variables”/“Custom Variables”/“Network Variables” 段
