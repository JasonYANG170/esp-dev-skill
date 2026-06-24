# Bluetooth Mesh 节点

> **适用摘要**: 使用 NimBLE Mesh 子系统实现一个 Bluetooth Mesh 节点，注册 Generic OnOff / Health / Vendor 模型，完成初始化与代理广播。

## 触发意图

- "Bluetooth Mesh / 蓝牙 mesh"
- "mesh 节点 / mesh node"
- "Generic OnOff model"
- "PB-GATT / PB-ADV provisioning"
- "bt_mesh_init"

## 前置条件

| 条件 | 要求 |
|---|---|
| 参考示例 | `apps/blemesh/src/main.c`、`apps/blemesh_light/src/main.c` |
| 配置 | `CONFIG_BT_NIMBLE_MESH=y` |
| 文档 | `docs/mesh/index.rst`、`docs/mesh/sample.rst` |
| 头文件 | `#include "mesh/mesh.h"`、`#include "mesh/glue.h"` |

## 分步说明

### 1. 定义元素与模型

mesh 节点由若干 element 组成，每个 element 挂载若干 model。下面用 Generic OnOff server + Health + Vendor 模型。

```c
#include "mesh/mesh.h"
#include "services/gap/ble_svc_gap.h"

#define CID_VENDOR 0x05C3   /* Espressif company id（示例沿用 blemesh） */

/* Health server fault 回调（见 apps/blemesh） */
static int fault_get_cur(struct bt_mesh_model *model, uint8_t *test_id,
                         uint16_t *company_id, uint8_t *faults,
                         uint8_t *fault_count);
static int fault_get_reg(struct bt_mesh_model *model, uint16_t company_id,
                         uint8_t *test_id, uint8_t *faults,
                         uint8_t *fault_count);
static int fault_clear(struct bt_mesh_model *model, uint16_t company_id);

static struct bt_mesh_health_srv_cb health_cb = {
    .fault_get_cur = fault_get_cur,
    .fault_get_reg = fault_get_reg,
    .fault_clear   = fault_clear,
};

static struct bt_mesh_health_srv health_srv = {
    .cb = &health_cb,
};

BT_MESH_HEALTH_PUB_DEFINE(health_pub, 0);

/* Generic OnOff server */
static void gen_onoff_get(struct bt_mesh_model *model,
                          struct bt_mesh_msg_ctx *ctx,
                          struct net_buf_simple *buf);
static void gen_onoff_set(struct bt_mesh_model *model,
                          struct bt_mesh_msg_ctx *ctx,
                          struct net_buf_simple *buf);

static struct bt_mesh_gen_onoff_srv_cb onoff_cb = {
    .get = gen_onoff_get,
    .set = gen_onoff_set,
};

static struct bt_mesh_gen_onoff_srv onoff_srv = {
    .cb = &onoff_cb,
};

/* Vendor model */
static int vendor_msg_recv(struct bt_mesh_model *model,
                           struct bt_mesh_msg_ctx *ctx,
                           struct net_buf_simple *buf);

#define VND_MODEL_ID 0x0001
BT_MESH_MODEL_VND_CB_DEFINE(vnd_srv, CID_VENDOR, VND_MODEL_ID,
                             bt_mesh_vendor_buf, NULL, vendor_msg_recv, NULL);

/* root element 的模型列表 */
static struct bt_mesh_model root_models[] = {
    BT_MESH_MODEL_CFG_SRV,           /* Configuration Server（必需） */
    BT_MESH_MODEL_HEALTH_SRV(&health_srv, &health_pub),
    BT_MESH_MODEL_GEN_ONOFF_SRV(&onoff_srv),
};

static struct bt_mesh_model vnd_models[] = {
    vnd_srv,
};

static struct bt_mesh_elem elements[] = {
    BT_MESH_ELEM(0, root_models, vnd_models),
};

static const struct bt_mesh_comp comp = {
    .cid = CID_VENDOR,
    .elem = elements,
    .elem_count = ARRAY_SIZE(elements),
};
```

### 2. 初始化 Mesh

```c
#include "host/ble_hs.h"

static uint8_t dev_uuid[16] = { /* 设备 UUID，需唯一 */ };
static const struct bt_mesh_prov prov = {
    .uuid = dev_uuid,
    /* .output_*、.input_*、.static_val 等 provisioning IO 回调按需设置 */
};

static void on_sync(void)
{
    int rc;
    rc = bt_mesh_init(&prov, &comp);
    assert(rc == 0);

    /* 启用 provisioning 广播（未被 provision 时） */
    if (!bt_mesh_is_provisioned()) {
        bt_mesh_prov_enable(BT_MESH_PROV_ADV | BT_MESH_PROV_GATT);
    }
}
```

> `on_sync` 即 NimBLE Host 的 sync 回调；Mesh 初始化必须在 Host 同步后进行。

### 3. Generic OnOff 处理

```c
static uint8_t onoff_state;

static void gen_onoff_get(struct bt_mesh_model *model,
                          struct bt_mesh_msg_ctx *ctx,
                          struct net_buf_simple *buf)
{
    NET_BUF_SIMPLE_DEFINE(msg, 2 + 1 + 4);
    bt_mesh_model_msg_init(&msg, BT_MESH_MODEL_OP_2(0x82, 0x04));  /* OnOff Status */
    net_buf_simple_add_u8(&msg, onoff_state);
    bt_mesh_model_send(model, ctx, &msg, NULL, NULL);
}

static void gen_onoff_set(struct bt_mesh_model *model,
                          struct bt_mesh_msg_ctx *ctx,
                          struct net_buf_simple *buf)
{
    onoff_state = net_buf_simple_pull_u8(buf);
    /* 在此驱动 GPIO / 上报状态 */
}
```

> 具体模型宏（`BT_MESH_MODEL_CFG_SRV`、`BT_MESH_MODEL_GEN_ONOFF_SRV`、`BT_MESH_HEALTH_PUB_DEFINE`）及 opcodes 定义在 `nimble/host/mesh/include/mesh/` 下，可参考 `apps/blemesh_light`、`apps/blemesh_models_example_1`。

## 常见错误

| 错误 | 原因 | 解决方法 |
|---|---|---|
| `bt_mesh_init` 返回错误 | Host 未 sync / comp 为空 | 在 `on_sync` 中调用；确保 `elem_count>0` 且含 CFG_SRV |
| 设备无法被 provision | 未调 `bt_mesh_prov_enable` | 未 provision 时 enable ADV/GATT |
| Configuration Server 缺失 | root element 缺 `BT_MESH_MODEL_CFG_SRV` | 第一个 element 必须含 CFG_SRV |
| 模型 opcode 不匹配 | op 编码错误 | 用 `BT_MESH_MODEL_OP_2`/`_3` 宏 |
| Mesh 与普通 BLE 共存冲突 | 同一 Host 实例混用 | Mesh 节点通常不再做普通 GATT 外设 |

## 参考

- `apps/blemesh/src/main.c` — 含 Health server fault 回调、provisioning、元素/模型定义
- `apps/blemesh_light/src/main.c` — Generic OnOff server（灯）实现
- `apps/blemesh_models_example_1/src/main.c`、`apps/blemesh_models_example_2/src/main.c` — 自定义模型示例
- `apps/blemesh_shell/src/main.c` — mesh shell 调试
- 文档：`docs/mesh/index.rst`、`docs/mesh/sample.rst`
- 头文件：`nimble/host/mesh/include/mesh/*.h`
