# 服务端通知 / 指示

> **适用摘要**: 在 GATT 服务端实现 NOTIFY / INDICATE 特征，通过 `ble_gatts_notify_custom` 发送通知数据，处理客户端的 CCCD 订阅事件。

## 触发意图

- "通知 / notify / indication"
- "推送数据到客户端"
- "CCCD 订阅"
- "ble_gatts_notify_custom"

## 前置条件

| 条件 | 要求 |
|---|---|
| 参考示例 | `apps/blehr/src/main.c`、`apps/blehr/src/gatt_svr.c` |
| 配置 | 已建立连接（外设角色） |
| 时序 | 客户端写 CCCD 后才能发送 notify |

## 分步说明

### 1. 定义带 NOTIFY 标志的特征

```c
#include "host/ble_hs.h"
#include "host/ble_uuid.h"

uint16_t hrs_hrm_handle;   /* 保存特征值句柄，由 val_handle 回填 */

static int hrs_access(uint16_t conn_handle, uint16_t attr_handle,
                      struct ble_gatt_access_ctxt *ctxt, void *arg);

static const struct ble_gatt_svc_def gatt_svr_svcs[] = {
    {
        .type = BLE_GATT_SVC_TYPE_PRIMARY,
        .uuid = BLE_UUID16_DECLARE(0x180D),                  /* Heart Rate */
        .characteristics = (struct ble_gatt_chr_def[]) { {
            .uuid = BLE_UUID16_DECLARE(0x2A37),              /* HR Measurement */
            .access_cb = hrs_access,
            .val_handle = &hrs_hrm_handle,                   /* 回填值句柄 */
            .flags = BLE_GATT_CHR_F_NOTIFY,                  /* 支持通知 */
        }, {
            0,
        } },
    },
    { 0, },
};
```

### 2. 处理订阅事件，记录订阅状态

客户端写 CCCD 时 Host 产生 `BLE_GAP_EVENT_SUBSCRIBE`。

```c
static bool notify_state;
static uint16_t conn_handle = BLE_HS_CONN_HANDLE_NONE;

static int gap_event(struct ble_gap_event *event, void *arg)
{
    switch (event->type) {
    case BLE_GAP_EVENT_CONNECT:
        if (event->connect.status == 0) {
            conn_handle = event->connect.conn_handle;
        }
        return 0;

    case BLE_GAP_EVENT_SUBSCRIBE:
        if (event->subscribe.attr_handle == hrs_hrm_handle) {
            notify_state = event->subscribe.cur_notify;   /* 1=已订阅 */
            if (notify_state) {
                start_notify_timer();   /* 开始周期推送 */
            }
        }
        return 0;
    }
    return 0;
}
```

### 3. 发送通知（数据必须为 mbuf）

```c
static void send_heart_rate(void)
{
    uint8_t hrm[2];
    struct os_mbuf *om;
    int rc;

    if (!notify_state) return;     /* 未订阅不发 */

    hrm[0] = 0x06;                 /* 传感器接触标志 */
    hrm[1] = 90;                   /* 心率 */

    /* 关键：用 ble_hs_mbuf_from_flat 转换为 mbuf */
    om = ble_hs_mbuf_from_flat(hrm, sizeof hrm);
    assert(om != NULL);

    rc = ble_gatts_notify_custom(conn_handle, hrs_hrm_handle, om);
    assert(rc == 0);
}
```

### 4. 指示（Indication）用法

把特征 flags 改为 `BLE_GATT_CHR_F_INDICATE`，发送用 `ble_gatts_indicate_custom`（或 `ble_gatts_indicate`）。指示需要客户端确认，订阅事件中用 `cur_indicate` 判断。

### 关键 API

| 函数 | 说明 |
|---|---|
| `ble_gatts_notify(conn_handle, chr_val_handle)` | 用现有值发送通知 |
| `ble_gatts_notify_custom(conn_handle, att_handle, om)` | 用自定义 mbuf 发送通知 |
| `ble_gatts_notify_multiple_custom(conn_handle, num, om)` | 多特征值合并通知 |
| `ble_gatts_indicate(conn_handle, chr_val_handle)` | 指示（带确认） |
| `ble_gatts_indicate_custom(conn_handle, chr_val_handle, om)` | 自定义 mbuf 指示 |
| `ble_hs_mbuf_from_flat(const void *buf, int len)` | 分配并填充 mbuf |
| `ble_gap_event.subscribe.{cur_notify,cur_indicate}` | 当前订阅状态 |

## 常见错误

| 错误 | 原因 | 解决方法 |
|---|---|---|
| 通知数据类型不匹配 | 未转 mbuf | 用 `ble_hs_mbuf_from_flat` 转换 |
| 客户端收不到 | 未订阅 CCCD / 特征无 NOTIFY 标志 | 确认 `BLE_GATT_CHR_F_NOTIFY` 且客户端已写 CCCD |
| `hrs_hrm_handle` 为 0 | 未设置 `val_handle` 回填 | 在特征定义中填 `.val_handle = &hrs_hrm_handle` |
| 加密需求未满足 | 特征带 `BLE_GATT_CHR_F_NOTIFY_INDICATE_ENC` | 先完成配对加密（见 `security_pairing.md`） |

## 参考

- `apps/blehr/src/gatt_svr.c` — HR Measurement NOTIFY 特征定义
- `apps/blehr/src/main.c` — `ble_gatts_notify_custom` + os_callout 定时推送
- 头文件：`nimble/host/include/host/ble_gatt.h`、`ble_hs.h`（`ble_hs_mbuf_from_flat`）
