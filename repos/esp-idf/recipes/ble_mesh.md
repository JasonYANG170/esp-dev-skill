# ESP-BLE-MESH 节点（Generic OnOff Server）

> **适用摘要**: 用 ESP-BLE-MESH 协议栈实现一个 Mesh 节点（Generic OnOff Server 模型）：配置 Composition Data（含 Config Server + Generic OnOff Server 模型）、`esp_ble_mesh_init` 启动、开启 PB-ADV/PB-GATT 配网承载（Provisioning）、`esp_ble_mesh_node_prov_enable` 进入待配网状态、处理 Generic Server 的 GET/SET 消息、`esp_ble_mesh_model_publish` 发布状态。适配自 `examples/bluetooth/esp_ble_mesh/onoff_models/onoff_server`。

> Version: ESP-IDF version used by the project.
> Validation: draft metadata added from repository routing; verify APIs, Kconfig symbols, and component dependencies against the selected version.

## 触发意图

- "BLE Mesh"
- "Mesh 组网"
- "配网 / Provisioning"
- "Generic OnOff"
- "蓝牙网格 / 多节点控制"
- "esp_ble_mesh"

## 前置条件

| 条件 | 要求 |
|---|---|
| 目标芯片 | 需支持 BLE（esp32/esp32c3/s3/c2/c5/c6/c61/h2 等） |
| Kconfig | `CONFIG_BT_ENABLED=y`；启用 BLE Mesh 组件（`CONFIG_BLE_MESH=y`）。menuconfig：`Component config → Bluetooth → Bluetooth Mesh support → BLE Mesh`；Host 可选 Bluedroid（`CONFIG_BT_BLUEDROID_ENABLED`）或 NimBLE（`CONFIG_BT_NIMBLE_ENABLED`） |
| 组件 | `bt`、`esp_ble_mesh`、`nvs_flash` |
| 头文件 | `esp_ble_mesh_defs.h`、`esp_ble_mesh_common_api.h`、`esp_ble_mesh_networking_api.h`、`esp_ble_mesh_provisioning_api.h`、`esp_ble_mesh_config_model_api.h`、`esp_ble_mesh_generic_model_api.h`、`esp_ble_mesh_local_data_operation_api.h` |
| 工具 | **配网器（Provisioner）**：手机 App（nRF Mesh / ESP BLE Mesh Provisioner）或 `examples/bluetooth/esp_ble_mesh/provisioner`；节点本身不能自配网 |
| 参考 | `examples/bluetooth/esp_ble_mesh/onoff_models/onoff_server`、`onoff_client`、`provisioner`、`fast_provisioning` |

> BLE Mesh 的初始化（controller + bluedroid）与普通 BLE 共用，差异在之上叠加 Mesh 协议层。示例中 `bluetooth_init()`（来自 `common_components/example_init/`）封装了 controller+host 启动。

## 分步说明

### Composition Data：声明模型与元素

Mesh 节点通过 Composition Data 向网络声明自己有哪些元素（Element）、每个元素上有哪些模型（Model）。最少要有 Config Server（配网时由 Provisioner 配置）+ 一个应用模型（此处 Generic OnOff Server）。

```c
#include "esp_ble_mesh_defs.h"
#include "esp_ble_mesh_common_api.h"
#include "esp_ble_mesh_networking_api.h"
#include "esp_ble_mesh_provisioning_api.h"
#include "esp_ble_mesh_config_model_api.h"
#include "esp_ble_mesh_generic_model_api.h"
#include "esp_ble_mesh_local_data_operation_api.h"
#include "esp_log.h"
#include "nvs_flash.h"

#define TAG      "BLE_MESH"
#define CID_ESP  0x02E5
static uint8_t dev_uuid[16] = { 0xdd, 0xdd };

/* Config Server 模型（必备，由协议栈自动响应配网消息） */
static esp_ble_mesh_cfg_srv_t config_server = {
    .net_transmit = ESP_BLE_MESH_TRANSMIT(2, 20),   /* 3 次重传，20ms 间隔 */
    .relay = ESP_BLE_MESH_RELAY_DISABLED,
    .relay_retransmit = ESP_BLE_MESH_TRANSMIT(2, 20),
    .beacon = ESP_BLE_MESH_BEACON_ENABLED,
    .gatt_proxy = ESP_BLE_MESH_GATT_PROXY_NOT_SUPPORTED,
    .friend_state = ESP_BLE_MESH_FRIEND_NOT_SUPPORTED,
    .default_ttl = 7,
};

/* Generic OnOff Server：rsp_ctrl 决定协议栈自动响应还是交给应用 */
/* ESP_BLE_MESH_MODEL_PUB_DEFINE(名字, publish_msg_len, role) 定义发布上下文 */
ESP_BLE_MESH_MODEL_PUB_DEFINE(onoff_pub_0, 2 + 3, ROLE_NODE);
static esp_ble_mesh_gen_onoff_srv_t onoff_server_0 = {
    .rsp_ctrl = {
        .get_auto_rsp = ESP_BLE_MESH_SERVER_AUTO_RSP,   /* GET 自动回当前状态 */
        .set_auto_rsp = ESP_BLE_MESH_SERVER_RSP_BY_APP, /* SET 交给应用处理 */
    },
};

/* 模型数组：第 0 号元素含 Config Server + OnOff Server */
static esp_ble_mesh_model_t root_models[] = {
    ESP_BLE_MESH_MODEL_CFG_SRV(&config_server),
    ESP_BLE_MESH_MODEL_GEN_ONOFF_SRV(&onoff_pub_0, &onoff_server_0),
};

/* 元素数组 */
static esp_ble_mesh_elem_t elements[] = {
    ESP_BLE_MESH_ELEMENT(0, root_models, ESP_BLE_MESH_MODEL_NONE),
};

/* Composition Data 根结构 */
static esp_ble_mesh_comp_t composition = {
    .cid = CID_ESP,
    .element_count = ARRAY_SIZE(elements),
    .elements = elements,
};
```

### Provisioning 配置 + 初始化 + 开启承载

```c
/* Provisioning 配置（OOB 信息，此处禁用 OOB 用静态配网） */
static esp_ble_mesh_prov_t provision = {
    .uuid = dev_uuid,
    .output_size = 0,
    .output_actions = 0,
};

/* 配网回调：跟踪配网进度 */
static void example_ble_mesh_provisioning_cb(esp_ble_mesh_prov_cb_event_t event,
                                             esp_ble_mesh_prov_cb_param_t *param)
{
    switch (event) {
    case ESP_BLE_MESH_PROV_REGISTER_COMP_EVT:
        ESP_LOGI(TAG, "prov register complete, err=%d", param->prov_register_comp.err_code);
        break;
    case ESP_BLE_MESH_NODE_PROV_ENABLE_COMP_EVT:
        ESP_LOGI(TAG, "node prov enable complete, err=%d", param->node_prov_enable_comp.err_code);
        break;
    case ESP_BLE_MESH_NODE_PROV_LINK_OPEN_EVT:
        ESP_LOGI(TAG, "link open, bearer=%s",
                 param->node_prov_link_open.bearer == ESP_BLE_MESH_PROV_ADV ? "PB-ADV" : "PB-GATT");
        break;
    case ESP_BLE_MESH_NODE_PROV_COMPLETE_EVT:
        /* 配网完成：拿到网络密钥索引与本节点单播地址 */
        ESP_LOGI(TAG, "provisioned! net_idx=0x%04x addr=0x%04x",
                 param->node_prov_complete.net_idx, param->node_prov_complete.addr);
        break;
    case ESP_BLE_MESH_NODE_PROV_RESET_EVT:
        ESP_LOGI(TAG, "node reset (unprovisioned again)");
        break;
    default:
        break;
    }
}

/* Generic Server 回调：处理 GET/SET 消息与状态变化 */
static void example_ble_mesh_generic_server_cb(esp_ble_mesh_generic_server_cb_event_t event,
                                               esp_ble_mesh_generic_server_cb_param_t *param)
{
    esp_ble_mesh_gen_onoff_srv_t *srv;
    switch (event) {
    case ESP_BLE_MESH_GENERIC_SERVER_STATE_CHANGE_EVT:
        /* 状态已被协议栈更新（auto_rsp 场景） */
        if (param->ctx.recv_op == ESP_BLE_MESH_MODEL_OP_GEN_ONOFF_SET ||
            param->ctx.recv_op == ESP_BLE_MESH_MODEL_OP_GEN_ONOFF_SET_UNACK) {
            ESP_LOGI(TAG, "onoff state -> %d", param->value.state_change.onoff_set.onoff);
            /* TODO: 在这里控制 GPIO / LED */
        }
        break;

    case ESP_BLE_MESH_GENERIC_SERVER_RECV_GET_MSG_EVT:
        /* 客户端发 GET（若 get_auto_rsp != AUTO_RSP 才会到这里） */
        if (param->ctx.recv_op == ESP_BLE_MESH_MODEL_OP_GEN_ONOFF_GET) {
            srv = (esp_ble_mesh_gen_onoff_srv_t *)param->model->user_data;
            example_handle_gen_onoff_msg(param->model, &param->ctx, NULL);
        }
        break;

    case ESP_BLE_MESH_GENERIC_SERVER_RECV_SET_MSG_EVT:
        /* 客户端发 SET（set_auto_rsp == RSP_BY_APP 才会到这里） */
        if (param->ctx.recv_op == ESP_BLE_MESH_MODEL_OP_GEN_ONOFF_SET ||
            param->ctx.recv_op == ESP_BLE_MESH_MODEL_OP_GEN_ONOFF_SET_UNACK) {
            example_handle_gen_onoff_msg(param->model, &param->ctx, &param->value.set.onoff);
        }
        break;

    default:
        break;
    }
}

/* 应用层响应 GET/SET 并发布状态（节选自 onoff_server 示例） */
static void example_handle_gen_onoff_msg(esp_ble_mesh_model_t *model,
        esp_ble_mesh_msg_ctx_t *ctx, esp_ble_mesh_server_recv_gen_onoff_set_t *set)
{
    esp_ble_mesh_gen_onoff_srv_t *srv = (esp_ble_mesh_gen_onoff_srv_t *)model->user_data;
    switch (ctx->recv_op) {
    case ESP_BLE_MESH_MODEL_OP_GEN_ONOFF_GET:
        /* 回 STATUS，附带当前 onoff 值（1 字节） */
        esp_ble_mesh_server_model_send_msg(model, ctx,
                ESP_BLE_MESH_MODEL_OP_GEN_ONOFF_STATUS, sizeof(srv->state.onoff), &srv->state.onoff);
        break;
    case ESP_BLE_MESH_MODEL_OP_GEN_ONOFF_SET:
    case ESP_BLE_MESH_MODEL_OP_GEN_ONOFF_SET_UNACK:
        srv->state.onoff = set->onoff;   /* 更新状态 */
        if (ctx->recv_op == ESP_BLE_MESH_MODEL_OP_GEN_ONOFF_SET) {
            /* SET（ack）需回 STATUS；SET_UNACK 不回 */
            esp_ble_mesh_server_model_send_msg(model, ctx,
                    ESP_BLE_MESH_MODEL_OP_GEN_ONOFF_STATUS, sizeof(srv->state.onoff), &srv->state.onoff);
        }
        /* 发布状态给订阅者（让 group 里其他节点同步） */
        esp_ble_mesh_model_publish(model, ESP_BLE_MESH_MODEL_OP_GEN_ONOFF_STATUS,
                sizeof(srv->state.onoff), &srv->state.onoff, ROLE_NODE);
        /* 应用层动作：点灯/灭灯 */
        break;
    default:
        break;
    }
}

static esp_err_t ble_mesh_init(void)
{
    esp_err_t err;
    /* 注册各类回调（配网、Config Server、Generic Server） */
    esp_ble_mesh_register_prov_callback(example_ble_mesh_provisioning_cb);
    esp_ble_mesh_register_config_server_callback(NULL);             /* 可选：监听 AppKey/Sub 绑定 */
    esp_ble_mesh_register_generic_server_callback(example_ble_mesh_generic_server_cb);

    /* 初始化 Mesh 协议栈（注册 provision + composition） */
    err = esp_ble_mesh_init(&provision, &composition);
    if (err != ESP_OK) {
        ESP_LOGE(TAG, "mesh init failed: %d", err);
        return err;
    }

    /* 开启配网承载：PB-ADV（广播承载）+ PB-GATT（GATT 承载） */
    err = esp_ble_mesh_node_prov_enable((esp_ble_mesh_prov_bearer_t)
                                        (ESP_BLE_MESH_PROV_ADV | ESP_BLE_MESH_PROV_GATT));
    if (err != ESP_OK) {
        ESP_LOGE(TAG, "node prov enable failed: %d", err);
        return err;
    }
    ESP_LOGI(TAG, "BLE Mesh node initialized, waiting to be provisioned...");
    return ESP_OK;
}
```

### app_main：controller/host 初始化 + Mesh 初始化

```c
void app_main(void)
{
    ESP_LOGI(TAG, "Initializing...");
    esp_err_t err = nvs_flash_init();
    if (err == ESP_ERR_NVS_NO_FREE_PAGES) {
        ESP_ERROR_CHECK(nvs_flash_erase());
        err = nvs_flash_init();
    }
    ESP_ERROR_CHECK(err);

    /* controller + bluedroid 初始化（与普通 BLE 一致）
     * 示例用 bluetooth_init() 封装；手写则同 ble_peripheral recipe */
    err = bluetooth_init();
    if (err) { ESP_LOGE(TAG, "bluetooth init failed: %d", err); return; }

    /* 用本机 BD 地址填充 dev_uuid（避免多节点 UUID 重复） */
    ble_mesh_get_dev_uuid(dev_uuid);

    err = ble_mesh_init();
    if (err) ESP_LOGE(TAG, "mesh init failed: %d", err);
}
```

> `bluetooth_init()` 与 `ble_mesh_get_dev_uuid()` 来自 `examples/bluetooth/esp_ble_mesh/common_components/example_init/ble_mesh_example_init.c`，封装了 `esp_bt_controller_init/enable(ESP_BT_MODE_BLE)` + `esp_bluedroid_init_with_cfg/enable`。

### 关键 API

```c
/* esp_ble_mesh_common_api.h */
esp_err_t esp_ble_mesh_init(esp_ble_mesh_prov_t *prov, esp_ble_mesh_comp_t *comp);
/* esp_ble_mesh_provisioning_api.h */
esp_err_t esp_ble_mesh_register_prov_callback(esp_ble_mesh_prov_cb_t callback);
esp_err_t esp_ble_mesh_node_prov_enable(esp_ble_mesh_prov_bearer_t bearers);   /* ADV | GATT */
esp_err_t esp_ble_mesh_node_prov_disable(esp_ble_mesh_prov_bearer_t bearers);
/* esp_ble_mesh_networking_api.h */
esp_err_t esp_ble_mesh_server_model_send_msg(esp_ble_mesh_model_t *model, esp_ble_mesh_msg_ctx_t *ctx,
        uint32_t op, uint16_t length, uint8_t *data);
esp_err_t esp_ble_mesh_model_publish(esp_ble_mesh_model_t *model, uint32_t op,
        uint16_t length, uint8_t *data, esp_ble_mesh_dev_role_t device_role);
/* esp_ble_mesh_generic_model_api.h（注册回调） */
esp_err_t esp_ble_mesh_register_generic_server_callback(esp_ble_mesh_generic_server_cb_t callback);
/* esp_ble_mesh_config_model_api.h */
esp_err_t esp_ble_mesh_register_config_server_callback(esp_ble_mesh_cfg_server_cb_t callback);
/* 模型声明宏 */
ESP_BLE_MESH_MODEL_CFG_SRV(cfg_srv_ptr);
ESP_BLE_MESH_MODEL_GEN_ONOFF_SRV(pub_ptr, srv_ptr);
ESP_BLE_MESH_ELEMENT(loc, models, vnd_models);
```

关键配网事件链：`PROV_REGISTER_COMP_EVT`（协议栈就绪）→ `node_prov_enable` → `NODE_PROV_LINK_OPEN_EVT`（配网器连入，区分 PB-ADV/PB-GATT）→ `NODE_PROV_COMPLETE_EVT`（配网完成，拿到 unicast addr + net_idx）→ 此后节点收到的 AppKey 由 Provisioner 通过 Config Server 模型自动绑定。

## 常见错误

| 错误 | 原因 | 解决方法 |
|---|---|---|
| 配网器找不到节点 | 未 `node_prov_enable` 或 UUID 重复 | 调用 `esp_ble_mesh_node_prov_enable(ADV|GATT)`；用 BD 地址填充 UUID（`ble_mesh_get_dev_uuid`） |
| 配网后收不到 SET | AppKey 未绑定或模型未绑 AppKey | 配网完成后，Provisioner 通过 Config Server 发 `AppKey Add` + `Model App Bind`；确认 Config Server 模型在 Composition 里 |
| `esp_ble_mesh_init` 报错 | controller/host 未启动 | 先 `bluetooth_init`（controller+bluedroid enable）再 `esp_ble_mesh_init` |
| `server_model_send_msg` 失败 | 模型未绑 AppKey 或未配网 | 仅在配网完成后、AppKey 绑定后才能发消息；检查 `NODE_PROV_COMPLETE_EVT` 是否触发 |
| `ESP_BLE_MESH_MODEL_OP_*` 未定义 | 未 include 对应模型头 | Generic OnOff 用 `esp_ble_mesh_generic_model_api.h`；Config 用 `esp_ble_mesh_config_model_api.h` |
| Generic Server 回调不触发 | `rsp_ctrl` 设了 AUTO_RSP | `set_auto_rsp = RSP_BY_APP` 才会进 `RECV_SET_MSG_EVT`；全 AUTO_RSP 则走 `STATE_CHANGE_EVT` |
| 链接 `esp_ble_mesh_*` undefined | 未启用 BLE Mesh 组件 | menuconfig 启用 `Bluetooth → Bluetooth Mesh support → BLE Mesh` |

## 参考

- `examples/bluetooth/esp_ble_mesh/onoff_models/onoff_server` — Generic OnOff Server（本 recipe 主来源）
- `examples/bluetooth/esp_ble_mesh/onoff_models/onoff_client` — OnOff Client（发起 GET/SET）
- `examples/bluetooth/esp_ble_mesh/provisioner` — Provisioner（配网器，配网其他节点）
- `examples/bluetooth/esp_ble_mesh/fast_provisioning` — 快速配网
- `examples/bluetooth/esp_ble_mesh/common_components/example_init/ble_mesh_example_init.c` — `bluetooth_init()` / `ble_mesh_get_dev_uuid()` 封装
- ESP-IDF `components/bt/esp_ble_mesh/api/core/include/esp_ble_mesh_common_api.h`、`esp_ble_mesh_provisioning_api.h`、`.../models/include/esp_ble_mesh_generic_model_api.h`
- 文档 `docs/en/api-reference/bluetooth/esp-ble-mesh.rst`、`docs/en/api-guides/esp-ble-mesh/`
