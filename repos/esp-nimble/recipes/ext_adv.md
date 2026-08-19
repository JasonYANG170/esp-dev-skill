# 扩展广播与周期广播

> **适用摘要**: 使用 NimBLE 扩展广播 API（`ble_gap_ext_adv_*`）配置多个 advertising instance，支持大广播数据、coded/2M PHY、周期广播（Periodic Advertising）。

> Version: selected repo/component version; align with the user project and dependency manifest.
> Evidence: `repos/esp-nimble/resources/`, source/examples in `repos/esp-nimble/`, and this recipe path `repos/esp-nimble/recipes/ext_adv.md`.
> Validation: draft metadata added from repository routing; verify APIs, Kconfig symbols, and component dependencies against the selected version.

## 触发意图

- "扩展广播 / extended advertising"
- "多实例广播"
- "周期广播 / periodic advertising"
- "coded PHY / 2M PHY 广播"
- "ble_gap_ext_adv_configure"

## 前置条件

| 条件 | 要求 |
|---|---|
| 参考示例 | `apps/ext_advertiser/src/main.c` |
| 配置 | `CONFIG_BT_NIMBLE_EXT_ADV=y`、`CONFIG_BT_NIMBLE_MAX_EXT_ADV_INSTANCES>=N` |
| 芯片 | 支持 BLE 5.0+ 控制器 |

## 分步说明

### 1. 配置一个不可连接扩展广播实例

扩展广播以 `instance` 为单位（0 ~ MAX-1），每个实例独立配置地址、PHY、数据。

```c
#include "host/ble_hs.h"
#include "host/util/util.h"
#include "os/os_mbuf.h"

static uint8_t id_addr_type;

static void start_non_connectable_ext(void)
{
    struct ble_gap_ext_adv_params params;
    struct os_mbuf *data;
    uint8_t instance = 0;
    int rc;

    memset(&params, 0, sizeof params);
    params.own_addr_type = id_addr_type;
    params.primary_phy   = BLE_HCI_LE_PHY_1M;
    params.secondary_phy = BLE_HCI_LE_PHY_1M;
    params.tx_power      = 127;     /* 127 = 控制器默认功率 */
    params.sid           = 0;       /* advertising set ID */

    /* 配置实例 */
    rc = ble_gap_ext_adv_configure(instance, &params, NULL, NULL, NULL);
    assert(rc == 0);

    /* 广播数据用 mbuf（最大可承载 1650 字节，远超 legacy 31 字节） */
    data = os_msys_get_pkthdr(600, 0);
    assert(data);
    rc = os_mbuf_append(data, ext_adv_data, 600);
    assert(rc == 0);
    rc = ble_gap_ext_adv_set_data(instance, data);
    assert(rc == 0);

    /* 启动：duration=0 永久，max_events=0 无限 */
    rc = ble_gap_ext_adv_start(instance, 0, 0);
    assert(rc == 0);
}
```

### 2. 可扫描扩展广播实例（带 scan response）

```c
static void start_scannable_ext(void)
{
    struct ble_gap_ext_adv_params params;
    struct os_mbuf *data;
    uint8_t instance = 1;
    ble_addr_t addr;
    int rc;

    memset(&params, 0, sizeof params);
    params.scannable = 1;
    params.own_addr_type = BLE_OWN_ADDR_RANDOM;
    params.primary_phy   = BLE_HCI_LE_PHY_1M;
    params.secondary_phy = BLE_HCI_LE_PHY_1M;
    params.sid = 1;

    rc = ble_gap_ext_adv_configure(instance, &params, NULL, gap_cb, NULL);
    assert(rc == 0);

    /* 用 NRPA 作为本实例地址 */
    ble_hs_id_gen_rnd(1, &addr);
    ble_gap_ext_adv_set_addr(instance, &addr);

    /* 仅设置 scan response（可扫描实例的 adv data 可另设） */
    data = os_msys_get_pkthdr(sizeof scan_rsp, 0);
    os_mbuf_append(data, scan_rsp, sizeof scan_rsp);
    ble_gap_ext_adv_rsp_set_data(instance, data);

    ble_gap_ext_adv_start(instance, 0, 0);
}
```

### 3. 周期广播（Periodic Advertising）

周期广播基于一个不可连接扩展实例，把数据周期性发送给已同步的观察者。

```c
static void start_periodic(void)
{
    struct ble_gap_ext_adv_params eparams;
    struct ble_gap_periodic_adv_params pparams;
    struct os_mbuf *data;
    uint8_t instance = 5;
    int rc;

    /* 先建一个不可连接扩展实例 */
    memset(&eparams, 0, sizeof eparams);
    eparams.own_addr_type = BLE_OWN_ADDR_RANDOM;
    eparams.primary_phy = BLE_HCI_LE_PHY_1M;
    eparams.secondary_phy = BLE_HCI_LE_PHY_1M;
    eparams.sid = 5;
    ble_gap_ext_adv_configure(instance, &eparams, NULL, NULL, NULL);
    ble_hs_id_gen_rnd(1, &addr);
    ble_gap_ext_adv_set_addr(instance, &addr);

    /* 配置周期广播参数 */
    memset(&pparams, 0, sizeof pparams);
    pparams.include_tx_power = 1;
    pparams.itvl_min = 160;    /* 1.25ms 单位 → 200ms */
    pparams.itvl_max = 240;    /* → 300ms */
    rc = ble_gap_periodic_adv_configure(instance, &pparams);
    assert(rc == 0);

    /* 周期数据 */
    data = os_msys_get_pkthdr(sizeof periodic_data, 0);
    os_mbuf_append(data, periodic_data, sizeof periodic_data);
    ble_gap_periodic_adv_set_data(instance, data);

    ble_gap_periodic_adv_start(instance);
    ble_gap_ext_adv_start(instance, 0, 0);
}
```

### 关键 API

| 函数 | 说明 |
|---|---|
| `ble_gap_ext_adv_configure(instance, params, cb, ...) ` | 配置实例 |
| `ble_gap_ext_adv_set_addr(instance, &addr)` | 设置实例地址 |
| `ble_gap_ext_adv_set_data(instance, om)` | 设置广播数据（mbuf） |
| `ble_gap_ext_adv_rsp_set_data(instance, om)` | 设置扫描响应 |
| `ble_gap_ext_adv_start(instance, duration, max_events)` | 启动 |
| `ble_gap_ext_adv_stop(instance)` | 停止 |
| `ble_gap_ext_adv_remove(instance)` | 移除实例 |
| `ble_gap_ext_adv_active(instance)` | 是否活跃 |
| `ble_gap_periodic_adv_configure(instance, pparams)` | 配置周期广播 |
| `ble_gap_periodic_adv_set_data(instance, om)` | 周期数据 |
| `ble_gap_periodic_adv_start(instance)` / `_stop` | 启停周期广播 |
| `ble_gap_periodic_adv_sync_create(...)` | 观察者侧建立周期同步 |

### `ble_gap_ext_adv_params` 关键字段

| 字段 | 含义 |
|---|---|
| `own_addr_type` | 本机地址类型 |
| `primary_phy` / `secondary_phy` | 主 / 副 PHY（`BLE_HCI_LE_PHY_1M` / `_2M` / `_CODED`） |
| `tx_power` | 发射功率（127=默认） |
| `sid` | advertising set ID |
| `scannable` / `connectable` / `legacy_pdu` | 模式标志 |
| `itvl_min` / `itvl_max` | 广播间隔 |
| `scan_req_notif` | 是否上报 SCAN_REQ |

## 常见错误

| 错误 | 原因 | 解决方法 |
|---|---|---|
| 数据仍被限制 31 字节 | 设了 `legacy_pdu=1` | 关闭 `legacy_pdu` 用扩展 PDU |
| `ext_adv_configure` 返回 `BLE_HS_EALREADY` | instance 已配置 | 先 `ext_adv_remove` |
| 用了 legacy `ble_gap_adv_set_fields` | 用错 API | 扩展广播用 `ble_gap_ext_adv_set_data`（mbuf） |
| coded PHY 不生效 | 控制器不支持 / 未启用 | 确认芯片支持 BLE 5 coded PHY |
| 周期广播观察者同步失败 | SID / 地址不匹配 | 观察者用相同 SID 与地址调 `periodic_adv_sync_create` |

## 参考

- `apps/ext_advertiser/src/main.c` — 含 non-connectable / scannable / legacy-duration / max-events / periodic 五种实例配置
- 头文件：`nimble/host/include/host/ble_gap.h`（`ble_gap_ext_adv_*`、`ble_gap_periodic_adv_*`、`ble_gap_ext_adv_params`）
