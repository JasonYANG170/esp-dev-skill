# 扫描（Observer）

> **适用摘要**: 实现 BLE 扫描（passive / active），解析广播报告 `BLE_GAP_EVENT_DISC`，并解析 `ble_hs_adv_fields` 各字段。

> Version: selected repo/component version; align with the user project and dependency manifest.
> Evidence: `repos/esp-nimble/resources/`, source/examples in `repos/esp-nimble/`, and this recipe path `repos/esp-nimble/recipes/scanner.md`.
> Validation: draft metadata added from repository routing; verify APIs, Kconfig symbols, and component dependencies against the selected version.

## 触发意图

- "扫描 / scan / discovery"
- "被动扫描 / active scan"
- "解析广播数据"
- "ble_gap_disc"
- "观察者 / observer"

## 前置条件

| 条件 | 要求 |
|---|---|
| 参考示例 | `apps/scanner/src/main.c`、`apps/blecent/src/main.c` |
| 配置 | `CONFIG_BT_NIMBLE_ROLE_OBSERVER=y` |
| 时序 | 在 `on_sync` 中调用 |

## 分步说明

### 1. 配置扫描参数并启动

```c
#include "host/ble_hs.h"
#include "host/util/util.h"

static int scan_event(struct ble_gap_event *event, void *arg);

static void scan(void)
{
    struct ble_gap_disc_params scan_params;
    int rc;

    scan_params.itvl            = 500;    /* 0.625ms 单位 → 312.5ms */
    scan_params.window          = 250;    /* 0.625ms 单位 → 156.25ms */
    scan_params.filter_policy   = 0;
    scan_params.limited         = 0;
    scan_params.passive         = 1;      /* 1=被动扫描；0=主动扫描 */
    scan_params.filter_duplicates = 1;    /* 过滤重复广播 */

    /* 用 NRPA 时 own_addr_type 为 BLE_OWN_ADDR_RANDOM */
    rc = ble_gap_disc(BLE_OWN_ADDR_RANDOM, 1000,
                      &scan_params, scan_event, NULL);
    assert(rc == 0);
}
```

> 主动扫描（`passive=0`）会发送 SCAN_REQ 以获取 scan response，被动扫描只收广播包。

### 2. 处理广播报告

```c
static int scan_event(struct ble_gap_event *event, void *arg)
{
    struct ble_hs_adv_fields fields;
    int rc;

    switch (event->type) {
    case BLE_GAP_EVENT_DISC:
        /* 解析广播数据 */
        rc = ble_hs_adv_parse_fields(&fields, event->disc.data,
                                     event->disc.length_data);
        if (rc != 0) {
            return 0;
        }
        print_adv_fields(&fields);
        /* event->disc.addr 是发送方地址 */
        return 0;

    case BLE_GAP_EVENT_DISC_COMPLETE:
        MODLOG_DFLT(INFO, "scan complete; reason=%d\n",
                    event->disc_complete.reason);
        scan();   /* 重新启动 */
        return 0;
    }
    return 0;
}
```

### 3. 扫描报告结构 `ble_gap_disc_desc`（`event->disc`）

| 字段 | 含义 |
|---|---|
| `addr` | 发送方 BLE 地址（`ble_addr_t`，含 `type` 与 `val[6]`） |
| `event_type` | 广播类型，如 `BLE_HCI_ADV_RPT_EVTYPE_ADV_IND` / `_DIR_IND` / `_SCAN_RSP` |
| `length_data` | 广播数据长度 |
| `data` | 广播数据指针 |
| `rssi` | 信号强度 |

### 4. 扫描参数结构 `ble_gap_disc_params`

| 字段 | 含义 |
|---|---|
| `itvl` | 扫描间隔（0.625ms 单位，0=默认） |
| `window` | 扫描窗口（0.625ms 单位） |
| `filter_policy` | 过滤策略（0=无白名单） |
| `limited` | 有限发现模式 |
| `passive` | 1=被动，0=主动 |
| `filter_duplicates` | 1=过滤重复 |

### 关键 API

| 函数 | 说明 |
|---|---|
| `ble_gap_disc(own_addr_type, duration_ms, params, cb, cb_arg)` | 启动扫描 |
| `ble_gap_ext_disc(own_addr_type, duration, period, ...) ` | 扩展扫描 |
| `ble_gap_disc_cancel(void)` | 取消扫描 |
| `ble_gap_disc_active(void)` | 是否正在扫描 |
| `ble_hs_adv_parse_fields(&fields, data, len)` | 解析广播数据 |

## 常见错误

| 错误 | 原因 | 解决方法 |
|---|---|---|
| `ble_gap_disc` 返回 `BLE_HS_EBUSY` | 已在扫描 | 先 `ble_gap_disc_cancel()` |
| own_addr_type 与 NRPA 不匹配 | 类型错误 | NRPA 用 `BLE_OWN_ADDR_RANDOM` |
| 扫描立刻停止 | duration_ms 过小 | 增大 duration 或用 `BLE_HS_FOREVER` |
| 收不到 scan response | 设为被动扫描 | 主动扫描设 `passive=0` |
| 重复报告刷屏 | 未过滤重复 | 设 `filter_duplicates=1` |

## 参考

- `apps/scanner/src/main.c` — NRPA 扫描 + 字段解析 + 自动重启
- `apps/blecent/src/main.c` — 扫描后选择性连接（`blecent_should_connect`）
- 头文件：`nimble/host/include/host/ble_gap.h`（`ble_gap_disc`、`ble_gap_disc_params`）、`ble_hs_adv.h`
