# NimBLE Host 初始化

> **适用摘要**: 在 ESP-IDF 工程中正确初始化 NimBLE Host：注册回调、注册 GATT 服务、启动 host 任务，并确保所有 GAP 操作在 Host 同步后发起。

> Evidence: `repos/esp-nimble/resources/`, source/examples in `repos/esp-nimble/`, and this recipe path `repos/esp-nimble/recipes/host_init.md`.
> Validation: draft metadata added from repository routing; verify APIs, Kconfig symbols, and component dependencies against the selected version.

## 触发意图

- "初始化 NimBLE"
- "启动 BLE host"
- "NimBLE 怎么开始"
- "sync callback"
- "nimble_host_task"

## 前置条件

| 条件 | 要求 |
|---|---|
| 参考示例 | `apps/bleprph/src/main.c`、`apps/peripheral/src/main.c` |
| 平台 | ESP-IDF（FreeRTOS NPL 端口） |
| 配置 | `CONFIG_BT_NIMBLE_ENABLED=y` |

## 分步说明

### 1. 初始化 port 与配置 Host 回调

`nimble_port_init()` 完成控制器与 host 的底层初始化。之后才能读写全局的 `ble_hs_cfg`。

```c
#include "nimble/nimble_port.h"
#include "host/ble_hs.h"
#include "nimble/nimble_port_freertos.h"

extern void nimble_host_task(void *param);

static void on_sync(void);
static void on_reset(int reason);

void app_main(void)
{
    nimble_port_init();

    /* Host 配置：回调在 sync / reset / gatt 注册 / 存储状态时触发 */
    ble_hs_cfg.sync_cb            = on_sync;
    ble_hs_cfg.reset_cb           = on_reset;
    ble_hs_cfg.gatts_register_cb  = gatt_svr_register_cb;
    ble_hs_cfg.store_status_cb    = ble_store_util_status_rr;
}
```

> `ble_hs_cfg` 是仓库定义的全局配置结构体（`host/ble_hs.h`），含 `sync_cb`、`reset_cb`、`gatts_register_cb`、`store_status_cb` 等回调字段。

### 2. 注册 GATT 服务（sync 之前）

GATT 服务必须在 Host 与控制器同步之前完成 `count_cfg` + `add_svcs`，这样 Host sync 时会自动 `ble_gatts_start()`。

```c
/* gatt_svr.c 中（参考 apps/blehr/src/gatt_svr.c） */
int gatt_svr_init(void)
{
    int rc;
    rc = ble_gatts_count_cfg(gatt_svr_svcs);   // 预计算 ATT 句柄数
    if (rc != 0) return rc;
    rc = ble_gatts_add_svcs(gatt_svr_svcs);    // 注册服务表
    return rc;
}
```

### 3. 设置设备名并启动 host 任务

```c
void app_main(void)
{
    nimble_port_init();
    ble_hs_cfg.sync_cb = on_sync;
    /* ... gatt_svr_init(); ... */

    ble_svc_gap_device_name_set("my_device");

    /* 启动 NimBLE host 任务（FreeRTOS），内部 nimble_port_run 不返回 */
    nimble_port_freertos_init(nimble_host_task);
}
```

### 4. sync 回调中启动 BLE 业务

`on_sync` 表示 Host 已与 Controller 完成同步，此时可推断地址、启动广播 / 扫描。

```c
static uint8_t own_addr_type;

static void on_sync(void)
{
    /* 推断本机地址类型（无隐私：privacy=0） */
    int rc = ble_hs_id_infer_auto(0, &own_addr_type);
    assert(rc == 0);

    /* 在此启动广播 / 扫描 */
    bleprph_advertise();
}

static void on_reset(int reason)
{
    MODLOG_DFLT(ERROR, "Resetting host; reason=%d\n", reason);
}
```

### 判断同步状态的辅助 API

| 函数 | 含义 |
|---|---|
| `ble_hs_synced(void)` | 返回非 0 表示 Host 已与控制器同步 |
| `ble_hs_is_enabled(void)` | 返回非 0 表示 Host 已启用 |
| `ble_hs_start(void)` | 显式启动 Host（部分移植场景） |

## 常见错误

| 错误 | 原因 | 解决方法 |
|---|---|---|
| 广播/连接 API 返回 `BLE_HS_ENOADDR` | 未配置地址 | 在 `on_sync` 中调用 `ble_hs_id_infer_auto` 或 `ble_hs_util_ensure_addr` |
| GAP 过程返回 `BLE_HS_ENOTCONFIGURED` | Host 未初始化 | 确保 `nimble_port_init()` 在配置 `ble_hs_cfg` 之前 |
| 回调不触发 | 未启动 host 任务 | 确认调用 `nimble_port_freertos_init(nimble_host_task)` |
| GATT 服务��客户端不可见 | `add_svcs` 在 sync 之后才调用 | 把 `gatt_svr_init()` 放在 `nimble_port_freertos_init` 之前 |

## 参考

- `apps/bleprph/src/main.c` — `ble_hs_cfg` 配置 + `on_sync` 启动广播
- `apps/peripheral/src/main.c` — 精简初始化流程
- `apps/blehr/src/main.c` — 配合 GATT 服务注册的初始化
- 头文件：`nimble/host/include/host/ble_hs.h`（`ble_hs_cfg`、`ble_hs_synced`）
