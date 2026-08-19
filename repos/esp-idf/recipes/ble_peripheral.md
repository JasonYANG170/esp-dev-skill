# BLE 外设（Bluedroid GATT Server）

> **适用摘要**: 用 Bluedroid 协议栈实现 BLE GATT Server：控制器/Host 初始化固定顺序、设置广播（`esp_ble_gap_config_adv_data` / raw）、用属性表注册自定义服务与特征值（`esp_ble_gatts_create_attr_tab`）、处理 `ESP_GATTS_WRITE_EVT`、发送 notification/indication（`esp_ble_gatts_send_indicate`）。适配自 `examples/bluetooth/bluedroid/ble/gatt_server_service_table`。

> Validation: draft metadata added from repository routing; verify APIs, Kconfig symbols, and component dependencies against the selected version.

## 触发意图

- "BLE 从机"
- "GATT Server"
- "BLE 广播"
- "BLE 通知 / notify"
- "esp_ble_gatts"
- "Bluedroid 初始化"

## 前置条件

| 条件 | 要求 |
|---|---|
| 目标芯片 | 需支持 BLE（esp32/esp32c3/esp32s3/esp32c2/c5/c6/c61/h2 等；esp32s2 无 BT） |
| Kconfig | `CONFIG_BT_ENABLED=y`、`CONFIG_BT_BLUEDROID_ENABLED=y`（Bluedroid 模式，非 NimBLE）。menuconfig 路径：`Component config → Bluetooth → Host → Bluedroid - Dual-mode` |
| 组件 | `bt`（由 `CONFIG_BT_BLUEDROID_ENABLED` 拉入）、`nvs_flash` |
| 头文件 | `esp_bt.h`、`esp_bt_main.h`、`esp_gap_ble_api.h`、`esp_gatts_api.h`、`esp_bt_device.h`、`esp_gatt_common_api.h` |
| 参考 | `examples/bluetooth/bluedroid/ble/gatt_server_service_table`、`examples/bluetooth/ble_get_started/bluedroid/Bluedroid_GATT_Server` |

## 分步说明

### 初始化顺序（不可颠倒，否则崩溃）

Bluedroid 启动有**固定顺序**，所有 BLE 应用一致：NVS → 释放不用的控制器内存 → controller init/enable → bluedroid init/enable → 注册回调 → 注册 app → 设置 MTU。

```c
/* app_main 中蓝牙初始化（适配自 gatt_server_service_table 示例） */
#include "esp_bt.h"
#include "esp_bt_main.h"
#include "esp_gap_ble_api.h"
#include "esp_gatts_api.h"
#include "esp_bt_device.h"
#include "esp_gatt_common_api.h"
#include "nvs_flash.h"
#include "esp_log.h"

static const char *TAG = "BLE_SRV";

void app_main(void)
{
    /* 1) NVS（蓝牙 bond 信息存 NVS，必须先初始化） */
    esp_err_t ret = nvs_flash_init();
    if (ret == ESP_ERR_NVS_NO_FREE_PAGES || ret == ESP_ERR_NVS_NEW_VERSION_FOUND) {
        ESP_ERROR_CHECK(nvs_flash_erase());
        ret = nvs_flash_init();
    }
    ESP_ERROR_CHECK(ret);

    /* 2) 仅用 BLE，释放 Classic BT 控制器内存（腾出堆给应用） */
    ESP_ERROR_CHECK(esp_bt_controller_mem_release(ESP_BT_MODE_CLASSIC_BT));

    /* 3) 控制器 init → enable，模式必须 BLE */
    esp_bt_controller_config_t bt_cfg = BT_CONTROLLER_INIT_CONFIG_DEFAULT();
    ESP_ERROR_CHECK(esp_bt_controller_init(&bt_cfg));
    ESP_ERROR_CHECK(esp_bt_controller_enable(ESP_BT_MODE_BLE));

    /* 4) Bluedroid Host init → enable（v5.x 用 init_with_cfg） */
    esp_bluedroid_config_t bluedroid_cfg = BT_BLUEDROID_INIT_CONFIG_DEFAULT();
    ESP_ERROR_CHECK(esp_bluedroid_init_with_cfg(&bluedroid_cfg));
    ESP_ERROR_CHECK(esp_bluedroid_enable());

    /* 5) 注册 GAP / GATTS 回调，再注册 app（触发 ESP_GATTS_REG_EVT） */
    ESP_ERROR_CHECK(esp_ble_gatts_register_callback(gatts_event_handler));
    ESP_ERROR_CHECK(esp_ble_gap_register_callback(gap_event_handler));
    ESP_ERROR_CHECK(esp_ble_gatts_app_register(ESP_APP_ID));

    /* 6) 协商更大的 MTU（默认 23 字节，扩到 500 提升吞吐） */
    ESP_ERROR_CHECK(esp_ble_gatt_set_local_mtu(500));
}
```

> `esp_bt_controller_mem_release(ESP_BT_MODE_CLASSIC_BT)` 仅当**只用 BLE** 时调用；若同时用 Classic BT（如 esp32 双模），则改用 `ESP_BT_MODE_BTDM` 且**不能** release。

### 用属性表一次性注册服务（推荐方式）

属性表（attribute table）把 service / characteristic / descriptor 用一个 `esp_gatts_attr_db_t[]` 数组声明，一次 `esp_ble_gatts_create_attr_tab` 调用建库，省去逐个 `esp_ble_gatts_add_char` 的异步繁琐。

```c
#define ESP_APP_ID              0x55
#define SAMPLE_DEVICE_NAME      "ESP_GATTS_DEMO"
#define GATTS_DEMO_CHAR_VAL_LEN_MAX  500
#define CHAR_DECLARATION_SIZE   (sizeof(uint8_t))

/* 服务 / 特征值 UUID（16-bit） */
static const uint16_t GATTS_SERVICE_UUID_TEST = 0x00FF;
static const uint16_t GATTS_CHAR_UUID_TEST_A  = 0xFF01;
static const uint16_t primary_service_uuid        = ESP_GATT_UUID_PRI_SERVICE;
static const uint16_t character_declaration_uuid  = ESP_GATT_UUID_CHAR_DECLARE;
static const uint16_t character_client_config_uuid= ESP_GATT_UUID_CHAR_CLIENT_CONFIG;

/* 特征值属性：读 + 写 + notify（CCC descriptor 配合 notify） */
static const uint8_t char_prop_read_write_notify =
        ESP_GATT_CHAR_PROP_BIT_WRITE | ESP_GATT_CHAR_PROP_BIT_READ | ESP_GATT_CHAR_PROP_BIT_NOTIFY;
static const uint8_t char_prop_read = ESP_GATT_CHAR_PROP_BIT_READ;
static const uint8_t char_prop_write = ESP_GATT_CHAR_PROP_BIT_WRITE;
static const uint8_t heart_measurement_ccc[2] = {0x00, 0x00};   /* CCC 初值 */
static const uint8_t char_value[4] = {0x11, 0x22, 0x33, 0x44};

/* 属性表索引枚举（与数组下标一一对应） */
enum { IDX_SVC, IDX_CHAR_A, IDX_CHAR_VAL_A, IDX_CHAR_CFG_A,
       IDX_CHAR_B, IDX_CHAR_VAL_B, IDX_CHAR_C, IDX_CHAR_VAL_C, HRS_IDX_NB };

/* attr_db：每项 {attr_control} + {uuid_len, uuid_p, perm, max_len, present_len, value_p}
 * ESP_GATT_AUTO_RSP 表示由协议栈自动回复读写请求，应用不参与 */
static const esp_gatts_attr_db_t gatt_db[HRS_IDX_NB] = {
    [IDX_SVC] = {{ESP_GATT_AUTO_RSP}, {ESP_UUID_LEN_16, (uint8_t *)&primary_service_uuid,
        ESP_GATT_PERM_READ, sizeof(uint16_t), sizeof(GATTS_SERVICE_UUID_TEST),
        (uint8_t *)&GATTS_SERVICE_UUID_TEST}},

    [IDX_CHAR_A] = {{ESP_GATT_AUTO_RSP}, {ESP_UUID_LEN_16, (uint8_t *)&character_declaration_uuid,
        ESP_GATT_PERM_READ, CHAR_DECLARATION_SIZE, CHAR_DECLARATION_SIZE,
        (uint8_t *)&char_prop_read_write_notify}},

    [IDX_CHAR_VAL_A] = {{ESP_GATT_AUTO_RSP}, {ESP_UUID_LEN_16, (uint8_t *)&GATTS_CHAR_UUID_TEST_A,
        ESP_GATT_PERM_READ | ESP_GATT_PERM_WRITE, GATTS_DEMO_CHAR_VAL_LEN_MAX,
        sizeof(char_value), (uint8_t *)char_value}},

    /* CCC descriptor：客户端写 0x0001 使能 notify，0x0002 使能 indicate */
    [IDX_CHAR_CFG_A] = {{ESP_GATT_AUTO_RSP}, {ESP_UUID_LEN_16, (uint8_t *)&character_client_config_uuid,
        ESP_GATT_PERM_READ | ESP_GATT_PERM_WRITE, sizeof(uint16_t),
        sizeof(heart_measurement_ccc), (uint8_t *)heart_measurement_ccc}},

    [IDX_CHAR_B] = {{ESP_GATT_AUTO_RSP}, {ESP_UUID_LEN_16, (uint8_t *)&character_declaration_uuid,
        ESP_GATT_PERM_READ, CHAR_DECLARATION_SIZE, CHAR_DECLARATION_SIZE, (uint8_t *)&char_prop_read}},
    [IDX_CHAR_VAL_B] = {{ESP_GATT_AUTO_RSP}, {ESP_UUID_LEN_16, (uint8_t *)&GATTS_CHAR_UUID_TEST_B,
        ESP_GATT_PERM_READ | ESP_GATT_PERM_WRITE, GATTS_DEMO_CHAR_VAL_LEN_MAX,
        sizeof(char_value), (uint8_t *)char_value}},

    [IDX_CHAR_C] = {{ESP_GATT_AUTO_RSP}, {ESP_UUID_LEN_16, (uint8_t *)&character_declaration_uuid,
        ESP_GATT_PERM_READ, CHAR_DECLARATION_SIZE, CHAR_DECLARATION_SIZE, (uint8_t *)&char_prop_write}},
    [IDX_CHAR_VAL_C] = {{ESP_GATT_AUTO_RSP}, {ESP_UUID_LEN_16, (uint8_t *)&GATTS_CHAR_UUID_TEST_C,
        ESP_GATT_PERM_READ | ESP_GATT_PERM_WRITE, GATTS_DEMO_CHAR_VAL_LEN_MAX,
        sizeof(char_value), (uint8_t *)char_value}},
};

static uint16_t heart_rate_handle_table[HRS_IDX_NB];
```

### 回调处理：注册成功后建库 + 配广播 + 处理写 + 发通知

`ESP_GATTS_REG_EVT`（`app_register` 成功）是启动一切的总入口：设设备名 → 配广播数据 → 创建属性表。属性表创建完成（`ESP_GATTS_CREAT_ATTR_TAB_EVT`）后启动服务。

```c
static uint8_t adv_config_done = 0;
#define ADV_CONFIG_FLAG       (1 << 0)
#define SCAN_RSP_CONFIG_FLAG  (1 << 1)

/* raw 广播数据（手工拼装 AD types，比 esp_ble_adv_data_t 更直接） */
static uint8_t raw_adv_data[] = {
    0x02, ESP_BLE_AD_TYPE_FLAG, 0x06,                          /* Flags: LE general disc., BR/EDR not supported */
    0x02, ESP_BLE_AD_TYPE_TX_PWR, 0xEB,                        /* TX power */
    0x03, ESP_BLE_AD_TYPE_16SRV_CMPL, 0xFF, 0x00,              /* Complete 16-bit Service UUIDs: 0x00FF */
    0x0F, ESP_BLE_AD_TYPE_NAME_CMPL,                           /* Complete Local Name */
    'E','S','P','_','G','A','T','T','S','_','D','E','M','O'
};

static esp_ble_adv_params_t adv_params = {
    .adv_int_min       = ESP_BLE_GAP_ADV_ITVL_MS(20),   /* 20ms */
    .adv_int_max       = ESP_BLE_GAP_ADV_ITVL_MS(40),   /* 40ms */
    .adv_type          = ADV_TYPE_IND,                  /* 可连接非定向广播 */
    .own_addr_type     = BLE_ADDR_TYPE_PUBLIC,
    .channel_map       = ADV_CHNL_ALL,
    .adv_filter_policy = ADV_FILTER_ALLOW_SCAN_ANY_CON_ANY,
};

static void gap_event_handler(esp_gap_ble_cb_event_t event, esp_ble_gap_cb_param_t *param)
{
    switch (event) {
    case ESP_GAP_BLE_ADV_DATA_RAW_SET_COMPLETE_EVT:
        adv_config_done &= (~ADV_CONFIG_FLAG);
        if (adv_config_done == 0) esp_ble_gap_start_advertising(&adv_params);
        break;
    case ESP_GAP_BLE_ADV_START_COMPLETE_EVT:
        ESP_LOGI(TAG, "advertising %s",
                 param->adv_start_cmpl.status == ESP_BT_STATUS_SUCCESS ? "started" : "failed");
        break;
    default:
        break;
    }
}

static void gatts_profile_event_handler(esp_gatts_cb_event_t event, esp_gatt_if_t gatts_if,
                                        esp_ble_gatts_cb_param_t *param)
{
    switch (event) {
    case ESP_GATTS_REG_EVT: {
        esp_ble_gap_set_device_name(SAMPLE_DEVICE_NAME);
        esp_ble_gap_config_adv_data_raw(raw_adv_data, sizeof(raw_adv_data));
        adv_config_done |= ADV_CONFIG_FLAG;
        /* 创建属性表，触发 ESP_GATTS_CREAT_ATTR_TAB_EVT */
        esp_ble_gatts_create_attr_tab(gatt_db, gatts_if, HRS_IDX_NB, 0);
        break;
    }
    case ESP_GATTS_CREAT_ATTR_TAB_EVT:
        if (param->add_attr_tab.status != ESP_GATT_OK) {
            ESP_LOGE(TAG, "create attr table failed, status=0x%x", param->add_attr_tab.status);
            break;
        }
        if (param->add_attr_tab.num_handle != HRS_IDX_NB) break;
        /* 拷贝分配到的 handle 数组，启动服务 */
        memcpy(heart_rate_handle_table, param->add_attr_tab.handles, sizeof(heart_rate_handle_table));
        esp_ble_gatts_start_service(heart_rate_handle_table[IDX_SVC]);
        break;

    case ESP_GATTS_CONNECT_EVT:
        ESP_LOGI(TAG, "connected, conn_id=%d", param->connect.conn_id);
        break;

    case ESP_GATTS_DISCONNECT_EVT:
        ESP_LOGI(TAG, "disconnected, reason=0x%x", param->disconnect.reason);
        /* 断开后重新广播，等待新连接 */
        esp_ble_gap_start_advertising(&adv_params);
        break;

    case ESP_GATTS_WRITE_EVT:
        if (param->write.is_prep) break;  /* prepare write 单独处理 */
        /* 客户端写 CCC 使能 notify/indicate */
        if (heart_rate_handle_table[IDX_CHAR_CFG_A] == param->write.handle && param->write.len == 2) {
            uint16_t descr_value = param->write.value[1] << 8 | param->write.value[0];
            if (descr_value == 0x0001) {
                uint8_t notify_data[15] = {0};
                for (int i = 0; i < sizeof(notify_data); i++) notify_data[i] = i % 0xff;
                /* 最后参 is_indicate=false 即 notify；true 即 indicate（需对端回 CONF） */
                esp_ble_gatts_send_indicate(gatts_if, param->write.conn_id,
                        heart_rate_handle_table[IDX_CHAR_VAL_A],
                        sizeof(notify_data), notify_data, false);
            }
        }
        /* need_rsp=true 时必须回 response，否则客户端会等待超时 */
        if (param->write.need_rsp) {
            esp_ble_gatts_send_response(gatts_if, param->write.conn_id,
                    param->write.trans_id, ESP_GATT_OK, NULL);
        }
        break;

    default:
        break;
    }
}

/* 顶层分发：REG_EVT 时保存 gatts_if，再分发给 profile handler */
static void gatts_event_handler(esp_gatts_cb_event_t event, esp_gatt_if_t gatts_if,
                                esp_ble_gatts_cb_param_t *param)
{
    if (event == ESP_GATTS_REG_EVT) {
        if (param->reg.status != ESP_GATT_OK) {
            ESP_LOGE(TAG, "reg app failed, status=%d", param->reg.status);
            return;
        }
        /* 保存 gatts_if 供后续事件匹配（多 profile 场景按 app_id 索引） */
    }
    gatts_profile_event_handler(event, gatts_if, param);
}
```

### 关键 API

```c
/* esp_bt.h —— 控制器 */
esp_err_t esp_bt_controller_mem_release(esp_bt_mode_t mode);   /* 释放不用的控制器内存 */
esp_err_t esp_bt_controller_init(esp_bt_controller_config_t *cfg);
esp_err_t esp_bt_controller_enable(esp_bt_mode_t mode);        /* ESP_BT_MODE_BLE / CLASSIC_BT / BTDM */
/* esp_bt_main.h —— Bluedroid Host */
esp_err_t esp_bluedroid_init_with_cfg(esp_bluedroid_config_t *cfg);  /* v5.x，BT_BLUEDROID_INIT_CONFIG_DEFAULT() */
esp_err_t esp_bluedroid_enable(void);
/* esp_gap_ble_api.h —— 广播 / GAP */
esp_err_t esp_ble_gap_register_callback(esp_gap_ble_cb_t cb);
esp_err_t esp_ble_gap_set_device_name(const char *name);
esp_err_t esp_ble_gap_config_adv_data(esp_ble_adv_data_t *adv_data);        /* 结构化广播 */
esp_err_t esp_ble_gap_config_adv_data_raw(uint8_t *raw_data, uint32_t len); /* 原始字节广播 */
esp_err_t esp_ble_gap_start_advertising(esp_ble_adv_params_t *adv_params);
esp_err_t esp_ble_gap_stop_advertising(void);
/* esp_gatts_api.h —— GATT Server */
esp_err_t esp_ble_gatts_register_callback(esp_gatts_cb_t cb);
esp_err_t esp_ble_gatts_app_register(uint16_t app_id);
esp_err_t esp_ble_gatts_create_attr_tab(const esp_gatts_attr_db_t *gatts_attr_db,
                                        esp_gatt_if_t gatts_if, uint16_t max_nb_attr, uint8_t srvc_inst_id);
esp_err_t esp_ble_gatts_start_service(uint16_t service_handle);
esp_err_t esp_ble_gatts_send_indicate(esp_gatt_if_t gatts_if, uint16_t conn_id, uint16_t attr_handle,
                                      uint16_t value_len, uint8_t *value, bool need_confirm);
esp_err_t esp_ble_gatts_send_response(esp_gatt_if_t gatts_if, uint16_t conn_id, uint32_t trans_id,
                                      esp_gatt_status_t status, esp_gatt_rsp_t *rsp);
/* esp_gatt_common_api.h —— MTU（client/server 通用） */
esp_err_t esp_ble_gatt_set_local_mtu(uint16_t mtu);   /* 调用须在 enable 之后、连接之前 */
```

关键事件链：`ESP_GATTS_REG_EVT`（app 注册成功）→ 设名+配广播+建表 → `ESP_GATTS_CREAT_ATTR_TAB_EVT`（建表成功）→ `esp_ble_gatts_start_service` → 客户端连入触发 `ESP_GATTS_CONNECT_EVT`。

## 常见错误

| 错误 | 原因 | 解决方法 |
|---|---|---|
| `esp_bluedroid_init` 返回非 OK | 控制器未先 init+enable | 严格按 controller init → enable → bluedroid init_with_cfg → enable 顺序 |
| 用了 NimBLE 的 API | Host 选错（`CONFIG_BT_NIMBLE_ENABLED`） | Bluedroid 用 `esp_ble_gatts_*`；NimBLE 用 `ble_*`。本 recipe 仅适用 `CONFIG_BT_BLUEDROID_ENABLED=y` |
| 客户端写 CCC 后收不到 notify | 未调 `esp_ble_gatts_send_indicate` 或 handle 取错 | 在 `ESP_GATTS_WRITE_EVT` 检测到 CCC 写入后立即 send_indicate；handle 用 `heart_rate_handle_table[IDX_CHAR_VAL_A]` |
| `set_local_mtu` 报错 | 在连接后才调 | 必须在 `esp_bluedroid_enable` 之后、任何连接之前调 |
| 编译报 `esp_ble_gatts_*` undefined | 未启用 Bluedroid 或未加 `bt` 组件 | menuconfig 开 `Bluetooth → Bluedroid`；`idf_component_register` 加 `bt` 到 `REQUIRES` |
| `esp_bt_controller_mem_release(CLASSIC_BT)` 崩溃 | esp32 双模需 Classic | 只在纯 BLE（不要 Classic）时 release；双模去掉这行 |
| `need_rsp` 写请求客户端超时 | 未回 response | `param->write.need_rsp==true` 时必须 `esp_ble_gatts_send_response(..., ESP_GATT_OK, NULL)` |

## 参考

- `examples/bluetooth/bluedroid/ble/gatt_server_service_table` — 属性表式 GATT Server（本 recipe 主要来源）
- `examples/bluetooth/bluedroid/ble/gatt_server` — 逐个 `esp_ble_gatts_add_char` 式（旧式）
- `examples/bluetooth/ble_get_started/bluedroid/Bluedroid_GATT_Server` — 分步教学版
- ESP-IDF `components/bt/host/bluedroid/api/include/api/esp_gatts_api.h`、`esp_gap_ble_api.h`、`esp_bt_main.h`、`components/bt/include/esp32/include/esp_bt.h`
- 文档 `docs/en/api-reference/bluetooth/esp_gatts.rst`、`esp_bt_main.rst`、`esp_gap_ble.rst`
