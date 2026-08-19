# 连接参数更新

> **适用摘要**: 连接建立后协商更合适的连接参数：从机用 L2CAP Connection Parameter Update Procedure 请求、主机用 Link-Layer Connection Parameters Request Procedure 下发，以及处理对端的更新请求回调（accept / reject）与最终的 `BLE_GAP_EVENT_CONN_UPDATE` 结果。

> Version: selected repo/component version; align with the user project and dependency manifest.
> Evidence: `repos/esp-nimble/resources/`, source/examples in `repos/esp-nimble/`, and this recipe path `repos/esp-nimble/recipes/conn_param_update.md`.
> Validation: draft metadata added from repository routing; verify APIs, Kconfig symbols, and component dependencies against the selected version.

## 触发意图

- "连接参数更新 / connection parameter update"
- "interval / latency / supervision timeout"
- "conn-update / l2cap-update"
- "ble_gap_update_params"
- "BLE_GAP_EVENT_CONN_UPDATE / L2CAP_UPDATE_REQ / CONN_UPDATE_REQ"
- "吞吐太低 / 延迟太大 / 连接超时断开"

## 前置条件

| 条件 | 要求 |
|---|---|
| 参考示例 | `apps/bleprph/src/main.c`（外设处理 `BLE_GAP_EVENT_CONN_UPDATE`）、`apps/blecent/src/main.c`（中心默认采用 `ble_gap_connect` 默认参数） |
| 文档 | `docs/btshell/btshell_GAP.rst`（conn-update-params 与 l2cap-update 两个过程）、`docs/ble_hs/ble_gap.rst`（GAP 负责 "connection updating operations"） |
| 配置 | `CONFIG_BT_NIMBLE_ROLE_PERIPHERAL=y` 或 `CONFIG_BT_NIMBLE_ROLE_CENTRAL=y` |
| 时序 | 连接已建立（已拿到 `conn_handle`） |

## 背景：两个参数更新过程

NimBLE 通过 GAP 暴露连接参数更新能力。根据规范，从机（peripheral）不能直接发 LL CP，需走 L2CAP Connection Parameter Update Procedure；主机（central）走 Link-Layer Connection Parameters Request Procedure。btshell 把两者拆成两个命令：

- **`conn-update-params`**（btshell_GAP.rst:228）—— 走 Link-Layer 过程（主机侧），参数：`conn` / `interval_min`（默认 30）/ `interval_max`（默认 50）/ `latency`（默认 0）/ `timeout`（默认 0x0100）/ `min_conn_event_len`（默认 0x0010）/ `max_conn_event_len`（默认 0x0300）。
- **`l2cap-update`**（btshell_GAP.rst:272）—— 走 L2CAP 过程（从机侧），参数：`interval_min` / `interval_max` / `latency` / `timeout`。

对应的 Host API 都是同一个 `ble_gap_update_params`，Host 会根据本端角色选择正确的过程。

### 参数单位（来自 `struct ble_gap_upd_params`）

| 字段 | 单位 | btshell 默认值 |
|---|---|---|
| `itvl_min` / `itvl_max` | 1.25 ms | 30 / 50（即 37.5 ms ~ 62.5 ms） |
| `latency` | 连接事件数（无符号） | 0 |
| `supervision_timeout` | 10 ms | 0x0100（即 2560 ms） |
| `min_ce_len` / `max_ce_len` | 0.625 ms | 0x0010 / 0x0300 |

## 分步说明

### 1. 主动发起参数更新（任意角色）

连接建立后调用 `ble_gap_update_params`。返回后不会立即生效——Host 异步完成 L2CAP/LL 过程，结果以 `BLE_GAP_EVENT_CONN_UPDATE` 事件送达。

```c
#include "host/ble_hs.h"
#include "host/ble_gap.h"

static uint16_t conn_handle = BLE_HS_CONN_HANDLE_NONE;

static int gap_event(struct ble_gap_event *event, void *arg);

/* 例：把外设切到低延迟高速档（更小 interval）。*/
static void request_fast_params(void)
{
    struct ble_gap_upd_params params = {
        .itvl_min     = 0x0006,   /* 7.5 ms  （BLE 规范允许的最小值） */
        .itvl_max     = 0x0010,   /* 20 ms   */
        .latency      = 0,        /* 不跳事件 */
        .supervision_timeout = 0x00c8, /* 2000 ms */
        .min_ce_len   = 0,        /* 0 表示让 controller 自选连接事件长度 */
        .max_ce_len   = 0,
    };
    int rc = ble_gap_update_params(conn_handle, &params);
    if (rc != 0) {
        MODLOG_DFLT(ERROR, "update_params failed; rc=%d\n", rc);
    }
}
```

> `ble_gap_update_params` 返回码：`0` 成功；`BLE_HS_ENOTCONN` 无此连接；`BLE_HS_EALREADY` 已有更新过程在跑；`BLE_HS_EINVAL` 参数非法（见头文件 `ble_gap.h`）。

### 2. 处理更新结果事件（外设，对照 bleprph:206）

bleprph 在 `BLE_GAP_EVENT_CONN_UPDATE` 中调�� `ble_gap_conn_find` 读取最新参数并打印。无论本端是发起方还是接收方，最终结果都从这里拿。

```c
case BLE_GAP_EVENT_CONN_UPDATE:
    MODLOG_DFLT(INFO, "connection updated; status=%d ",
                event->conn_update.status);
    if (event->conn_update.status == 0) {
        struct ble_gap_conn_desc desc;
        int rc = ble_gap_conn_find(event->conn_update.conn_handle, &desc);
        assert(rc == 0);
        MODLOG_DFLT(INFO, "itvl=%d latency=%d timeout=%d\n",
                    desc.conn_itvl, desc.conn_latency,
                    desc.supervision_timeout);
    }
    return 0;
```

> `event->conn_update` 字段：`int status`（0=成功）、`uint16_t conn_handle`。最新的 interval / latency / timeout 要用 `ble_gap_conn_find` 从 `struct ble_gap_conn_desc` 读取（`conn_itvl` / `conn_latency` / `supervision_timeout`）。

### 3. 应答对端的更新请求（accept / reject）

当对端发起参数更新（L2CAP 或 LL），Host 触发 `BLE_GAP_EVENT_L2CAP_UPDATE_REQ` 或 `BLE_GAP_EVENT_CONN_UPDATE_REQ`，回调返回 0 表示接受、返回非零 HCI 错误码表示拒绝。bleprph 默认不拦截（直接返回 0），如需限制可自行校验。

```c
case BLE_GAP_EVENT_L2CAP_UPDATE_REQ:
case BLE_GAP_EVENT_CONN_UPDATE_REQ: {
    const struct ble_gap_upd_params *peer = event->conn_update_req.peer_params;
    /* 拒绝过大的 supervision timeout，避免链路挂死检测过慢 */
    if (peer->supervision_timeout > 0x0c80) {   /* > 32 s */
        return BLE_ERR_REM_USER_CONN_TERM;       /* 返回非 0 即拒绝 */
    }
    return 0;   /* 接受（self_params 默认拷自 peer_params） */
}
```

> `event->conn_update_req` 字段：`const struct ble_gap_upd_params *peer_params`（对端期望值）、`struct ble_gap_upd_params *self_params`（本端回写值，默认已拷自 peer_params）、`uint16_t conn_handle`。

### 4. 在连接时就下发自定义参数（中心，对照 blecent）

blecent 用 `ble_gap_connect(..., NULL, ...)` 用默认参数；如果中心希望连接一建立就用自己的参数，把第 4 个参数从 `NULL` 换成 `struct ble_gap_conn_params *`（注意：`ble_gap_connect` 用的是 `ble_gap_conn_params`，而 `ble_gap_update_params` 用的是 `ble_gap_upd_params`，两者字段名相同但属不同结构体）。

```c
struct ble_gap_conn_params conn_params = {
    .scan_itvl = 0x0010, .scan_window = 0x0010,
    .itvl_min = 0x0006, .itvl_max = 0x0010,
    .latency = 0, .supervision_timeout = 0x00c8,
    .min_ce_len = 0x0010, .max_ce_len = 0x0030,
};
ble_gap_connect(own_addr_type, &disc->addr, 30000,
                &conn_params, gap_event, NULL);
```

## 为什么默认参数常常不够

btshell 的默认 `interval_min=30 / interval_max=50`（37.5~62.5 ms）是"通用折中"档；不同应用应当主动调：

| 目标 | 建议 itvl_min~itvl_max | latency |
|---|---|---|
| 高吞吐 / 低延迟（HID、音频） | 6~12（7.5~15 ms） | 0 |
| 低功耗传感器（少发数据） | 60~240（75~300 ms） | 4~10 |
| 平衡 | 24~40（30~50 ms） | 0 |

## 常见错误

| 错误 | 原因 | 解决方法 |
|---|---|---|
| `ble_gap_update_params` 返回 `BLE_HS_EINVAL` | `itvl_min > itvl_max`，或 `supervision_timeout` 与 interval 不匹配（规范要求 `supervision_timeout > (1+latency) * itvl_max * 2`） | 校验三者的规范约束再传 |
| `ble_gap_update_params` 返回 `BLE_HS_EALREADY` | 上一次更新过程尚未结束 | 等 `BLE_GAP_EVENT_CONN_UPDATE` 到来后再发 |
| 返回 `BLE_HS_ENOTCONN` | `conn_handle` 已失效或未保存 | 在 `BLE_GAP_EVENT_CONNECT` 里保存，`DISCONNECT` 里复位 |
| 更新事件 status 非 0 | 对端拒绝（外设发 L2CAP 时中心回了 reject） | 在中心侧 `L2CAP_UPDATE_REQ` 回调放宽阈值 |
| 外设更新一直不生效 | 外设试图直接走 LL CP——规范要求外设走 L2CAP | 由 Host 自动选择，但需确保对端中心支持 L2CAP CP Update |
| 串口 / 通知速率上不去 | interval 太大 | 主动 `ble_gap_update_params` 收到更小 interval |

## 参考

- `apps/bleprph/src/main.c` — `BLE_GAP_EVENT_CONN_UPDATE` 处理（line 206），`ble_gap_conn_find` 后打印 conn_itvl/latency/timeout。
- `apps/blecent/src/main.c` — 中心用 `ble_gap_connect` 默认参数连接。
- `docs/btshell/btshell_GAP.rst` — `conn-update-params`（line 228，LL 过程）与 `l2cap-update`（line 272，L2CAP 过程）两套参数表。
- `docs/ble_hs/ble_gap.rst` — GAP 负责 "connection updating operations"。
- 头文件：`nimble/host/include/host/ble_gap.h`（`ble_gap_update_params`、`struct ble_gap_upd_params`、`BLE_GAP_EVENT_CONN_UPDATE/_CONN_UPDATE_REQ/_L2CAP_UPDATE_REQ`、`event->conn_update` / `event->conn_update_req`）。
