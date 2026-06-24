---
name: esp-nimble-skill
description: >-
  AI Skill for developing BLE firmware on top of the Apache NimBLE BLE stack
  (Espressif esp-nimble fork). Used when creating, modifying, or debugging
  NimBLE host applications including GAP advertising/scanning/connection,
  GATT server/client, SMP pairing/bonding, extended/periodic advertising,
  and Bluetooth Mesh. Covers the NimBLE Host API surface (ble_gap / ble_gatt /
  ble_sm / ble_hs_id) and the FreeRTOS NPL port used by ESP chips.
  Trigger words: "NimBLE", "esp-nimble", "BLE", "bluetooth", "蓝牙", "GAP", "GATT", "SMP",
  "advertising", "广播", "scan", "扫描", "pairing", "配对", "ESP32", "ESP32-C3", "ESP32-S3", "ESP32-H2"
tags:
  - embedded
  - BLE
  - bluetooth
  - NimBLE
  - esp-nimble
  - GAP
  - GATT
  - SMP
  - mesh
  - ESP32
  - firmware
license: Apache-2.0
compatibility: >-
  Target chips: ESP32 (with controller lib), ESP32-C3, ESP32-S3, ESP32-C2,
  ESP32-H2, ESP32-H4, and other NimBLE-supported controllers. Build via
  ESP-IDF (FreeRTOS NPL port in porting/npl/freertos and porting/npl/esp-idf).
metadata:
  author: Community
  version: "1.1.0"
---

# esp-nimble-skill

面向 Apache NimBLE BLE 协议栈（Espressif 的 esp-nimble 分支）固件开发的 AI Skill。NimBLE 是一个完全开源的 Bluetooth 5.x Host + Controller 协议栈，Host 提供完整的 GAP（广播 / 扫描 / 连接 / 连接参数更新）、GATT（服务端注册与客户端读写订阅）、ATT、L2CAP、SMP（Legacy 与 Secure Connections 配对 / 绑定）能力。本 Skill 提供场景化配方、真实 API 参考、配置项说明与高频陷阱，全部内容均取自仓库 `docs/`、`nimble/host/include/host/` 头文件与 `apps/` 示例代码。

## Core Principles

1. **绝不臆造 API** — 任何函数 / 结构体 / 宏必须能在 `resources/api_reference.md` 或仓库头文件中找到，否则视为不存在。
2. **Host 与 Controller 分离** — NimBLE Host 通过 HCI 与 Controller 通信；在 ESP-IDF 集成下两者运行在同一 CPU（RAM transport），Host 由 `nimble_port_freertos_init` / `esp_nimble_enable` 启动的独立任务驱动。
3. **初始化顺序不可乱** — Host 同步前不要调用任何 GAP/GATT 过程。所有 BLE 操作必须在 `ble_hs_cfg.sync_cb` 被触发（即 Host 已与 Controller 同步）之后发起。
4. **回调驱动模型** — GAP 广播 / 扫描 / 连接、GATT 过程都通过 `ble_gap_event_fn` 回调返回结果，应用在回调里读取 `struct ble_gap_event` 的对应字段。
5. **地址由 Host 管理** — 使用 `ble_hs_id_infer_auto(privacy, &own_addr_type)` 推断本机地址类型，再传给 `ble_gap_adv_start` / `ble_gap_disc` / `ble_gap_connect`；不要直接硬编码地址类型。
6. **GATT 服务以静态表注册** — 用 `struct ble_gatt_svc_def[]` 描述服务 / 特征 / 描述符，先 `ble_gatts_count_cfg()` 计数再 `ble_gatts_add_svcs()` 注册，最后由 Host 在 sync 时 `ble_gatts_start()`。
7. **连接句柄贯穿一切** — `conn_handle` 是后续 terminate / update / read / write / notify 的唯一标识，必须在 `BLE_GAP_EVENT_CONNECT` 中保存 `event->connect.conn_handle`。
8. **notify/indicate 数据用 mbuf** — 通知数据必须通过 `ble_hs_mbuf_from_flat()` 转成 `struct os_mbuf *`，再交给 `ble_gatts_notify_custom()`。
9. **扫描必须先停后连** — `ble_gap_connect` 前若正在扫描，需先 `ble_gap_disc_cancel()`（参见 apps/blecent）。
10. **配对 IO 能力由应用响应** — `BLE_GAP_EVENT_PASSKEY_ACTION` 到达时，应用通过 `ble_sm_inject_io(conn_handle, &io)` 提供输入。
11. **扩展广播以 instance 为单位** — `ble_gap_ext_adv_*` 系列面向每个 advertising instance 单独配置（地址、PHY、数据、周期），最多支持 `CONFIG_BT_NIMBLE_MAX_EXT_ADV_INSTANCES` 个实例。
12. **不要直接修改仓库源码** — NimBLE 源码（`nimble/`、`porting/`）应视为只读，应用代码引用其头文件即可。

## When to Use

**Applicable:**
- 创建基于 NimBLE 的 BLE 固件（外设 / 中心 / 广播者 / 观察者 / 多角色）
- 实现 GATT 服务端（自定义服务、HRS、DIS 等）或 GATT 客户端（服务发现、读写、订阅）
- 实现 SMP 配对 / 绑定 / 加密（Legacy Just Works / Passkey / Secure Connections / OOB）
- 配置传统广播、扩展广播、周期广播、扫描响应
- 实现 Bluetooth Mesh 节点（PB-GATT/PB-ADV、Generic OnOff、Vendor Model）
- 在 ESP-IDF 工程中集成 NimBLE 组件、配置 Kconfig 选项

**Not applicable:**
- 使用 Bluedroid（BT/BLE 双模控制器）协议栈的工程——应使用 Bluedroid API
- 经典蓝牙（BR/EDR）功能——NimBLE 仅支持 BLE
- 非 BLE 的无线协议（Wi-Fi、Zigbee、Thread、ESP-NOW）
- PCB / 射频硬件设计与天线调谐

---

## Scenario Quick Reference (Recipes)

当用户意图匹配下列场景时，**先读对应 recipe**——其中包含完整调用链、分步说明、常见错误与可复制代码。

### Host 初始化与基础

| recipe | scenario |
|---|---|
| `recipes/host_init.md` | NimBLE Host 初始化：ble_hs_cfg 回调、sync/reset、启动 host 任务、保证 sync 后再操作 |
| `recipes/address_setup.md` | 设备地址配置：public / random / NRPA / RPA 隐私、ble_hs_id_infer_auto |

### GAP — 外设（Peripheral / Broadcaster）

| recipe | scenario |
|---|---|
| `recipes/peripheral_adv.md` | 传统可连接广播外设：设置 adv fields、启动广播、处理 CONNECT/DISCONNECT |
| `recipes/beacon.md` | 不可连接广播（Beacon）：non-connectable advertising、自定义 manufacturer data |
| `recipes/notify.md` | 服务端通知 / 指示：NOTIFY/INDICATE 特征、os_mbuf、ble_gatts_notify_custom |
| `recipes/conn_param_update.md` | 连接参数更新：L2CAP/LL 两个过程、ble_gap_update_params、CONN_UPDATE 结果、accept/reject 对端请求 |

### GAP — 中心（Central / Observer）

| recipe | scenario |
|---|---|
| `recipes/scanner.md` | 扫描：passive/active scan、解析 adv fields、BLE_GAP_EVENT_DISC |
| `recipes/central_connect.md` | 中心连接：ble_gap_connect、服务发现、GATT 读 / 写 / 订阅 |

### GATT 服务端

| recipe | scenario |
|---|---|
| `recipes/gatt_server.md` | 自定义 GATT 服务：ble_gatt_svc_def 表、access_cb、count+add、读写权限标志 |

### 安全 / 扩展特性

| recipe | scenario |
|---|---|
| `recipes/security_pairing.md` | SMP 配对与加密：IO 能力、PASSKEY_ACTION、ble_sm_inject_io、加密状态 |
| `recipes/ext_adv.md` | 扩展广播与周期广播：ble_gap_ext_adv_*、多 instance、periodic advertising |
| `recipes/phy_update.md` | LE PHY 选择（1M/2M/Coded）：mask+phy_opts 编码、set_prefered_default/set_prefered_le_phy、PHY_UPDATE_COMPLETE |

### Mesh

| recipe | scenario |
|---|---|
| `recipes/mesh_node.md` | Bluetooth Mesh 节点：bt_mesh_init、Generic OnOff server、CID_VENDOR |

---

## GAP 角色与关键事件速查

### NimBLE 支持的四种 BLE 角色（可并发）

| 角色 | 触发 API | 典型事件 |
|---|---|---|
| Broadcaster（广播者，不可连接） | `ble_gap_adv_start` with `conn_mode=BLE_GAP_CONN_MODE_NON` | `BLE_GAP_EVENT_ADV_COMPLETE` |
| Peripheral（外设，可连接） | `ble_gap_adv_start` with `conn_mode=BLE_GAP_CONN_MODE_UND` | `BLE_GAP_EVENT_CONNECT` / `_DISCONNECT` / `_SUBSCRIBE` / `_MTU` |
| Observer（观察者，扫描） | `ble_gap_disc` | `BLE_GAP_EVENT_DISC` / `BLE_GAP_EVENT_DISC_COMPLETE` |
| Central（中心，发起连接） | `ble_gap_connect` | `BLE_GAP_EVENT_CONNECT` / `_DISCONNECT` / `_MTU` |

### 广播 / 发现模式常量（`ble_gap.h`）

| 常量 | 含义 |
|---|---|
| `BLE_GAP_CONN_MODE_NON` | 不可连接 |
| `BLE_GAP_CONN_MODE_DIR` | 定向可连接 |
| `BLE_GAP_CONN_MODE_UND` | 非定向可连接（最常用） |
| `BLE_GAP_DISC_MODE_NON` | 不可发现 |
| `BLE_GAP_DISC_MODE_LTD` | 有限可发现 |
| `BLE_GAP_DISC_MODE_GEN` | 一般可发现（最常用） |

### 本机地址类型（传给 GAP API 的 `own_addr_type`）

| 常量 | 含义 |
|---|---|
| `BLE_OWN_ADDR_PUBLIC` | 公共地址 |
| `BLE_OWN_ADDR_RANDOM` | 静态随机地址 |
| `BLE_OWN_ADDR_RPA_PUBLIC_DEFAULT` | RPA 隐私，身份地址为 public |
| `BLE_OWN_ADDR_RPA_RANDOM_DEFAULT` | RPA 隐私，身份地址为 random |

### GATT 特征权限标志（`ble_gatt.h`，常用子集）

| 标志 | 含义 |
|---|---|
| `BLE_GATT_CHR_F_READ` | 可读 |
| `BLE_GATT_CHR_F_WRITE` | 可写（带响应） |
| `BLE_GATT_CHR_F_WRITE_NO_RSP` | 可写（无响应） |
| `BLE_GATT_CHR_F_NOTIFY` | 支持通知 |
| `BLE_GATT_CHR_F_INDICATE` | 支持指示 |
| `BLE_GATT_CHR_F_READ_ENC` | 读需加密 |
| `BLE_GATT_CHR_F_READ_AUTHEN` | 读需认证 |
| `BLE_GATT_CHR_F_WRITE_ENC` | 写需加密 |
| `BLE_GATT_CHR_F_NOTIFY_INDICATE_ENC` | notify/indicate 需加密 |

---

## Critical Pitfalls (Must Read)

以下是 NimBLE 开发中最常见的错误，违反任何一条都会导致功能异常。

### 1. Host 未同步就发起 GAP 过程

```c
// ❌ WRONG — 在 main() 里直接启动广播，此时 Host 尚未与 Controller 同步
int main(void) {
    nimble_port_init();
    ble_hs_cfg.sync_cb = on_sync;
    bleprph_advertise();   // 失败：ble_hs_synced() == 0
    nimble_port_freertos_init(nimble_host_task);
}

// ✅ CORRECT — 所有 GAP 操作放在 sync_cb 中
static void on_sync(void) {
    ble_hs_id_infer_auto(0, &own_addr_type);
    bleprph_advertise();   // 此时已同步
}
int main(void) {
    nimble_port_init();
    ble_hs_cfg.sync_cb = on_sync;
    nimble_port_freertos_init(nimble_host_task);
}
```

### 2. 没有保存 conn_handle

```c
// ❌ WRONG — 连接后未保存 handle，无法发送 notify / terminate
case BLE_GAP_EVENT_CONNECT:
    MODLOG_DFLT(INFO, "connected\n");
    return 0;

// ✅ CORRECT — 保存 handle，断开时复位
case BLE_GAP_EVENT_CONNECT:
    if (event->connect.status == 0) {
        conn_handle = event->connect.conn_handle;
    } else {
        advertise();   // 连接失败，恢复广播
    }
    return 0;
case BLE_GAP_EVENT_DISCONNECT:
    conn_handle = BLE_HS_CONN_HANDLE_NONE;
    advertise();
    return 0;
```

### 3. 广播数据超过 31 字节

```c
// ❌ WRONG — name 过长导致 ble_gap_adv_set_fields 返回 BLE_HS_EMSGSIZE
fields.name = (uint8_t *)device_name;
fields.name_len = strlen(device_name);   // 40 字节
fields.name_is_complete = 1;
ble_gap_adv_set_fields(&fields);   // 失败

// ✅ CORRECT — 长名称放 scan response，adv data 只放短数据
ble_gap_adv_set_fields(&fields);              // flags + uuids（<=31）
ble_gap_adv_rsp_set_fields(&rsp_fields);      // name 放 scan response
```

### 4. 未先取消扫描就发起连接

```c
// ❌ WRONG — 扫描进行中直接 connect，返回 BLE_HS_EBUSY
ble_gap_connect(own_addr_type, &disc->addr, 30000, NULL, cb, NULL);

// ✅ CORRECT — 先 disc_cancel 再 connect（见 apps/blecent）
ble_gap_disc_cancel();
ble_gap_connect(own_addr_type, &disc->addr, 30000, NULL, cb, NULL);
```

### 5. notify 数据未用 mbuf

```c
// ❌ WRONG — 直接传裸指针，类型不匹配
uint8_t hrm[2] = {0x06, 90};
ble_gatts_notify_custom(conn_handle, hrs_hrm_handle, hrm);

// ✅ CORRECT — 用 ble_hs_mbuf_from_flat 转换
struct os_mbuf *om = ble_hs_mbuf_from_flat(hrm, sizeof hrm);
ble_gatts_notify_custom(conn_handle, hrs_hrm_handle, om);
```

### 6. GATT 服务表未以 {0} 结尾 / 未 count

```c
// ❌ WRONG — 直接 add，缺 count_cfg 导致 ATT 句柄分配错误
static const struct ble_gatt_svc_def svcs[] = {
    { .type = BLE_GATT_SVC_TYPE_PRIMARY, ... },
    { 0 },
};
ble_gatts_add_svcs(svcs);   // 跳过了 count_cfg

// ✅ CORRECT — 先 count 再 add（见 apps/blehr gatt_svr_init）
ble_gatts_count_cfg(svcs);
ble_gatts_add_svcs(svcs);
```

### 7. access_cb 用错 ctxt 字段

```c
// ❌ WRONG — 对 descriptor 读访问仍取 ctxt->chr
static int access_cb(uint16_t conn, uint16_t attr,
                     struct ble_gatt_access_ctxt *ctxt, void *arg) {
    uint16_t uuid = ble_uuid_u16(ctxt->chr->uuid);   // 描述符访问会崩溃
}

// ✅ CORRECT — 根据 ctxt->op 区分 chr / dsc
if (ctxt->op == BLE_GATT_ACCESS_OP_READ_CHR) {
    uint16_t uuid = ble_uuid_u16(ctxt->chr->uuid);
} else if (ctxt->op == BLE_GATT_ACCESS_OP_READ_DSC) {
    uint16_t uuid = ble_uuid_u16(ctxt->dsc->uuid);
}
```

### 8. 扫描地址类型与实际不匹配

```c
// ❌ WRONG — 设置了 NRPA 却用 PUBLIC 作为 own_addr_type
ble_hs_id_gen_rnd(1, &addr);
ble_hs_id_set_rnd(addr.val);
ble_gap_disc(BLE_OWN_ADDR_PUBLIC, 1000, &scan_params, cb, NULL);

// ✅ CORRECT — NRPA 必须用 BLE_OWN_ADDR_RANDOM（见 apps/scanner、apps/advertiser）
ble_gap_disc(BLE_OWN_ADDR_RANDOM, 1000, &scan_params, cb, NULL);
```

### 9. 配对 PASSKEY 事件未响应

```c
// ❌ WRONG — 收到 PASSKEY_ACTION 不调用 ble_sm_inject_io，配对超时
case BLE_GAP_EVENT_PASSKEY_ACTION:
    MODLOG_DFLT(INFO, "passkey action\n");
    return 0;

// ✅ CORRECT — 按 action 类型注入 IO 数据
case BLE_GAP_EVENT_PASSKEY_ACTION:
    handle_passkey(event->passkey.conn_handle,
                   &event->passkey.params);
    return 0;
// 内部：
struct ble_sm_io io; io.action = BLE_SM_IOACT_INPUT; io.passkey = 123456;
ble_sm_inject_io(conn_handle, &io);
```

### 10. 扩展广播用了 legacy 的 set_fields

```c
// ❌ WRONG — 扩展广播用 ble_gap_adv_set_fields（属于 legacy API）
ble_gap_ext_adv_configure(instance, &params, NULL, cb, NULL);
ble_gap_adv_set_fields(legacy_fields);   // 用错了 API

// ✅ CORRECT — 扩展广播数据用 mbuf + ble_gap_ext_adv_set_data（见 apps/ext_advertiser）
struct os_mbuf *data = os_msys_get_pkthdr(600, 0);
os_mbuf_append(data, adv_data, 600);
ble_gap_ext_adv_set_data(instance, data);
```

### 11. 重复注册 GAP/GATT 服务

```c
// ❌ WRONG — 每次 sync_cb 都 add_svcs，导致句柄重复
static void on_sync(void) {
    ble_gatts_add_svcs(my_svcs);
    advertise();
}

// ✅ CORRECT — gatts_add_svcs 在初始化阶段调用一次，sync_cb 只做地址+广播
int app_init(void) {
    ble_gatts_count_cfg(my_svcs);
    ble_gatts_add_svcs(my_svcs);
    return 0;
}
```

### 12. 把 BLE_HS_FOREVER 当成"立即"

```c
// ❌ WRONG — 误以为 duration_ms=BLE_HS_FOREVER 是 0 立即返回
// 实际它表示"持续到被显式停止"，广播不会自动结束
ble_gap_adv_start(own_addr_type, NULL, BLE_HS_FOREVER, &adv_params, cb, NULL);

// ✅ CORRECT — 需要限时广播时传有限毫秒数（如 peripheral 示例传 100）
ble_gap_adv_start(own_addr_type, NULL, 100, &adv_params, cb, NULL);
// 广播结束后会收到 BLE_GAP_EVENT_ADV_COMPLETE，可在其中重启
```

---

## Execution Workflow

| Step | Name | Description |
|------|------|-------------|
| 1 | Plan | 确认目标角色（外设 / 中心 / 广播者 / 观察者 / Mesh），目标芯片与 ESP-IDF 版本 |
| 2 | Recipe | 在 `recipes/` 中匹配场景，遵循其调用链与分步说明 |
| 3 | Query | recipe 未覆盖的 API，查 `resources/api_reference.md`（按模块分组的真实签名） |
| 4 | Validate | 校验函数签名、头文件包含、GAP 模式常量、GATT 权限标志均来自真实头文件 |
| 5 | Confirm | 向用户呈现实现方案：include 列表、ble_hs_cfg 回调、GAP 参数、GATT 服务表、连接句柄处理 |
| 6 | Execute | 参考最近示例（apps/bleprph、blecent、blehr、scanner、advertiser、ext_advertiser、blemesh）改写 |
| 7 | Check | 验证：sync 后才操作、conn_handle 已保存、notify 用 mbuf、服务表 {0} 结尾且已 count |
| 8 | Build | idf.py set-target / idf.py build，确保 `CONFIG_BT_ENABLED=y`、`CONFIG_BT_NIMBLE_ENABLED=y` |
| 9 | Debug | 用 ESP-IDF monitor 看日志，用 nRF Connect / Wireshark + btsnoop 验证空口行为 |

### Step 6 Detail — 示例选取策略

根据需求选择最接近的仓库示例作为起点（路径相对仓库根 `apps/`）：

- 可连接外设 → `apps/bleprph`（完整 GAP 事件处理）或 `apps/peripheral`（精简版）
- 心率服务 + 通知 → `apps/blehr`（GATT 服务表 + notify + 定时器）
- 中心：扫描 + 连接 + 服务发现 + 读写订阅 → `apps/blecent`
- 仅扫描 → `apps/scanner`
- 不可连接广播 → `apps/advertiser`
- 扩展 / 周期广播 → `apps/ext_advertiser`
- Mesh 节点 → `apps/blemesh`、`apps/blemesh_light`、`apps/blemesh_models_example_1`

复制示例源文件后按需修改服务表、广播数据、回调逻辑，再集成进 ESP-IDF 工程的 `main/` 目录。

---

## Failure Strategies

| Situation | Action |
|---|---|
| API 在 `resources/` 中查不到 | 立即停止，告知用户该 API 不属于 NimBLE Host，不臆造 |
| 不确定是否已 sync | 用 `ble_hs_synced()` / `ble_hs_is_enabled()` 判断；GAP 操作放 `sync_cb` |
| 广播启动返回 BLE_HS_EBUSY | 先 `ble_gap_adv_stop()` / `ble_gap_adv_active()` 检查是否已在广播 |
| connect 返回 EBUSY | 先 `ble_gap_disc_cancel()` 停止扫描 |
| 配对一直超时 | 检查 `BLE_GAP_EVENT_PASSKEY_ACTION` 是否被处理并 `ble_sm_inject_io` |
| notify 客户端收不到 | 确认客户端已写 CCCD（订阅），特征带 `BLE_GATT_CHR_F_NOTIFY` |
| GATT 服务在客户端看不到 | 确认 `ble_gatts_count_cfg` + `ble_gatts_add_svcs` 在 sync 前完成 |
| Mesh 不工作 | 确认 `CONFIG_BT_NIMBLE_MESH=y` 且 `bt_mesh_init` 已调用 |

## References

- 场景配方 → `recipes/` 目录
- Host API 参考 → `resources/api_reference.md`
- 配置项参考 → `resources/config_reference.md`
- 常见陷阱汇总 → `resources/pitfalls.md`
- 仓库示例索引 → `resources/example_list.md`
- 仓库官方文档 → `D:/esp-skill/espressif-repos/esp-nimble/docs/`（index.rst、ble_setup/、ble_hs/、ble_sec.rst、mesh/）
- 仓库示例 → `D:/esp-skill/espressif-repos/esp-nimble/apps/`（bleprph、blecent、blehr、scanner、advertiser、ext_advertiser、blemesh 等）
