# BLE 主机（Bluedroid GATT Client）

> **适用摘要**: 用 Bluedroid 实现 BLE GATT Client：扫描（`esp_ble_gap_set_scan_params` + `esp_ble_gap_start_scanning`）、连接（`esp_ble_gattc_open` / `esp_ble_gattc_enh_open`）、服务/特征值发现（`esp_ble_gattc_search_service` + `esp_ble_gattc_get_char_by_uuid`）、读写（`esp_ble_gattc_read_char` / `esp_ble_gattc_write_char`）、订阅 notify。适配自 `examples/bluetooth/bluedroid/ble/gatt_client`。

> Version: ESP-IDF version used by the project.
> Validation: draft metadata added from repository routing; verify APIs, Kconfig symbols, and component dependencies against the selected version.

## 触发意图

- "BLE 主机"
- "GATT Client"
- "BLE 扫描 / scan"
- "BLE 连接从机"
- "esp_ble_gattc"
- "读 / 写 BLE 特征值"

## 前置条件

| 条件 | 要求 |
|---|---|
| 目标芯片 | 需支持 BLE（esp32/esp32c3/esp32s3/esp32c2/c5/c6/c61/h2 等） |
| Kconfig | `CONFIG_BT_ENABLED=y`、`CONFIG_BT_BLUEDROID_ENABLED=y` |
| 组件 | `bt`、`nvs_flash` |
| 头文件 | `esp_bt.h`、`esp_bt_main.h`、`esp_gap_ble_api.h`、`esp_gattc_api.h`、`esp_gatt_defs.h`、`esp_gatt_common_api.h` |
| 参考 | `examples/bluetooth/bluedroid/ble/gatt_client`、`examples/bluetooth/bluedroid/ble/gattc_multi_connect` |

## 分步说明

### 初始化 + 注册 client app（与 Server 共用 controller/Host 启动顺序）

客户端初始化与 GATT Server 几乎一致，差异仅在最后注册的是 `gattc` 回调与 app。

```c
#include "esp_bt.h"
#include "esp_bt_main.h"
#include "esp_gap_ble_api.h"
#include "esp_gattc_api.h"
#include "esp_gatt_defs.h"
#include "esp_gatt_common_api.h"
#include "nvs_flash.h"
#include "esp_log.h"

#define GATTC_TAG           "BLE_CLI"
#define REMOTE_SERVICE_UUID 0x00FF
#define REMOTE_NOTIFY_CHAR  0xFF01
#define PROFILE_A_APP_ID    0

/* 要找的远端设备名（扫描时按名匹配） */
static char remote_device_name[ESP_BLE_ADV_NAME_LEN_MAX] = "ESP_GATTS_DEMO";
static bool connect = false;
static bool get_server = false;

static esp_bt_uuid_t remote_filter_service_uuid = {
    .len = ESP_UUID_LEN_16, .uuid = {.uuid16 = REMOTE_SERVICE_UUID}};
static esp_bt_uuid_t remote_filter_char_uuid = {
    .len = ESP_UUID_LEN_16, .uuid = {.uuid16 = REMOTE_NOTIFY_CHAR}};
static esp_bt_uuid_t notify_descr_uuid = {
    .len = ESP_UUID_LEN_16, .uuid = {.uuid16 = ESP_GATT_UUID_CHAR_CLIENT_CONFIG}};

static esp_ble_scan_params_t ble_scan_params = {
    .scan_type          = BLE_SCAN_TYPE_ACTIVE,
    .own_addr_type      = BLE_ADDR_TYPE_PUBLIC,
    .scan_filter_policy = BLE_SCAN_FILTER_ALLOW_ALL,
    .scan_interval      = ESP_BLE_GAP_SCAN_ITVL_MS(50),
    .scan_window        = ESP_BLE_GAP_SCAN_WIN_MS(30),
    .scan_duplicate     = BLE_SCAN_DUPLICATE_DISABLE,
};

void app_main(void)
{
    esp_err_t ret = nvs_flash_init();
    if (ret == ESP_ERR_NVS_NO_FREE_PAGES || ret == ESP_ERR_NVS_NEW_VERSION_FOUND) {
        ESP_ERROR_CHECK(nvs_flash_erase());
        ret = nvs_flash_init();
    }
    ESP_ERROR_CHECK(ret);

    ESP_ERROR_CHECK(esp_bt_controller_mem_release(ESP_BT_MODE_CLASSIC_BT));
    esp_bt_controller_config_t bt_cfg = BT_CONTROLLER_INIT_CONFIG_DEFAULT();
    ESP_ERROR_CHECK(esp_bt_controller_init(&bt_cfg));
    ESP_ERROR_CHECK(esp_bt_controller_enable(ESP_BT_MODE_BLE));

    esp_bluedroid_config_t cfg = BT_BLUEDROID_INIT_CONFIG_DEFAULT();
    ESP_ERROR_CHECK(esp_bluedroid_init_with_cfg(&cfg));
    ESP_ERROR_CHECK(esp_bluedroid_enable());

    ESP_ERROR_CHECK(esp_ble_gap_register_callback(esp_gap_cb));
    ESP_ERROR_CHECK(esp_ble_gattc_register_callback(esp_gattc_cb));   /* 顶层分发回调 */
    ESP_ERROR_CHECK(esp_ble_gattc_app_register(PROFILE_A_APP_ID));     /* 触发 ESP_GATTC_REG_EVT */
    ESP_ERROR_CHECK(esp_ble_gatt_set_local_mtu(500));
}
```

### 扫描 → 按名匹配 → 连接

`ESP_GATTC_REG_EVT`（client app 注册成功）里设置扫描参数；扫描参数设置完成事件里开始扫描；扫到结果按设备名过滤后停止扫描并发起连接。

```c
struct gattc_profile_inst {
    esp_gattc_cb_t gattc_cb;
    esp_gatt_if_t  gattc_if;
    uint16_t       conn_id;
    uint16_t       service_start_handle;
    uint16_t       service_end_handle;
    uint16_t       char_handle;
    esp_bd_addr_t  remote_bda;
};
static struct gattc_profile_inst gl_profile_tab[1] = {
    [0] = { .gattc_cb = gattc_profile_event_handler, .gattc_if = ESP_GATT_IF_NONE },
};

static void esp_gap_cb(esp_gap_ble_cb_event_t event, esp_ble_gap_cb_param_t *param)
{
    switch (event) {
    case ESP_GAP_BLE_SCAN_PARAM_SET_COMPLETE_EVT:
        esp_ble_gap_start_scanning(30);   /* 30 秒；0 表示持续到 stop */
        break;

    case ESP_GAP_BLE_SCAN_START_COMPLETE_EVT:
        if (param->scan_start_cmpl.status != ESP_BT_STATUS_SUCCESS)
            ESP_LOGE(GATTC_TAG, "scan start failed, status=%x", param->scan_start_cmpl.status);
        break;

    case ESP_GAP_BLE_SCAN_RESULT_EVT: {
        if (param->scan_rst.search_evt != ESP_GAP_SEARCH_INQ_RES_EVT) break;
        /* 从广播数据里解析 Complete Local Name */
        uint8_t adv_name_len = 0;
        uint8_t *adv_name = esp_ble_resolve_adv_data_by_type(param->scan_rst.ble_adv,
                param->scan_rst.adv_data_len + param->scan_rst.scan_rsp_len,
                ESP_BLE_AD_TYPE_NAME_CMPL, &adv_name_len);
        if (adv_name != NULL &&
            strlen(remote_device_name) == adv_name_len &&
            strncmp((char *)adv_name, remote_device_name, adv_name_len) == 0) {
            if (connect == false) {
                connect = true;
                esp_ble_gap_stop_scanning();
                /* 发起连接（v5.x 推荐 enh_open，可指定 PHY） */
                esp_ble_gatt_creat_conn_params_t p = {0};
                memcpy(&p.remote_bda, param->scan_rst.bda, ESP_BD_ADDR_LEN);
                p.remote_addr_type = param->scan_rst.ble_addr_type;
                p.own_addr_type = BLE_ADDR_TYPE_PUBLIC;
                p.is_direct = true;
                p.is_aux = false;
                p.phy_mask = 0x0;
                esp_ble_gattc_enh_open(gl_profile_tab[0].gattc_if, &p);
            }
        }
        break;
    }
    default:
        break;
    }
}
```

> 旧版 `esp_ble_gattc_open(gattc_if, remote_bda, addr_type, is_direct)` 仍可用；`esp_ble_gattc_enh_open` 是 v5.x 新增，参数更完整。

### 连接后：服务发现 → 特征值发现 → 订阅 notify → 写

连接成功（`ESP_GATTC_CONNECT_EVT`）→ MTU 协商 → 服务发现完成（`ESP_GATTC_DIS_SRVC_CMPL_EVT`）触发 `search_service` → 搜索结果（`ESP_GATTC_SEARCH_RES_EVT`）匹配 UUID → 搜索完成（`ESP_GATTC_SEARCH_CMPL_EVT`）按 UUID 取特征值并注册 notify。

```c
static void gattc_profile_event_handler(esp_gattc_cb_event_t event, esp_gatt_if_t gattc_if,
                                        esp_ble_gattc_cb_param_t *param)
{
    switch (event) {
    case ESP_GATTC_REG_EVT:
        esp_ble_gap_set_scan_params(&ble_scan_params);   /* 注册成功后开始配扫描参数 */
        break;

    case ESP_GATTC_CONNECT_EVT:
        gl_profile_tab[0].conn_id = param->connect.conn_id;
        memcpy(gl_profile_tab[0].remote_bda, param->connect.remote_bda, sizeof(esp_bd_addr_t));
        esp_ble_gattc_send_mtu_req(gattc_if, param->connect.conn_id);  /* 协商 MTU */
        break;

    case ESP_GATTC_DIS_SRVC_CMPL_EVT:
        if (param->dis_srvc_cmpl.status != ESP_GATT_OK) break;
        /* 按服务 UUID 搜索（filter_uuid 非空时只返回匹配的） */
        esp_ble_gattc_search_service(gattc_if, param->dis_srvc_cmpl.conn_id,
                                     &remote_filter_service_uuid);
        break;

    case ESP_GATTC_SEARCH_RES_EVT:
        /* 命中目标服务：记录 handle 范围 */
        if (param->search_res.srvc_id.uuid.len == ESP_UUID_LEN_16 &&
            param->search_res.srvc_id.uuid.uuid.uuid16 == REMOTE_SERVICE_UUID) {
            get_server = true;
            gl_profile_tab[0].service_start_handle = param->search_res.start_handle;
            gl_profile_tab[0].service_end_handle   = param->search_res.end_handle;
        }
        break;

    case ESP_GATTC_SEARCH_CMPL_EVT:
        if (!get_server) break;
        /* 在服务 handle 范围内按 UUID 查特征值 */
        uint16_t count = 0;
        esp_ble_gattc_get_attr_count(gattc_if, gl_profile_tab[0].conn_id,
                ESP_GATT_DB_CHARACTERISTIC, gl_profile_tab[0].service_start_handle,
                gl_profile_tab[0].service_end_handle, 0, &count);
        if (count == 0) break;
        esp_gattc_char_elem_t *char_elem_result = malloc(sizeof(*char_elem_result) * count);
        esp_ble_gattc_get_char_by_uuid(gattc_if, gl_profile_tab[0].conn_id,
                gl_profile_tab[0].service_start_handle, gl_profile_tab[0].service_end_handle,
                remote_filter_char_uuid, char_elem_result, &count);
        if (count > 0 && (char_elem_result[0].properties & ESP_GATT_CHAR_PROP_BIT_NOTIFY)) {
            gl_profile_tab[0].char_handle = char_elem_result[0].char_handle;
            /* 注册 notify（远端会回 CCC 写入后才真正使能） */
            esp_ble_gattc_register_for_notify(gattc_if, gl_profile_tab[0].remote_bda,
                    char_elem_result[0].char_handle);
        }
        free(char_elem_result);
        break;

    case ESP_GATTC_REG_FOR_NOTIFY_EVT:
        if (param->reg_for_notify.status != ESP_GATT_OK) break;
        /* 注册成功后，写 CCC descriptor 使能 notify（值 0x0001） */
        uint16_t notify_en = 1;
        /* 先按 char_handle 找 CCC descriptor handle，再写 */
        esp_gattc_descr_elem_t *descr = malloc(sizeof(*descr));
        uint16_t dcount = 1;
        esp_ble_gattc_get_descr_by_char_handle(gattc_if, gl_profile_tab[0].conn_id,
                param->reg_for_notify.handle, notify_descr_uuid, descr, &dcount);
        if (dcount > 0) {
            esp_ble_gattc_write_char_descr(gattc_if, gl_profile_tab[0].conn_id,
                    descr[0].handle, sizeof(notify_en), (uint8_t *)&notify_en,
                    ESP_GATT_WRITE_TYPE_RSP, ESP_GATT_AUTH_REQ_NONE);
        }
        free(descr);
        break;

    case ESP_GATTC_NOTIFY_EVT:
        /* 远端发来的 notify/indicate 数据 */
        ESP_LOGI(GATTC_TAG, "%s len=%d", param->notify.is_notify ? "notify" : "indicate",
                 param->notify.value_len);
        ESP_LOG_BUFFER_HEX(GATTC_TAG, param->notify.value, param->notify.value_len);
        break;

    case ESP_GATTC_WRITE_DESCR_EVT: {
        /* CCC 写完后，可主动写特征值 */
        uint8_t write_char_data[4] = {0xDE, 0xAD, 0xBE, 0xEF};
        esp_ble_gattc_write_char(gattc_if, gl_profile_tab[0].conn_id,
                gl_profile_tab[0].char_handle, sizeof(write_char_data), write_char_data,
                ESP_GATT_WRITE_TYPE_RSP, ESP_GATT_AUTH_REQ_NONE);
        break;
    }

    case ESP_GATTC_DISCONNECT_EVT:
        connect = false;
        get_server = false;
        ESP_LOGI(GATTC_TAG, "disconnected, reason=0x%02x", param->disconnect.reason);
        break;

    default:
        break;
    }
}

/* 顶层分发：REG_EVT 时把 gatts_if 存入 profile 表 */
static void esp_gattc_cb(esp_gattc_cb_event_t event, esp_gatt_if_t gattc_if,
                         esp_ble_gattc_cb_param_t *param)
{
    if (event == ESP_GATTC_REG_EVT) {
        if (param->reg.status == ESP_GATT_OK)
            gl_profile_tab[param->reg.app_id].gattc_if = gattc_if;
        else return;
    }
    /* 按 gattc_if 匹配并分发给对应 profile */
    for (int idx = 0; idx < 1; idx++) {
        if (gattc_if == ESP_GATT_IF_NONE || gattc_if == gl_profile_tab[idx].gattc_if) {
            if (gl_profile_tab[idx].gattc_cb)
                gl_profile_tab[idx].gattc_cb(event, gattc_if, param);
        }
    }
}
```

### 关键 API

```c
/* esp_gap_ble_api.h —— 扫描 */
esp_err_t esp_ble_gap_set_scan_params(esp_ble_scan_params_t *scan_params);
esp_err_t esp_ble_gap_start_scanning(uint32_t duration);   /* 秒；0=持续 */
esp_err_t esp_ble_gap_stop_scanning(void);
uint8_t  *esp_ble_resolve_adv_data_by_type(const uint8_t *adv_data, uint8_t adv_data_len,
                                           esp_ble_adv_data_type type, uint8_t *length);
/* esp_gattc_api.h —— GATT Client */
esp_err_t esp_ble_gattc_register_callback(esp_gattc_cb_t callback);
esp_err_t esp_ble_gattc_app_register(uint16_t app_id);
esp_err_t esp_ble_gattc_open(esp_gatt_if_t gattc_if, esp_bd_addr_t remote_bda,
                             esp_ble_addr_type_t remote_addr_type, bool is_direct);            /* 简化版 */
esp_err_t esp_ble_gattc_enh_open(esp_gatt_if_t gattc_if, esp_ble_gatt_creat_conn_params_t *p); /* v5.x 完整版 */
esp_err_t esp_ble_gattc_send_mtu_req(esp_gatt_if_t gattc_if, uint16_t conn_id);
esp_err_t esp_ble_gattc_search_service(esp_gatt_if_t gattc_if, uint16_t conn_id, esp_bt_uuid_t *filter_uuid);
esp_err_t esp_ble_gattc_get_attr_count(esp_gatt_if_t gattc_if, uint16_t conn_id,
        esp_gatt_db_element_type_t type, uint16_t start_handle, uint16_t end_handle,
        uint16_t char_handle, uint16_t *count);
esp_err_t esp_ble_gattc_get_char_by_uuid(esp_gatt_if_t gattc_if, uint16_t conn_id,
        uint16_t start_handle, uint16_t end_handle, esp_bt_uuid_t char_uuid,
        esp_gattc_char_elem_t *result, uint16_t *count);
esp_err_t esp_ble_gattc_get_descr_by_char_handle(esp_gatt_if_t gattc_if, uint16_t conn_id,
        uint16_t char_handle, esp_bt_uuid_t descr_uuid, esp_gattc_descr_elem_t *result, uint16_t *count);
esp_err_t esp_ble_gattc_read_char(esp_gatt_if_t gattc_if, uint16_t conn_id, uint16_t handle,
        esp_gatt_auth_req_t auth_req);
esp_err_t esp_ble_gattc_write_char(esp_gatt_if_t gattc_if, uint16_t conn_id, uint16_t handle,
        uint16_t value_len, uint8_t *value, esp_gatt_write_type_t write_type, esp_gatt_auth_req_t auth_req);
esp_err_t esp_ble_gattc_register_for_notify(esp_gatt_if_t gattc_if, esp_bd_addr_t remote_bda, uint16_t handle);
esp_err_t esp_ble_gattc_write_char_descr(esp_gatt_if_t gattc_if, uint16_t conn_id, uint16_t handle,
        uint16_t value_len, uint8_t *value, esp_gatt_write_type_t write_type, esp_gatt_auth_req_t auth_req);
esp_err_t esp_ble_gattc_close(esp_gatt_if_t gattc_if, uint16_t conn_id);
```

关键事件链：`ESP_GATTC_REG_EVT`（开始配扫描）→ 扫到目标 → `esp_ble_gap_stop_scanning` + `esp_ble_gattc_open/enh_open` → `ESP_GATTC_CONNECT_EVT`（存 conn_id、协商 MTU）→ `ESP_GATTC_DIS_SRVC_CMPL_EVT`（`search_service`）→ `ESP_GATTC_SEARCH_CMPL_EVT`（查 char、注册 notify）→ `ESP_GATTC_REG_FOR_NOTIFY_EVT`（写 CCC）。

## 常见错误

| 错误 | 原因 | 解决方法 |
|---|---|---|
| 一直扫描不到目标 | 设备名不匹配或对方未广播 | 用 `nRF Connect` 等工具核对广播里的 Complete Local Name；确认对方 `start_advertising` 成功 |
| `esp_ble_gattc_open` 报错 | controller/bluedroid 未 enable 或 `gattc_if` 无效 | 确保启动顺序对；`gattc_if` 来自 `ESP_GATTC_REG_EVT`，未注册前是 `ESP_GATT_IF_NONE` |
| 连上但 `search_service` 无结果 | MTU 未协商或服务未启动 | `ESP_GATTC_CONNECT_EVT` 里先 `send_mtu_req`；确认对端已 `start_service` |
| 收不到 notify | 未写 CCC 或写错 handle | 注册 notify 后，在 `REG_FOR_NOTIFY_EVT` 里用 `get_descr_by_char_handle` 找 CCC 并写 `0x0001` |
| `write_char` 不生效 | `write_type` 不对 | 用 `ESP_GATT_WRITE_TYPE_RSP`（需响应）确保写入；`WRITE_TYPE_NO_RSP` 不回确认但更快 |
| `get_attr_count` 返回 0 | handle 范围错或服务未找到 | 先在 `SEARCH_RES_EVT` 里正确记录 `service_start/end_handle` |

## 参考

- `examples/bluetooth/bluedroid/ble/gatt_client` — 单连接 GATT Client（本 recipe 主来源）
- `examples/bluetooth/bluedroid/ble/gattc_multi_connect` — 多连接客户端
- `examples/bluetooth/bluedroid/ble/gatt_security_client` — 带加密的安全连接客户端
- ESP-IDF `components/bt/host/bluedroid/api/include/api/esp_gattc_api.h`、`esp_gap_ble_api.h`、`esp_gatt_defs.h`
- 文档 `docs/en/api-reference/bluetooth/esp_gattc.rst`、`esp_gap_ble.rst`
