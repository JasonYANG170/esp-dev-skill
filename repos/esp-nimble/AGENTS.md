# AGENTS.md — Supplementary Agent Guide

> 核心规则、配方索引、GAP 事件表、陷阱与执行流程均在 `SKILL.md`。
> 本文件仅补充 `SKILL.md` 未覆盖的工程约定与工具链指引，不重复内容。

## Project Context

- **Language**: C（C99/C11）
- **Target**: 运行 Apache NimBLE BLE 协议栈（Host + Controller）的固件；在 ESP-IDF 集成下主要面向 ESP32 / ESP32-C3 / ESP32-S3 / ESP32-C2 / ESP32-H2 / ESP32-H4 等 Espressif 芯片，也支持其它 Mynewt / FreeRTOS / NuttX / RIOT / Linux 平台（见 `porting/npl/`）。
- **Build**: ESP-IDF（`idf.py build`）。NimBLE 作为 managed component 或 IDF 内置组件提供；配置经 Kconfig（`CONFIG_BT_NIMBLE_*`）。
- **Stack 架构**: Host（`nimble/host/`）通过 HCI transport（`nimble/transport/`，ESP 上为 RAM/直接调用）与 Controller（`nimble/controller/`、芯片厂商 controller lib）通信；NPL 抽象层（`porting/npl/`）适配不同 RTOS。

## Code Generation Conventions

### File Naming
- 源文件：`*.c`，头文件：`*.h`
- 应用主文件：`main.c` 或 `<app>_main.c`
- GATT 服务定义文件：`gatt_svr.c` / `gatt_svr.h`（参见 `apps/blehr/src/gatt_svr.c`）
- 中心端 peer 管理可选：`peer.c` / `peer.h`（参见 `apps/blecent/src/peer.c`）

### Include Pattern（ESP-IDF / FreeRTOS）

```c
/* NimBLE Host 核心 */
#include "nimble/nimble_port.h"
#include "host/ble_hs.h"
#include "host/ble_gap.h"
#include "host/ble_gatt.h"
#include "host/ble_uuid.h"
#include "host/util/util.h"

/* 标准服务 */
#include "services/gap/ble_svc_gap.h"
#include "services/gatt/ble_svc_gatt.h"

/* FreeRTOS NPL 端口 */
#include "nimble/nimble_port_freertos.h"
```

> 注意：Mynewt 原生示例（`apps/bleprph` 等）使用 `#include "os/mynewt.h"` + `sysinit()`。在 ESP-IDF 下不要照搬 Mynewt 初始化，应使用 `nimble_port_init()` + `nimble_port_freertos_init(nimble_host_task)`（见 `recipes/host_init.md`）。

### Standard ESP-IDF Project Structure

```
my_ble_project/
├── main/
│   ├── main.c              # app_main()：nimble_port_init、ble_hs_cfg 配置、启动 host task
│   ├── gatt_svr.c          # GATT 服务表 + access_cb
│   ├── gatt_svr.h
│   └── ble_app.c           # GAP 广播 / 连接 / 事件处理
├── CMakeLists.txt
├── sdkconfig               # CONFIG_BT_NIMBLE_* 等
└── idf_component.yml       # （如作为 managed component 引用 esp_nimble）
```

### Canonical Init Pattern（ESP-IDF）

```c
#include "nimble/nimble_port.h"
#include "host/ble_hs.h"
#include "nimble/nimble_port_freertos.h"

extern void nimble_host_task(void *param);

void app_main(void)
{
    /* 1. 初始化 NimBLE port（controller + host 配置） */
    nimble_port_init();

    /* 2. 注册 Host 回调与初始服务（在 sync 之前） */
    ble_hs_cfg.sync_cb = on_sync;
    ble_hs_cfg.reset_cb = on_reset;
    ble_hs_cfg.gatts_register_cb = gatt_svr_register_cb;
    ble_hs_cfg.store_status_cb = ble_store_util_status_rr;

    /* 3. 注册 GATT 服务 */
    gatt_svr_init();              // 内部 ble_gatts_count_cfg + ble_gatts_add_svcs

    /* 4. 设置默认设备名 */
    ble_svc_gap_device_name_set("my_device");

    /* 5. 启动 NimBLE host 任务（FreeRTOS） */
    nimble_port_freertos_init(nimble_host_task);
}
```

> `nimble_host_task` 是 SDK 提供的入口（内部 `nimble_port_run`），不要自行实现。`on_sync` 中再启动广播。

### GAP Event Callback Signature

```c
static int gap_event(struct ble_gap_event *event, void *arg);
```
所有 GAP 过程（adv / disc / connect）共享此回调签名，通过 `event->type` 分发。

### GATT access_cb Signature

```c
static int access_cb(uint16_t conn_handle, uint16_t attr_handle,
                     struct ble_gatt_access_ctxt *ctxt, void *arg);
```
通过 `ctxt->op`（`BLE_GATT_ACCESS_OP_READ_CHR` / `_WRITE_CHR` / `_READ_DSC` / `_WRITE_DSC`）区分，并用 `os_mbuf_append(ctxt->om, ...)` 返回读数据。

## Build Workflow（ESP-IDF）

1. `idf.py set-target esp32c3`（或 esp32s3 / esp32h2 等）
2. `idf.py menuconfig` → `Component config → Bluetooth → NimBLE options`：
   - `CONFIG_BT_NIMBLE_ENABLED=y`
   - 按需开启 `CONFIG_BT_NIMBLE_ROLE_PERIPHERAL` / `_CENTRAL` / `_BROADCASTER` / `_OBSERVER`
   - Mesh 时 `CONFIG_BT_NIMBLE_MESH=y`
3. `idf.py build`
4. `idf.py -p /dev/ttyUSB0 flash monitor`
5. 抓包验证：`idf.py monitor` 看日志，nRF Connect / Wireshark + HCI btsnoop 验证

## Code Generation Checklist

- [ ] `nimble_port_init()` 在任何 `ble_hs_*` 配置之前
- [ ] `ble_hs_cfg.sync_cb` 已注册；所有 GAP 操���置于 `on_sync` 中
- [ ] GATT 服务表以 `{ 0 }` 结尾（服务、特征、描述符三层都要）
- [ ] `ble_gatts_count_cfg(svcs)` → `ble_gatts_add_svcs(svcs)` 顺序调用且仅一次
- [ ] 广播 own_addr_type 来自 `ble_hs_id_infer_auto()`，未硬编码
- [ ] `BLE_GAP_EVENT_CONNECT` 中保存 `conn_handle`，失败时恢复广播
- [ ] `BLE_GAP_EVENT_DISCONNECT` 中复位 `conn_handle` 并恢复广播
- [ ] notify 数据经 `ble_hs_mbuf_from_flat()` 转 mbuf
- [ ] 扩展广播使用 `ble_gap_ext_adv_*` 而非 legacy `ble_gap_adv_*`
- [ ] `BLE_GAP_EVENT_PASSKEY_ACTION` 已处理并 `ble_sm_inject_io()`
- [ ] `nimble_port_freertos_init(nimble_host_task)` 已启动 host 任务

## Do Not Modify

- `D:/esp-skill/espressif-repos/esp-nimble/nimble/` — 协议栈源码（host / controller / transport）
- `D:/esp-skill/espressif-repos/esp-nimble/porting/` — NPL 端口实现
- `D:/esp-skill/espressif-repos/esp-nimble/ext/` — 外部依赖
- `SKILL.md` frontmatter — Skill 元数据
- 如需自定义，在自己的工程 `main/` 目录中实现，仅引用 NimBLE 公开头文件。
