# 可连接广播外设

> **适用摘要**: 实现一个传统可连接 BLE 外设：设置广播数据 / 扫描响应、启动非定向广播、处理连接与断开并在断开后恢复广播。

## 触发意图

- "BLE 外设 / 从机"
- "广播 / advertising"
- "可连接广播"
- "peripheral"
- "ble_gap_adv_start"

## 前置条件

| 条件 | 要求 |
|---|---|
| 参考示例 | `apps/bleprph/src/main.c`、`apps/peripheral/src/main.c`、`apps/blehr/src/main.c` |
| 配置 | `CONFIG_BT_NIMBLE_ROLE_PERIPHERAL=y` |
| 时序 | 在 `on_sync` 中调用 |

## 分步说明

### 1. 准备广播参数与数据

```c
#include "host/ble_hs.h"
#include "host/util/util.h"
#include "services/gap/ble_svc_gap.h"

static uint8_t own_addr_type;
static uint16_t conn_handle = BLE_HS_CONN_HANDLE_NONE;
static int gap_event(struct ble_gap_event *event, void *arg);

static void advertise(void)
{
    struct ble_gap_adv_params adv_params;
    struct ble_hs_adv_fields fields;
    struct ble_hs_adv_fields rsp_fields;
    const char *name = ble_svc_gap_device_name();
    int rc;

    /* 广播数据：flags + tx power + 16-bit service uuid */
    memset(&fields, 0, sizeof fields);
    fields.flags = BLE_HS_ADV_F_DISC_GEN | BLE_HS_ADV_F_BREDR_UNSUP;
    fields.tx_pwr_lvl_is_present = 1;
    fields.tx_pwr_lvl = BLE_HS_ADV_TX_PWR_LVL_AUTO;
    fields.uuids16 = (ble_uuid16_t[]){ BLE_UUID16_INIT(0x180D) };  /* HRU */
    fields.num_uuids16 = 1;
    fields.uuids16_is_complete = 1;

    rc = ble_gap_adv_set_fields(&fields);
    assert(rc == 0);

    /* 扫描响应：完整设备名 */
    memset(&rsp_fields, 0, sizeof rsp_fields);
    rsp_fields.name = (uint8_t *)name;
    rsp_fields.name_len = strlen(name);
    rsp_fields.name_is_complete = 1;
    rc = ble_gap_adv_rsp_set_fields(&rsp_fields);
    assert(rc == 0);

    /* 启动非定向可连接广播 */
    memset(&adv_params, 0, sizeof adv_params);
    adv_params.conn_mode = BLE_GAP_CONN_MODE_UND;
    adv_params.disc_mode = BLE_GAP_DISC_MODE_GEN;

    rc = ble_gap_adv_start(own_addr_type, NULL, BLE_HS_FOREVER,
                           &adv_params, gap_event, NULL);
    assert(rc == 0);
}
```

### 2. 处理 GAP 事件

```c
static int gap_event(struct ble_gap_event *event, void *arg)
{
    struct ble_gap_conn_desc desc;
    int rc;

    switch (event->type) {
    case BLE_GAP_EVENT_CONNECT:
        if (event->connect.status == 0) {
            conn_handle = event->connect.conn_handle;
            rc = ble_gap_conn_find(conn_handle, &desc);
            assert(rc == 0);
            /* 打印连接信息 */
        } else {
            advertise();   /* 连接失败，恢复广播 */
        }
        return 0;

    case BLE_GAP_EVENT_DISCONNECT:
        conn_handle = BLE_HS_CONN_HANDLE_NONE;
        advertise();       /* 断开后恢复广播 */
        return 0;

    case BLE_GAP_EVENT_ADV_COMPLETE:
        /* 限时广播结束（duration_ms 非 FOREVER 时触发） */
        advertise();
        return 0;

    case BLE_GAP_EVENT_MTU:
        MODLOG_DFLT(INFO, "mtu update; conn=%d mtu=%d\n",
                    event->mtu.conn_handle, event->mtu.value);
        return 0;

    case BLE_GAP_EVENT_SUBSCRIBE:
        /* 客户端订阅 CCCD（见 notify.md） */
        return 0;
    }
    return 0;
}
```

### 关键 API

| 函数 | 说明 |
|---|---|
| `ble_gap_adv_set_fields(const struct ble_hs_adv_fields *)` | 设置广播数据 |
| `ble_gap_adv_rsp_set_fields(const struct ble_hs_adv_fields *)` | 设置扫描响应 |
| `ble_gap_adv_start(own_addr_type, direct_addr, duration_ms, adv_params, cb, cb_arg)` | 启动广播 |
| `ble_gap_adv_stop(void)` | 停止广播 |
| `ble_gap_adv_active(void)` | 是否正在广播 |
| `ble_gap_conn_find(conn_handle, &desc)` | 按 handle 查连接描述 |

### 广播 flags（`ble_hs_adv.h`）

| 标志 | 含义 |
|---|---|
| `BLE_HS_ADV_F_DISC_GEN` | 一般可发现 |
| `BLE_HS_ADV_F_DISC_LTD` | 有限可发现 |
| `BLE_HS_ADV_F_BREDR_UNSUP` | 仅 BLE（不支持 BR/EDR） |

## 常见错误

| 错误 | 原因 | 解决方法 |
|---|---|---|
| `ble_gap_adv_set_fields` 返回 `BLE_HS_EMSGSIZE` | adv 数据超 31 字节 | 长名称移到 scan response |
| `ble_gap_adv_start` 返回 `BLE_HS_EBUSY` | 已在广播 | 先 `ble_gap_adv_stop()` |
| 广播名称显示不全 | name_is_complete=1 但被截断 | 截断时设 `name_is_complete=0` |
| 连接后无法恢复广播 | 未在 DISCONNECT 中重启 | DISCONNECT 分支调用 `advertise()` |

## 参考

- `apps/bleprph/src/main.c` — 完整事件处理（CONNECT/DISCONNECT/ENC_CHANGE/SUBSCRIBE/MTU/REPEAT_PAIRING）
- `apps/peripheral/src/main.c` — 含 scan response 的精简广播
- `apps/blehr/src/main.c` — 配合通知的广播
- 头文件：`nimble/host/include/host/ble_gap.h`、`ble_hs_adv.h`
