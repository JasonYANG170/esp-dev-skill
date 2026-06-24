# LE PHY 选择（1M / 2M / Coded 远距离）

> **适用摘要**: 在已建立的 BLE 连接上切换 LE PHY——1 Mbps 兼容、2 Mbps 高吞吐、Coded PHY（S=2 / S=8）远距离；包含设置默认偏好、按连接设置偏好、读取当前 PHY，以及处理 `BLE_GAP_EVENT_PHY_UPDATE_COMPLETE` 结果事件。

## 触发意图

- "PHY / 物理通道 / physical channel"
- "2M / 2M PHY / 高吞吐"
- "Coded PHY / Long Range / 远距离 / S=2 / S=8"
- "ble_gap_set_prefered_le_phy / ble_gap_set_prefered_default_le_phy"
- "BLE_GAP_EVENT_PHY_UPDATE_COMPLETE"
- "phy-set / phy-set-default（btshell）"

## 前置条件

| 条件 | 要求 |
|---|---|
| 参考示例 | `apps/bleprph/src/phy.c`（按键切换 PHY + 读 PHY + LED 指示）、`apps/bleprph/src/main.c`（`phy_conn_changed` / `BLE_GAP_EVENT_PHY_UPDATE_COMPLETE` 集成） |
| 文档 | `docs/btshell/btshell_GAP.rst`（`phy-set` line 252、`phy-set-default` line 262、`phy-read` line 268），`docs/index.rst`（NimBLE 支持 "2Msym/s PHY for higher throughput" 与 "Coded PHY for LE Long Range"） |
| 配置 | 控制器须支持对应 PHY（ESP32-C3/S3/H2 等 BT 5.x 芯片支持 2M+Coded；ESP32 经典版仅 1M） |
| 时序 | 设置默认偏好须在 sync 后、连接前；按连接偏好与读取须在连接已建立时 |

## 背景：bit-mask 与 phy_opts 编码（btshell_GAP.rst:252-267）

PHY 选择用 **位掩码**（可同时偏好多个 PHY，让 controller 选），而不是单个枚举值：

| 掩码宏（`ble_gap.h`） | 值 | 含义 |
|---|---|---|
| `BLE_GAP_LE_PHY_1M_MASK` | 0x01 | 偏好 1M PHY |
| `BLE_GAP_LE_PHY_2M_MASK` | 0x02 | 偏好 2M PHY |
| `BLE_GAP_LE_PHY_CODED_MASK` | 0x04 | 偏好 Coded PHY |
| `BLE_GAP_LE_PHY_ANY_MASK` | 0x0F | 无偏好（全部允许） |

读取 / 事件里返回的是单一 PHY 枚举：

| 枚举（`ble_gap.h`） | 值 |
|---|---|
| `BLE_GAP_LE_PHY_1M` | 1 |
| `BLE_GAP_LE_PHY_2M` | 2 |
| `BLE_GAP_LE_PHY_CODED` | 3 |

Coded PHY 还需指定编码选项（`phy_opts`）：

| 选项宏（`ble_gap.h`） | 值 | 含义 |
|---|---|---|
| `BLE_GAP_LE_PHY_CODED_ANY` | 0 | 不指定（controller 自选 S=2 或 S=8） |
| `BLE_GAP_LE_PHY_CODED_S2` | 1 | 偏好 S=2（500 kbps，中等距离） |
| `BLE_GAP_LE_PHY_CODED_S8` | 2 | 偏好 S=8（125 kbps，最远距离） |

## 分步说明

### 1. 设置默认 PHY 偏好（连接前）

在 sync 后、建立连接前调用 `ble_gap_set_prefered_default_le_phy`，新连接自动套用。对应 btshell 的 `phy-set-default`。

```c
#include "host/ble_gap.h"

/* 默认偏好 2M PHY（高速档）；mask=0 表示无偏好，可同时设多个 */
static void set_default_phy(void)
{
    int rc = ble_gap_set_prefered_default_le_phy(
        BLE_GAP_LE_PHY_1M_MASK | BLE_GAP_LE_PHY_2M_MASK,   /* tx_phys_mask */
        BLE_GAP_LE_PHY_1M_MASK | BLE_GAP_LE_PHY_2M_MASK);  /* rx_phys_mask */
    if (rc != 0) {
        MODLOG_DFLT(ERROR, "set_prefered_default_le_phy; rc=%d\n", rc);
    }
}
```

> 当 mask=0（无偏好）时，`phy_opts` 参数被忽略。`phy_opts` 仅在选 Coded 时有意义。

### 2. 在已建立的连接上切换 PHY（对应 btshell `phy-set`）

连接建立后，按用户/事件触发调用 `ble_gap_set_prefered_le_phy`。这是 bleprph `phy.c` 在按键中断里调用的 API。

```c
static uint16_t conn_handle = BLE_HS_CONN_HANDLE_NONE;

/* 切到 Coded PHY S=8（最远距离）。tx/rx mask 通常设为一致 */
static void request_coded_s8(void)
{
    if (conn_handle == BLE_HS_CONN_HANDLE_NONE) return;
    int rc = ble_gap_set_prefered_le_phy(conn_handle,
            BLE_GAP_LE_PHY_CODED_MASK,   /* tx_phys_mask */
            BLE_GAP_LE_PHY_CODED_MASK,   /* rx_phys_mask */
            BLE_GAP_LE_PHY_CODED_S8);    /* phy_opts */
    if (rc != 0) {
        MODLOG_DFLT(ERROR, "set_prefered_le_phy; rc=%d\n", rc);
    }
}
```

bleprph 的 phy.c 把四个按键分别映射到不同 PHY（`phy.c:85-92`）：

| 按键 | mask | phy_opts |
|---|---|---|
| button 0 | `BLE_GAP_LE_PHY_1M_MASK` | `BLE_GAP_LE_PHY_CODED_ANY` |
| button 1 | `BLE_GAP_LE_PHY_2M_MASK` | `BLE_GAP_LE_PHY_CODED_ANY` |
| button 2 | `BLE_GAP_LE_PHY_CODED_MASK` | `BLE_GAP_LE_PHY_CODED_S2` |
| button 3 | `BLE_GAP_LE_PHY_CODED_MASK` | `BLE_GAP_LE_PHY_CODED_S8` |

### 3. 读取当前 PHY

`ble_gap_read_le_phy` 同步返回当前 tx/rx PHY（对应 btshell `phy-read`）。bleprph 在 `phy_conn_changed`（`phy.c:100`）连接建立后立即读取：

```c
uint8_t tx_phy, rx_phy;
int rc = ble_gap_read_le_phy(conn_handle, &tx_phy, &rx_phy);
if (rc == 0) {
    /* tx_phy / rx_phy ∈ {BLE_GAP_LE_PHY_1M, _2M, _CODED} */
}
```

### 4. 处理 PHY 更新完成事件

切换完成后 Host 触发 `BLE_GAP_EVENT_PHY_UPDATE_COMPLETE`。bleprph 在 main.c:268 处理：

```c
case BLE_GAP_EVENT_PHY_UPDATE_COMPLETE:
    MODLOG_DFLT(INFO, "phy updated; status=%d tx=%d rx=%d\n",
                event->phy_updated.status,
                event->phy_updated.tx_phy,
                event->phy_updated.rx_phy);
    return 0;
```

> `event->phy_updated` 字段：`int status`（0=成功）、`uint16_t conn_handle`、`uint8_t tx_phy`、`uint8_t rx_phy`（后者取值 `BLE_GAP_LE_PHY_1M/2M/CODED`）。bleprph 假设对称 PHY，只取 `tx_phy` 点亮对应 LED。

## 常见错误

| 错误 | 原因 | 解决方法 |
|---|---|---|
| mask 写成裸数字 1/2/3 | 把枚举值当掩码用（1=1M 枚举，但 1M mask 是 0x01） | 用 `BLE_GAP_LE_PHY_*_MASK` 宏，而非 `BLE_GAP_LE_PHY_*` 枚举 |
| `phy-set` 选 2M 后实际仍是 1M | 对端控制器不支持 2M PHY，或某侧仍偏好 1M | 双方都设 mask 含 2M；ESP32 经典版仅 1M |
| Coded PHY 选了但 `phy_opts` 没传 | mask 含 Coded 时必须给有效的 `phy_opts` | 传 `BLE_GAP_LE_PHY_CODED_ANY/S2/S8` |
| mask=0（无偏好）但 `phy_opts` 非零 | 无偏好时 `phy_opts` 被忽略 | mask=0 时 `phy_opts` 设 `BLE_GAP_LE_PHY_CODED_ANY` |
| 切 PHY 后吞吐没变 | 同时受连接 interval / Data Length 限制 | 配合 `ble_gap_update_params`（见 conn_param_update.md）与 `ble_gap_set_data_len` |
| 一直收不到 `PHY_UPDATE_COMPLETE` | 未在 GAP 事件回调里 case 该事件 | 加 `case BLE_GAP_EVENT_PHY_UPDATE_COMPLETE` 分支 |

## 参考

- `apps/bleprph/src/phy.c` — 完整 PHY 切换：按键映射 mask+opts（`phy.c:85`）、`ble_gap_set_prefered_le_phy`（`phy.c:59`）、连接建立后 `ble_gap_read_le_phy`（`phy.c:108`）、LED 指示当前 PHY（`phy.c:115`）。
- `apps/bleprph/src/main.c` — `BLE_GAP_EVENT_PHY_UPDATE_COMPLETE`（main.c:268）与 `BLE_GAP_EVENT_CONNECT/DISCONNECT` 调用 `phy_conn_changed`（main.c:182 / 199）。
- `docs/btshell/btshell_GAP.rst` — `phy-set`（line 252，含 tx_phys_mask/rx_phys_mask/phy_opts 编码表）、`phy-set-default`（line 262）、`phy-read`（line 268）。
- `docs/index.rst` — NimBLE 特性："2Msym/s PHY for higher throughput"（line 39）、"Coded PHY for LE Long Range"（line 40）。
- 头文件：`nimble/host/include/host/ble_gap.h`（`ble_gap_set_prefered_default_le_phy` / `ble_gap_set_prefered_le_phy` / `ble_gap_read_le_phy`、`BLE_GAP_LE_PHY_*` / `*_MASK` / `BLE_GAP_LE_PHY_CODED_*`、`event->phy_updated`、`BLE_GAP_EVENT_PHY_UPDATE_COMPLETE`）。
