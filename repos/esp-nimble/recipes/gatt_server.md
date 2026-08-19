# 自定义 GATT 服务

> **适用摘要**: 用 NimBLE 的静态服务表 `struct ble_gatt_svc_def` 定义自定义 GATT 服务与特征，实现 access_cb 读写回调，并通过 count + add 注册。

> Version: selected repo/component version; align with the user project and dependency manifest.
> Evidence: `repos/esp-nimble/resources/`, source/examples in `repos/esp-nimble/`, and this recipe path `repos/esp-nimble/recipes/gatt_server.md`.
> Validation: draft metadata added from repository routing; verify APIs, Kconfig symbols, and component dependencies against the selected version.

## 触发意图

- "自定义 GATT 服务"
- "GATT 服务端 / characteristic"
- "服务表 / ble_gatt_svc_def"
- "access_cb 读写"
- "ble_gatts_add_svcs"

## 前置条件

| 条件 | 要求 |
|---|---|
| 参考示例 | `apps/blehr/src/gatt_svr.c` |
| 时序 | `gatt_svr_init()` 在 `nimble_port_freertos_init` 之前 |

## 分步说明

### 1. 定义服务 / 特征 / 描述符表

服务表是三层嵌套的静态数组，每层必须以 `{ 0 }` 结尾。

```c
#include "host/ble_hs.h"
#include "host/ble_uuid.h"

static uint16_t my_chr_val_handle;

static int my_chr_access(uint16_t conn_handle, uint16_t attr_handle,
                         struct ble_gatt_access_ctxt *ctxt, void *arg);

static const struct ble_gatt_svc_def gatt_svr_svcs[] = {
    {
        .type = BLE_GATT_SVC_TYPE_PRIMARY,
        .uuid = BLE_UUID16_DECLARE(0x180D),     /* 自定义/标准服务 UUID */
        .characteristics = (struct ble_gatt_chr_def[]) { {
            .uuid = BLE_UUID16_DECLARE(0x2A37),
            .access_cb = my_chr_access,
            .val_handle = &my_chr_val_handle,
            .flags = BLE_GATT_CHR_F_READ |
                     BLE_GATT_CHR_F_WRITE |
                     BLE_GATT_CHR_F_NOTIFY,
            .descriptors = (struct ble_gatt_dsc_def[]) { {
                .uuid = BLE_UUID16_DECLARE(0x2901),   /* User Description */
                .att_flags = BLE_ATT_F_READ,
                .access_cb = dsc_access,
            }, {
                0,
            } },
        }, {
            0,    /* 特征列表结尾 */
        } },
    },
    {
        0,        /* 服务列表结尾 */
    },
};
```

### 2. 实现 access_cb

通过 `ctxt->op` 区分读写、特征/描述符；读返回值用 `os_mbuf_append(ctxt->om, ...)`。

```c
static int my_chr_access(uint16_t conn_handle, uint16_t attr_handle,
                         struct ble_gatt_access_ctxt *ctxt, void *arg)
{
    uint8_t buf[16];
    int rc;

    switch (ctxt->op) {
    case BLE_GATT_ACCESS_OP_READ_CHR: {
        uint16_t uuid = ble_uuid_u16(ctxt->chr->uuid);
        /* 准备读返回数据 */
        size_t len = prepare_read_payload(uuid, buf, sizeof buf);
        rc = os_mbuf_append(ctxt->om, buf, len);
        return rc == 0 ? 0 : BLE_ATT_ERR_INSUFFICIENT_RES;
    }

    case BLE_GATT_ACCESS_OP_WRITE_CHR: {
        /* 从 ctxt->om 读取写入数据 */
        uint16_t datalen = OS_MBUF_PKTLEN(ctxt->om);
        rc = ble_hs_mbuf_to_flat(ctxt->om, buf, sizeof buf, &datalen);
        if (rc != 0) return BLE_ATT_ERR_UNLIKELY;
        handle_write(uuid_from_chr(ctxt->chr), buf, datalen);
        return 0;
    }

    case BLE_GATT_ACCESS_OP_READ_DSC:
    case BLE_GATT_ACCESS_OP_WRITE_DSC:
        /* 描述符读写，用 ctxt->dsc->uuid */
        break;
    }
    return BLE_ATT_ERR_UNLIKELY;
}
```

### 3. count + add 注册（sync 前）

```c
int gatt_svr_init(void)
{
    int rc;
    rc = ble_gatts_count_cfg(gatt_svr_svcs);   /* 预计算 ATT 句柄 */
    if (rc != 0) return rc;
    rc = ble_gatts_add_svcs(gatt_svr_svcs);    /* 加入待注册队列 */
    return rc;
}
```

Host 在 sync 时会自动 `ble_gatts_start()`。

### 4. 注册回调（可选，用于调试句柄分配）

```c
void gatt_svr_register_cb(struct ble_gatt_register_ctxt *ctxt, void *arg)
{
    char buf[BLE_UUID_STR_LEN];
    switch (ctxt->op) {
    case BLE_GATT_REGISTER_OP_SVC:
        MODLOG_DFLT(DEBUG, "registered svc %s handle=%d\n",
                    ble_uuid_to_str(ctxt->svc.svc_def->uuid, buf),
                    ctxt->svc.handle);
        break;
    case BLE_GATT_REGISTER_OP_CHR:
        MODLOG_DFLT(DEBUG, "registered chr %s val_handle=%d\n",
                    ble_uuid_to_str(ctxt->chr.chr_def->uuid, buf),
                    ctxt->chr.val_handle);
        break;
    default:
        break;
    }
}
```

> 在 `ble_hs_cfg.gatts_register_cb = gatt_svr_register_cb;` 中设置。

### GATT 服务表字段速查

| 结构 | 关键字段 |
|---|---|
| `ble_gatt_svc_def` | `type`（`BLE_GATT_SVC_TYPE_PRIMARY`/`_SECONDARY`）、`uuid`、`characteristics`、`includes` |
| `ble_gatt_chr_def` | `uuid`、`access_cb`、`flags`、`val_handle`（回填）、`descriptors`、`min_key_size` |
| `ble_gatt_dsc_def` | `uuid`、`att_flags`、`access_cb`、`min_key_size` |

### 关键 API

| 函数 | 说明 |
|---|---|
| `ble_gatts_count_cfg(const struct ble_gatt_svc_def *)` | 预计算句柄数 |
| `ble_gatts_add_svcs(const struct ble_gatt_svc_def *)` | 注册静态服务表 |
| `ble_gatts_add_dynamic_svcs(...)` | 注册动态服务 |
| `ble_gatts_reset(void)` / `ble_gatts_start(void)` | 重置 / 启动服务 |
| `ble_gatts_find_svc(uuid, &handle)` | 按 UUID 查服务句柄 |
| `ble_gatts_find_chr(svc_uuid, chr_uuid, &out_dsc, &out_val)` | 查特征句柄 |
| `ble_gatts_find_dsc(...)` | 查描述符句柄 |

## 常见错误

| 错误 | 原因 | 解决方法 |
|---|---|---|
| 句柄分配错误 | 跳过 `count_cfg` | 先 `count_cfg` 再 `add_svcs` |
| 表注册后客户端看不到 | 未以 `{0}` 结尾或注册在 sync 之后 | 三层都补 `{0}`；在 sync 前注册 |
| 读访问崩溃 | 用 `ctxt->chr` 处理描述符 | 按 `ctxt->op` 区分 chr/dsc |
| 写访问拿不到数据 | 未用 mbuf API | 用 `ble_hs_mbuf_to_flat` 或 `os_mbuf_getdata` |
| 加密后才能访问 | 特征带 `_ENC` / `_AUTHEN` 标志 | 先完成配对（见 `security_pairing.md`） |

## 参考

- `apps/blehr/src/gatt_svr.c` — HRS + DIS 双服务、NOTIFY 特征、register_cb、count+add
- `apps/bleprph/src/gatt_svr.c` — 含 ANS 服务
- 头文件：`nimble/host/include/host/ble_gatt.h`（`ble_gatt_svc_def`、`ble_gatt_chr_def`、`ble_gatts_*`）
