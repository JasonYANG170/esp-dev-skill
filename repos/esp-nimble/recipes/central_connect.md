# 中心连接与 GATT 客户端

> **适用摘要**: 实现中心角色：扫描到目标后取消扫描、`ble_gap_connect` 发起连接、进行服务发现、对特征执行读 / 写 / 订阅。

## 触发意图

- "BLE 中心 / 主机 / central"
- "连接从机 / ble_gap_connect"
- "服务发现 / service discovery"
- "GATT 客户端读 / 写 / 订阅"
- "ble_gattc_read / ble_gattc_write"

## 前置条件

| 条件 | 要求 |
|---|---|
| 参考示例 | `apps/blecent/src/main.c`、`apps/blecent/src/peer.c` |
| 配置 | `CONFIG_BT_NIMBLE_ROLE_CENTRAL=y` |

## 分步说明

### 1. 扫描到目标后停止扫描并发起连接

```c
#include "host/ble_hs.h"
#include "host/util/util.h"

static int gap_event(struct ble_gap_event *event, void *arg);

static void connect_if_interesting(const struct ble_gap_disc_desc *disc)
{
    uint8_t own_addr_type;
    int rc;

    if (!should_connect(disc)) return;

    /* 关键：连接前必须先取消扫描 */
    rc = ble_gap_disc_cancel();
    if (rc != 0) return;

    rc = ble_hs_id_infer_auto(0, &own_addr_type);
    assert(rc == 0);

    /* 发起连接，超时 30s */
    rc = ble_gap_connect(own_addr_type, &disc->addr, 30000,
                         NULL, gap_event, NULL);
    if (rc != 0) {
        MODLOG_DFLT(ERROR, "connect failed; rc=%d\n", rc);
    }
}
```

> `ble_gap_connect` 签名：`ble_gap_connect(own_addr_type, peer_addr, duration_ms, conn_params, cb, cb_arg)`。`conn_params=NULL` 使用默认连接参数。

### 2. 连接成功后做服务发现

blecent 用 `peer.c` 提供的 `peer_disc_all` 完成全量服务发现。

```c
static void on_disc_complete(const struct peer *peer, int status, void *arg)
{
    if (status != 0) {
        ble_gap_terminate(peer->conn_handle, BLE_ERR_REM_USER_CONN_TERM);
        return;
    }
    /* 发现完成，开始读写订阅 */
    do_gatt_ops(peer);
}

case BLE_GAP_EVENT_CONNECT:
    if (event->connect.status == 0) {
        peer_add(event->connect.conn_handle);
        peer_disc_all(event->connect.conn_handle,
                      on_disc_complete, NULL);
    } else {
        scan();   /* 连接失败恢复扫描 */
    }
    return 0;
```

### 3. GATT 客户端操作

```c
static int on_read(uint16_t conn_handle, const struct ble_gatt_error *error,
                   struct ble_gatt_attr *attr, void *arg)
{
    if (error->status == 0) {
        /* attr->om 包含读到的值 */
    }
    return 0;
}

/* 读特征 */
const struct peer_chr *chr = peer_chr_find_uuid(peer,
        BLE_UUID16_DECLARE(SVC_UUID), BLE_UUID16_DECLARE(CHR_UUID));
ble_gattc_read(peer->conn_handle, chr->chr.val_handle, on_read, NULL);

/* 写特征（flat 简易接口） */
uint8_t value[2] = { 99, 100 };
ble_gattc_write_flat(peer->conn_handle, chr->chr.val_handle,
                     value, sizeof value, on_write, NULL);

/* 订阅通知：写 CCCD（1,0 = enable notify） */
const struct peer_dsc *dsc = peer_dsc_find_uuid(peer,
        BLE_UUID16_DECLARE(SVC_UUID), BLE_UUID16_DECLARE(CHR_UUID),
        BLE_UUID16_DECLARE(BLE_GATT_DSC_CLT_CFG_UUID16));
uint8_t cccd[2] = { 1, 0 };
ble_gattc_write_flat(peer->conn_handle, dsc->dsc.handle,
                     cccd, sizeof cccd, on_subscribe, NULL);
```

### GATT 客户端关键 API

| 函数 | 说明 |
|---|---|
| `ble_gattc_disc_all_svcs(conn_handle, cb, cb_arg)` | 发现所有服务 |
| `ble_gattc_disc_svc_by_uuid(conn_handle, uuid, cb, cb_arg)` | 按 UUID 发现服务 |
| `ble_gattc_disc_all_chrs(conn_handle, start, end, cb, cb_arg)` | 发现所有特征 |
| `ble_gattc_disc_chrs_by_uuid(conn_handle, start, end, uuid, cb, cb_arg)` | 按 UUID 发现特征 |
| `ble_gattc_disc_all_dscs(conn_handle, start, end, cb, cb_arg)` | 发现所有描述符 |
| `ble_gattc_read(conn_handle, attr_handle, cb, cb_arg)` | 读特征值 |
| `ble_gattc_read_by_uuid(conn_handle, start, end, uuid, cb, cb_arg)` | 按 UUID 读 |
| `ble_gattc_write(conn_handle, attr_handle, txom, cb, cb_arg)` | 写（mbuf） |
| `ble_gattc_write_flat(conn_handle, attr_handle, data, data_len, cb, cb_arg)` | 写（裸数据，最常用） |
| `ble_gattc_write_no_rsp_flat(conn_handle, attr_handle, data, data_len)` | 无响应写 |
| `ble_gattc_exchange_mtu(conn_handle, cb, cb_arg)` | 协商 MTU |

> 客户端回调签名统一：`int (*cb)(uint16_t conn_handle, const struct ble_gatt_error *error, struct ble_gatt_attr *attr, void *arg)`。

## 常见错误

| 错误 | 原因 | 解决方法 |
|---|---|---|
| `ble_gap_connect` 返回 `BLE_HS_EBUSY` | 正在扫描 | 先 `ble_gap_disc_cancel()` |
| 服务发现无结果 | 对端未注册 GATT 服务 | 检查外设端 `ble_gatts_add_svcs` |
| 读 / 写回调 `error->status != 0` | 句柄错误 / 权限不足 | 用 `peer_chr_find_uuid` 取真实 val_handle |
| 订阅后收不到 notify | 未写 CCCD 或写错值 | CCCD 写 `{1,0}`（notify）或 `{2,0}`（indicate） |
| 连接参数不合适 | 用了默认 params | 传入自定义 `ble_gap_conn_params` |

## 参考

- `apps/blecent/src/main.c` — 扫描、取消、连接、服务发现、读写订阅完整链路
- `apps/blecent/src/peer.c` — peer 管理与 `peer_disc_all` / `peer_chr_find_uuid` / `peer_dsc_find_uuid`
- 头文件：`nimble/host/include/host/ble_gatt.h`（`ble_gattc_*`）、`ble_gap.h`
