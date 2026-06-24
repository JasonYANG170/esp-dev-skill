# BTHome 协议（Home Assistant 集成）

> **适用摘要**: 使用 `bthome_v2` 组件实现 BTHome V2 协议，对接 Home Assistant。支持加密/非加密、传感器/二元传感器/事件数据上报与解析：广播端用 `bthome_make_adv_data` + `bthome_payload_adv_add_*` 拼广播包；接收端用 `bthome_create` + `bthome_set_encrypt_key` + `bthome_parse_adv_data` 解出 `bthome_reports_t`。配合 `ble_hci` 组件做底层 BLE 收发。

## 触发意图

- "BTHome"
- "Home Assistant 集成"
- "bthome_v2"
- "bthome_parse_adv_data"
- "ESP32 智能家居广播"
- "BTHome 加密广播"

## 前置条件

| 条件 | 要求 |
|---|---|
| 目标芯片 | ESP32 / ESP32-C2 / ESP32-C3 / ESP32-S3（带 BLE 的芯片） |
| IDF 环境 | ESP-IDF v5.3+，启用 `CONFIG_BT_ENABLED` + `CONFIG_BT_NIMBLE_ENABLED`（或用 `ble_hci` 直接 HCI） |
| 组件依赖 | `espressif/bthome_v2`、`espressif/ble_hci`（底层 BLE） |
| Home Assistant | HA ≥ 含 [BTHome 集成](https://bthome.io/) 的版本；加密需在 HA 与设备两端配同 key + 对端 MAC |
| 参考示例 | `examples/bluetooth/ble_adv/bthome/bulb`（接收端，控制 WS2812）、`examples/bluetooth/ble_adv/bthome/dimmer`（广播端，按键+旋钮） |

## 分步说明

### 1. 添加组件依赖

```bash
idf.py add-dependency "espressif/bthome_v2"
idf.py add-dependency "espressif/ble_hci"
```

### 2. 接收端（解析对端 BTHome 广播）

创建 handle → 注册存储回调（加密计数器需持久化，防重放）→ 设加密 key + 对端 MAC → 在扫描回调里 `bthome_parse_adv_data`。

```c
#include "bthome_v2.h"
#include "ble_hci.h"
#include "nvs_flash.h"

static const uint8_t encrypt_key[] = {
    0x23, 0x1d, 0x39, 0xc1, 0xd7, 0xcc, 0x1a, 0xb1,
    0xae, 0xe2, 0x24, 0xcd, 0x09, 0x6d, 0xb9, 0x32
};
static const uint8_t peer_mac[] = { 0x54, 0x48, 0xE6, 0x8F, 0x80, 0xA5 };

// 加密需要的 last-counter 存取（防重放），回调由组件调用
static void settings_store(bthome_handle_t h, const char *key,
                           const uint8_t *data, uint8_t len) {
    nvs_handle_t nh;
    if (nvs_open("storage", NVS_READWRITE, &nh) != ESP_OK) return;
    nvs_set_blob(nh, key, data, len);
    nvs_commit(nh);
    nvs_close(nh);
}
static void settings_load(bthome_handle_t h, const char *key,
                          uint8_t *data, uint8_t len) {
    nvs_handle_t nh;
    if (nvs_open("storage", NVS_READWRITE, &nh) != ESP_OK) { memset(data, 0, len); return; }
    size_t need = len;
    if (nvs_get_blob(nh, key, data, &need) != ESP_OK) memset(data, 0, len);
    nvs_close(nh);
}

// ble_hci 扫描结果回调
static void ble_hci_scan_cb(ble_hci_scan_result_t *scan_result, uint16_t result_len) {
    for (int i = 0; i < result_len; i++) {
        // 把每条结果丢队列或直接解析，下面用同步示例
        bthome_reports_t *r = bthome_parse_adv_data(bthome_recv,
                                  scan_result[i].ble_adv,
                                  scan_result[i].adv_data_len);
        if (r == NULL) continue;
        for (int j = 0; j < r->num_reports; j++) {
            // r->report[j].id / .len / .data
            ESP_LOGI(TAG, "report id=0x%02x len=%d", r->report[j].id, r->report[j].len);
        }
        bthome_free_reports(r);   // 必须释放
    }
}

bthome_handle_t bthome_recv;   // 全局，便于回调访问

void app_main(void) {
    ESP_ERROR_CHECK(nvs_flash_init());   // 失败 erase 重试略

    // 1) ble_hci 底层扫描
    ble_hci_init();
    ble_hci_scan_param_t sp = {
        .scan_type       = BLE_SCAN_TYPE_PASSIVE,
        .scan_interval   = 0x50, .scan_window = 0x50,
        .own_addr_type   = BLE_ADDR_TYPE_PUBLIC,
        .filter_policy   = ADV_FILTER_ALLOW_SCAN_WLST_CON_ANY,
    };
    ble_hci_set_scan_param(&sp);
    ble_hci_set_register_scan_callback(ble_hci_scan_cb);
    ble_hci_addr_t peer = {0};
    memcpy((uint8_t *)peer, peer_mac, BLE_HCI_ADDR_LEN);
    ble_hci_add_to_accept_list(peer, BLE_ADDR_TYPE_RANDOM);   // 只收对端
    ble_hci_set_scan_enable(true, true);

    // 2) bthome 解析器
    ESP_ERROR_CHECK(bthome_create(&bthome_recv));
    bthome_callbacks_t cbs = { .store = settings_store, .load = settings_load };
    ESP_ERROR_CHECK(bthome_register_callbacks(bthome_recv, &cbs));
    ESP_ERROR_CHECK(bthome_set_encrypt_key(bthome_recv, encrypt_key));
    ESP_ERROR_CHECK(bthome_set_peer_mac_addr(bthome_recv, peer_mac));
}
```

### 3. 解析 report.id（传感器 / 二元 / 事件）

`report[i].id` 取值见 `bthome_sensor_id_t`（如 `BTHOME_SENSOR_ID_TEMPERATURE`=0x45、`BTHOME_SENSOR_ID_HUMIDITY`=0x2E）、`bthome_bin_sensor_id_t`（如 `BTHOME_BIN_SENSOR_ID_MOTION`=0x21）、`bthome_event_id_t`（`BTHOME_EVENT_ID_BUTTON`=0x3A、`BTHOME_EVENT_ID_DIMMER`=0x3C）。

```c
for (int j = 0; j < r->num_reports; j++) {
    uint8_t id = r->report[j].id;
    if (id == BTHOME_EVENT_ID_BUTTON) {
        // data[0]: 1=click 短按, 2=double, ...
        if (r->report[j].data[0] == 0x01) ESP_LOGI(TAG, "button click");
    } else if (id == BTHOME_EVENT_ID_DIMMER) {
        // data[0]: 1=左旋 2=右旋；data[1]: 步数
    } else if (id == BTHOME_SENSOR_ID_TEMPERATURE) {
        // 解析按 BTHome 格式（有符号/无符号 little-endian，见 bthome.io/format）
    }
}
```

### 4. 广播端（把传感器/事件发出去）

用 `bthome_payload_adv_add_sensor_data` / `_bin_sensor_data` / `_evt_data` 拼 payload，再 `bthome_make_adv_data` 包成完整广播数据，交 `ble_hci_set_adv_data` + `ble_hci_set_adv_enable`。

```c
bthome_handle_t bthome_tx;

void send_event(uint8_t btn_evt_id, uint8_t *dim_evt, uint8_t dim_len) {
    uint8_t payload[31], adv[31];
    uint8_t plen = 0;

    plen = bthome_payload_adv_add_evt_data(payload, plen,
            BTHOME_EVENT_ID_BUTTON, &btn_evt_id, 1);
    plen = bthome_payload_adv_add_evt_data(payload, plen,
            BTHOME_EVENT_ID_DIMMER, dim_evt, dim_len);

    bthome_device_info_t info = {
        .bit.bthome_version   = 2,
        .bit.encryption_flag  = true,        // 与 key 配套
        .bit.trigger_based_flag = 0,
    };
    uint8_t name[] = { 'D','I','Y' };
    uint8_t adv_len = bthome_make_adv_data(bthome_tx, adv, name, sizeof(name),
                                           info, payload, plen);
    if (adv_len == 0) { ESP_LOGE(TAG, "build adv fail"); return; }

    ble_hci_set_adv_data(adv_len, adv);
    ble_hci_set_adv_enable(true);
    // 定时关闭广播（事件型），用 xTimer 在 300ms 后 set_adv_enable(false)
}

// app_main：
ESP_ERROR_CHECK(bthome_create(&bthome_tx));
bthome_callbacks_t cbs = { .store = settings_store, .load = settings_load };
ESP_ERROR_CHECK(bthome_register_callbacks(bthome_tx, &cbs));
ESP_ERROR_CHECK(bthome_set_encrypt_key(bthome_tx, encrypt_key));
ESP_ERROR_CHECK(bthome_set_local_mac_addr(bthome_tx, (uint8_t *)local_mac));
ESP_ERROR_CHECK(bthome_load_params(bthome_tx));   // 读回加密计数器
```

传感器数据拼接：

```c
uint8_t temp_le16[2] = { 0xC8, 0x00 };   // 例 20.0°C *100 = 2000 (0x07D0), 按格式
plen = bthome_payload_adv_add_sensor_data(payload, plen,
        BTHOME_SENSOR_ID_TEMPERATURE, temp_le16, 2);

uint8_t bat_pct = 88;
plen = bthome_payload_adv_add_sensor_data(payload, plen,
        BTHOME_SENSOR_ID_BATTERY, &bat_pct, 1);

plen = bthome_payload_adv_add_bin_sensor_data(payload, plen,
        BTHOME_BIN_SENSOR_ID_MOTION, 1);   // 1 = 触发
```

> 广播包总长 ≤ 31 字节（含 name + device_info + payload + 加密头/计数器/MIC），超长会被截断。事件型设备发完关广播省电；周期型传感器定时广播。

## 关键枚举（节选）

| 类别 | 宏示例 | id 值 |
|---|---|---|
| 传感器 | `BTHOME_SENSOR_ID_BATTERY` | 0x01 |
|  | `BTHOME_SENSOR_ID_TEMPERATURE` | 0x45 |
|  | `BTHOME_SENSOR_ID_HUMIDITY` | 0x2E |
|  | `BTHOME_SENSOR_ID_ILLUMINANCE` | 0x05 |
|  | `BTHOME_SENSOR_ID_CO2` | 0x12 |
| 二元 | `BTHOME_BIN_SENSOR_ID_MOTION` | 0x21 |
|  | `BTHOME_BIN_SENSOR_ID_DOOR` | 0x1A |
|  | `BTHOME_BIN_SENSOR_ID_OCCUPANCY` | 0x23 |
| 事件 | `BTHOME_EVENT_ID_BUTTON` | 0x3A |
|  | `BTHOME_EVENT_ID_DIMMER` | 0x3C |

## 常见错误

| 错误 | 原因 | 解决方法 |
|---|---|---|
| HA 收不到设备 | 广播未启用或 MAC 不匹配 | 确认 `ble_hci_set_adv_enable(true)`；HA 端 MAC 与设备一致 |
| 解析返回 NULL | 加密 key/MAC 不一致或非 BTHome 包 | key 16 字节两端相同；广播端用 `RANDOM` 地址时 HA 端也用该地址 |
| 重放攻击误判 | 没持久化加密计数器 | 注册 store/load 回调写 NVS；`bthome_load_params` 启动时读回 |
| 广播包被截断 | payload + name + 加密开销 > 31B | 拆成多个广播或精简 name；事件型可省 name |
| `bthome_free_reports` 漏调 | 内存泄漏 | 解析后一定 `bthome_free_reports(r)` |
| 解析出乱码 id | 拿错字节当 id | `r->report[i].id` 是 object id，按 sensor/bin/event 枚举判 |
| 加密广播对端收不到 | 用了 PUBLIC 地址但 key 绑 RANDOM | 广播端/接收端地址类型要一致（示例 dimmer 用 `BLE_ADDR_TYPE_RANDOM`） |
| 扫描不到 | filter 太严或未加白名单 | 接收端 `ble_hci_add_to_accept_list(peer, BLE_ADDR_TYPE_RANDOM)` |

## 参考项目

- 组件头文件：`components/bluetooth/ble_adv/bthome/include/bthome_v2.h`
- 接收端示例（bulb 控制 WS2812）：`examples/bluetooth/ble_adv/bthome/bulb/main/app_main.c`
- 广播端示例（dimmer，按键+旋钮+加密）：`examples/bluetooth/ble_adv/bthome/dimmer/main/app_main.c`
- 在线文档：`docs/en/bluetooth/bt_home.rst`
- 协议格式：https://bthome.io/format/
