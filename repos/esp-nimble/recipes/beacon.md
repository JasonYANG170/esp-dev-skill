# 不可连接广播（Beacon）

> **适用摘要**: 实现不可连接的广播者（Broadcaster）：non-connectable advertising，常用于 Beacon / 厂商自定义数据广播，使用 NRPA 作为本机地址。

## 触发意图

- "Beacon / 信标"
- "不可连接广播"
- "iBeacon / Eddystone"
- "manufacturer data / 厂商数据"
- "non-connectable advertising"

## 前置条件

| 条件 | 要求 |
|---|---|
| 参考示例 | `apps/advertiser/src/main.c` |
| 配置 | `CONFIG_BT_NIMBLE_ROLE_BROADCASTER=y` |
| 时序 | 在 `on_sync` 中调用 |

## 分步说明

### 1. 生成 NRPA 并设置

不可连接广播常使用一次性 NRPA。生成后必须用 `BLE_OWN_ADDR_RANDOM`。

```c
#include "host/ble_hs.h"
#include "host/util/util.h"
#include "services/gap/ble_svc_gap.h"

static const char *device_name = "NimbleBeacon";
static int adv_event(struct ble_gap_event *event, void *arg);

static void set_ble_addr(void)
{
    ble_addr_t addr;
    int rc = ble_hs_id_gen_rnd(1, &addr);   /* nrpa=1 */
    assert(rc == 0);
    rc = ble_hs_id_set_rnd(addr.val);
    assert(rc == 0);
}
```

### 2. 启动不可连接广播

```c
static void advertise(void)
{
    struct ble_gap_adv_params adv_params;
    struct ble_hs_adv_fields fields;
    /* 厂商自定义数据示例 */
    static const uint8_t mfg_data[] = { 0x5C, 0x03, 0xAA, 0xBB, 0xCC };
    int rc;

    memset(&adv_params, 0, sizeof adv_params);
    adv_params.conn_mode = BLE_GAP_CONN_MODE_NON;   /* 不可连接 */
    adv_params.disc_mode = BLE_GAP_DISC_MODE_GEN;

    memset(&fields, 0, sizeof fields);
    fields.flags = BLE_HS_ADV_F_DISC_GEN;
    fields.tx_pwr_lvl_is_present = 1;
    fields.tx_pwr_lvl = BLE_HS_ADV_TX_PWR_LVL_AUTO;
    fields.name = (uint8_t *)device_name;
    fields.name_len = strlen(device_name);
    fields.name_is_complete = 1;

    /* 厂商数据：前两字节为 company id（小端） */
    fields.mfg_data = (uint8_t *)mfg_data;
    fields.mfg_data_len = sizeof mfg_data;

    rc = ble_gap_adv_set_fields(&fields);
    assert(rc == 0);

    /* duration 10000ms；NRPA → own_addr_type = BLE_OWN_ADDR_RANDOM */
    rc = ble_gap_adv_start(BLE_OWN_ADDR_RANDOM, NULL, 10000,
                           &adv_params, adv_event, NULL);
    assert(rc == 0);
}
```

### 3. 广播结束回调

`duration_ms` 用尽后收到 `BLE_GAP_EVENT_ADV_COMPLETE`，可在其中重启以持续广播。

```c
static int adv_event(struct ble_gap_event *event, void *arg)
{
    switch (event->type) {
    case BLE_GAP_EVENT_ADV_COMPLETE:
        MODLOG_DFLT(INFO, "adv complete; reason=%d\n",
                    event->adv_complete.reason);
        advertise();   /* 重启广播 */
        return 0;
    default:
        return 0;
    }
}
```

### 4. sync 中触发

```c
static void on_sync(void)
{
    set_ble_addr();
    advertise();
}
```

### ble_hs_adv_fields 常用字段

| 字段 | 含义 |
|---|---|
| `flags` | 广播 flags（DISC_GEN / DISC_LTD / BREDR_UNSUP） |
| `name` / `name_len` / `name_is_complete` | 设备名 |
| `tx_pwr_lvl` / `tx_pwr_lvl_is_present` | 发射功率（`BLE_HS_ADV_TX_PWR_LVL_AUTO` 自动填充） |
| `uuids16` / `num_uuids16` / `uuids16_is_complete` | 16-bit 服务 UUID 列表 |
| `uuids128` / `num_uuids128` / `uuids128_is_complete` | 128-bit 服务 UUID 列表 |
| `mfg_data` / `mfg_data_len` | 厂商自定义数据 |
| `appearance` / `appearance_is_present` | 外观 |

> 参考 `apps/scanner/src/main.c` 的 `print_adv_fields` 查看全部可解析字段。

## 常见错误

| 错误 | 原因 | 解决方法 |
|---|---|---|
| 广播只发一次 | duration 到期未重启 | 在 ADV_COMPLETE 中再次 `advertise()`，或用 `BLE_HS_FOREVER` |
| own_addr_type 用 PUBLIC 但设了 NRPA | 类型不匹配 | NRPA 一律用 `BLE_OWN_ADDR_RANDOM` |
| 厂商数据解析错乱 | company id 字节序错误 | 前两字节按小端填写 |
| adv 数据超 31 字节 | 字段过多 | 使用扩展广播（见 `ext_adv.md`） |

## 参考

- `apps/advertiser/src/main.c` — 不可连接广播 + NRPA
- `apps/scanner/src/main.c` — `print_adv_fields` 全字段解析
- `docs/ble_setup/ble_addr.rst` — 地址配置
- 头文件：`nimble/host/include/host/ble_hs_adv.h`
