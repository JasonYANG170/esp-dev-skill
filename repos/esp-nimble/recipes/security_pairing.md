# SMP 配对与加密

> **适用摘要**: 实现 BLE 安全管理：响应 `BLE_GAP_EVENT_PASSKEY_ACTION`、通过 `ble_sm_inject_io` 提供输入、发起配对、读取加密状态。

## 触发意图

- "配对 / pairing / 绑定 / bonding"
- "加密 / encryption"
- "Just Works / Passkey / 数值比较"
- "SMP / Security Manager"
- "BLE_GAP_EVENT_PASSKEY_ACTION"

## 前置条件

| 条件 | 要求 |
|---|---|
| 参考示例 | `apps/bleprph/src/main.c`（`BLE_GAP_EVENT_ENC_CHANGE`、`REPEAT_PAIRING`） |
| 配置 | `CONFIG_BT_NIMBLE_SMSC_ENABLE=y`（Secure Connections） |
| 文档 | `docs/ble_sec.rst` |

## 分步说明

### 1. 配置 Host 安全能力

```c
#include "host/ble_hs.h"
#include "host/ble_sm.h"

/* 在 nimble_port_init 之后 */
ble_hs_cfg.sm_io_cap = BLE_HS_IO_NO_INPUT_OUTPUT;  /* Just Works 场景 */
ble_hs_cfg.sm_bonding = 1;       /* 允许绑定 */
ble_hs_cfg.sm_mitm = 1;          /* 要求 MITM 保护 */
ble_hs_cfg.sm_sc = 1;            /* 启用 Secure Connections */
ble_hs_cfg.sm_our_key_dist = BLE_SM_PAIR_KEY_DIST_ENC |
                             BLE_SM_PAIR_KEY_DIST_ID;
ble_hs_cfg.sm_their_key_dist = BLE_SM_PAIR_KEY_DIST_ENC |
                               BLE_SM_PAIR_KEY_DIST_ID;
```

### 2. 发起配对 / 加密

```c
/* 主动发起配对（中心或外设均可） */
int rc = ble_gap_pair_initiate(conn_handle);

/* 仅发起加密（已有绑定 key） */
int rc = ble_gap_security_initiate(conn_handle);

/* 指定 key size 与 auth 要求发起加密 */
rc = ble_gap_encryption_initiate(conn_handle, key_size,
                                 0, 0, 0, 0);
```

### 3. 响应 PASSKEY_ACTION（关键）

Host 在需要用户输入 / 显示时触发该事件，应用必须调用 `ble_sm_inject_io`。

```c
static void handle_passkey(uint16_t conn_handle,
                           struct ble_gap_passkey_params *params)
{
    struct ble_sm_io io = {0};
    int rc;

    switch (params->action) {
    case BLE_SM_IOACT_INPUT:                 /* 输入 6 位 passkey */
        io.action = BLE_SM_IOACT_INPUT;
        io.passkey = get_user_passkey();     /* 应用实现 */
        break;
    case BLE_SM_IOACT_DISP:                  /* 显示 6 位 passkey */
        display_passkey(params->numcmp);     /* 显示给用户 */
        return;                              /* 显示无需注入 */
    case BLE_SM_IOACT_NUMCMP:                /* 数值比较：yes/no */
        io.action = BLE_SM_IOACT_NUMCMP;
        io.numcmp_accept = user_confirm();   /* 1=接受 */
        break;
    case BLE_SM_IOACT_NONE:                  /* Just Works：无需 IO */
        return;
    default:
        return;
    }
    rc = ble_sm_inject_io(conn_handle, &io);
    assert(rc == 0);
}

case BLE_GAP_EVENT_PASSKEY_ACTION:
    handle_passkey(event->passkey.conn_handle,
                   &event->passkey.params);
    return 0;
```

### 4. 处理加密变化与重复配对

```c
case BLE_GAP_EVENT_ENC_CHANGE: {
    struct ble_gap_conn_desc desc;
    ble_gap_conn_find(event->enc_change.conn_handle, &desc);
    MODLOG_DFLT(INFO, "enc=%d auth=%d bonded=%d\n",
                desc.sec_state.encrypted,
                desc.sec_state.authenticated,
                desc.sec_state.bonded);
    return 0;
}

case BLE_GAP_EVENT_REPEAT_PAIRING:
    /* 已有绑定但对端再次请求：删旧绑定后允许重试 */
    ble_gap_conn_find(event->repeat_pairing.conn_handle, &desc);
    ble_store_util_delete_peer(&desc.peer_id_addr);
    return BLE_GAP_REPEAT_PAIRING_RETRY;
```

### IO 能力常量（`ble_hs_cfg.sm_io_cap`）

| 常量 | 含义 |
|---|---|
| `BLE_HS_IO_DISPLAY_ONLY` | 仅显示 |
| `BLE_HS_IO_DISPLAY_YESNO` | 显示 + 是/否 |
| `BLE_HS_IO_KEYBOARD_ONLY` | 仅键盘 |
| `BLE_HS_IO_NO_INPUT_OUTPUT` | 无 IO（Just Works） |
| `BLE_HS_IO_KEYBOARD_DISPLAY` | 键盘 + 显示 |

### 关键 API

| 函数 | 说明 |
|---|---|
| `ble_gap_pair_initiate(conn_handle)` | 发起配对 |
| `ble_gap_security_initiate(conn_handle)` | 发起加密 |
| `ble_gap_encryption_initiate(conn_handle, key_size, ...)` | 带参数加密 |
| `ble_sm_inject_io(conn_handle, &io)` | 注入 IO 响应 |
| `ble_sm_sc_oob_generate_data(&oob)` | 生成 OOB 数据（SC） |
| `ble_sm_configure_static_passkey(passkey, enable)` | 配置静态 passkey |
| `ble_gap_unpair(&peer_addr)` | 解除某设备绑定 |
| `ble_gap_unpair_oldest_peer(void)` | 解除最旧绑定 |
| `ble_store_util_delete_peer(&addr)` | 删除某设备的所有存储数据 |

### `ble_gap_sec_state` 字段

| 字段 | 含义 |
|---|---|
| `encrypted` | 链路是否加密 |
| `authenticated` | 是否经过认证（MITM） |
| `bonded` | 是否已绑定 |
| `key_size` | 加密密钥长度 |

## 常见错误

| 错误 | 原因 | 解决方法 |
|---|---|---|
| 配对超时 | 未响应 PASSKEY_ACTION | 在回调中调用 `ble_sm_inject_io` |
| Just Works 不触发 PASSKEY | action 为 `BLE_SM_IOACT_NONE` | 无需注入，直接放行 |
| 加密状态读不到 | 未用 `ble_gap_conn_find` | 用 conn_find 取 `sec_state` |
| 重复配对失败 | 旧绑定冲突 | 处理 REPEAT_PAIRING：删旧 + RETRY |
| OOB 配对失败 | 未先 `ble_sm_sc_oob_generate_data` | 双方生成并交换 OOB 数据 |

## 参考

- `apps/bleprph/src/main.c` — `BLE_GAP_EVENT_ENC_CHANGE`、`REPEAT_PAIRING` 处理
- `docs/ble_sec.rst` — 安全模型（Pairing/Bonding/Authentication/Encryption）
- 头文件：`nimble/host/include/host/ble_sm.h`、`ble_gap.h`（`passkey`、`sec_state`）
