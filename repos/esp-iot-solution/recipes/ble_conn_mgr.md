# BLE 连接管理 esp_ble_conn_mgr

> **适用摘要**: 使用 `ble_conn_mgr` 组件的简化 API（`esp_ble_conn_init` / `esp_ble_conn_start`）搭建 BLE 外设/中心角色，覆盖周期广播（periodic advertising）、周期同步（periodic sync）、SPP 串口透传、L2CAP CoC 信道。组件基于 NimBLE，用 `esp_ble_conn_config_t` 配设备名/广播数据/扩展广播/周期广播，事件经 `esp_event` 投递到 `BLE_CONN_MGR_EVENTS`。

## 触发意图

- "BLE 外设 / 中心"
- "esp_ble_conn_mgr"
- "BLE 周期广播 / periodic advertising"
- "BLE 周期同步 / periodic sync"
- "BLE SPP 串口透传"
- "L2CAP CoC 信道"

## 前置条件

| 条件 | 要求 |
|---|---|
| 目标芯片 | ESP32 / ESP32-C2 / ESP32-C3 / ESP32-S3（带 BLE 的芯片） |
| IDF 环境 | ESP-IDF v5.3+（master），启用 `CONFIG_BT_ENABLED` + `CONFIG_BT_NIMBLE_ENABLED` |
| 组件依赖 | `espressif/ble_conn_mgr` |
| menuconfig | 选角色 `CONFIG_BLE_CONN_MGR_ROLE_PERIPHERAL` / `_CENTRAL` / `_BOTH`；扩展广播/周期同步等特性按需开启 |
| 参考示例 | `examples/bluetooth/ble_conn_mgr/ble_periodic_adv`、`ble_periodic_sync`、`ble_spp`、`examples/bluetooth/ble_l2cap_coc/*` |

## 分步说明

### 1. 添加组件依赖并选角色

```bash
idf.py add-dependency "espressif/ble_conn_mgr"
```

在 `menuconfig → BLE Connection Management` 选角色（默认 peripheral）：

- `CONFIG_BLE_CONN_MGR_ROLE_PERIPHERAL`（0）= 外设
- `CONFIG_BLE_CONN_MGR_ROLE_CENTRAL`（1）= 中心
- `CONFIG_BLE_CONN_MGR_ROLE_BOTH`（2）= 同时支持

周期广播需 `CONFIG_BLE_CONN_MGR_EXTENDED_ADV=y` + `CONFIG_BLE_CONN_MGR_PERIODIC_ADV=y`（依赖 `SOC_BLE_50_SUPPORTED`）。
周期同步需中心角色 + `CONFIG_BLE_CONN_MGR_PERIODIC_SYNC=y`。

### 2. 最小外设初始化（init → start）

`esp_ble_conn_config_t.device_name` 为广播名（≤ `MAX_BLE_DEVNAME_LEN`=29 字节），`broadcast_data` 为厂商数据（≤ `BROADCAST_PARAM_LEN`=15 字节）。

```c
#include "esp_ble_conn_mgr.h"
#include "nvs_flash.h"

void app_main(void)
{
    esp_ble_conn_config_t config = {
        .device_name   = CONFIG_EXAMPLE_BLE_ADV_NAME,    // 例 "ESP_BLE"
        .broadcast_data = CONFIG_EXAMPLE_BLE_SUB_ADV,    // 例 "SUB_ADV"
    };

    esp_err_t ret = nvs_flash_init();
    if (ret == ESP_ERR_NVS_NO_FREE_PAGES || ret == ESP_ERR_NVS_NEW_VERSION_FOUND) {
        ESP_ERROR_CHECK(nvs_flash_erase());
        ret = nvs_flash_init();
    }
    ESP_ERROR_CHECK(ret);

    ESP_ERROR_CHECK(esp_ble_conn_init(&config));
    if (esp_ble_conn_start() != ESP_OK) {
        esp_ble_conn_stop();
        esp_ble_conn_deinit();
    }
}
```

### 3. 监听 BLE 事件（esp_event）

事件 base 为 `BLE_CONN_MGR_EVENTS`，`event_data` 是 `esp_ble_conn_event_data_t *`（联合体按事件 id 取成员）。

```c
#include "esp_event.h"

static void app_ble_conn_event_handler(void *arg, esp_event_base_t base,
                                       int32_t id, void *event_data)
{
    if (base != BLE_CONN_MGR_EVENTS) return;

    const esp_ble_conn_event_data_t *d = event_data;
    switch (id) {
    case ESP_BLE_CONN_EVENT_STARTED:
        ESP_LOGI(TAG, "BLE started"); break;
    case ESP_BLE_CONN_EVENT_CONNECTED:
        ESP_LOGI(TAG, "CONNECTED conn=%u peer=" BLE_CONN_MGR_ADDR_STR,
                 d->connected.conn_handle,
                 BLE_CONN_MGR_ADDR_HEX(d->connected.peer_addr));
        break;
    case ESP_BLE_CONN_EVENT_DISCONNECTED:
        ESP_LOGI(TAG, "DISCONNECTED conn=%u reason=0x%04x",
                 d->disconnected.conn_handle, d->disconnected.reason);
        break;
    case ESP_BLE_CONN_EVENT_DATA_RECEIVE:
        // 外设收到 write、或中心收到 notify/indicate（uuid_fn 为 NULL 时走这里）
        // 注意：d->data_receive.data 是堆分配的，处理完须 free()
        if (d->data_receive.data) {
            ESP_LOG_BUFFER_HEX(TAG, d->data_receive.data, d->data_receive.data_len);
            free(d->data_receive.data);
        }
        break;
    default: break;
    }
}

// app_main 中（init 之前）：
esp_event_loop_create_default();
esp_event_handler_register(BLE_CONN_MGR_EVENTS, ESP_EVENT_ANY_ID,
                           app_ble_conn_event_handler, NULL);
```

### 4. 注册自定义 GATT 服务（SPP 示例）

用 `esp_ble_conn_svc_t` 描述服务，`esp_ble_conn_character_t` 描述每个特征值（含 `uuid_fn` 回调，inbuf=NULL=读，非 NULL=写）。`esp_ble_conn_add_svc` 必须在 `esp_ble_conn_start()` 之前调用。

```c
#define BLE_SVC_SPP_UUID16       0xABF0
#define BLE_SVC_SPP_CHR_UUID16   0xABF1

static esp_err_t spp_chr_cb(const uint8_t *inbuf, uint16_t inlen,
                            uint8_t **outbuf, uint16_t *outlen,
                            void *priv_data, uint8_t *att_status)
{
    if (!outbuf || !outlen) { *att_status = ESP_IOT_ATT_INTERNAL_ERROR; return ESP_ERR_INVALID_ARG; }
    if (!inbuf) {                       // 读
        *outlen = strlen("SPP_CHR");
        *outbuf = (uint8_t *)strndup("SPP_CHR", *outlen);
    } else {                            // 写
        *outbuf = calloc(1, inlen);
        memcpy(*outbuf, inbuf, inlen);
        *outlen = inlen;
    }
    *att_status = ESP_IOT_ATT_SUCCESS;
    return ESP_OK;
}

static const esp_ble_conn_character_t spp_chrs[] = {
    { "spp_chr", BLE_CONN_UUID_TYPE_16,
      BLE_CONN_GATT_CHR_READ | BLE_CONN_GATT_CHR_WRITE |
      BLE_CONN_GATT_CHR_NOTIFY | BLE_CONN_GATT_CHR_INDICATE,
      { .uuid16 = BLE_SVC_SPP_CHR_UUID16 }, spp_chr_cb },
};

static const esp_ble_conn_svc_t spp_svc = {
    .type = BLE_CONN_UUID_TYPE_16,
    .uuid = { .uuid16 = BLE_SVC_SPP_UUID16 },
    .nu_lookup_count = sizeof(spp_chrs) / sizeof(spp_chrs[0]),
    .nu_lookup = (esp_ble_conn_character_t *)spp_chrs,
};

// init 之后、start 之前：
ESP_ERROR_CHECK(esp_ble_conn_add_svc(&spp_svc));
```

### 5. 主动 notify / read / write（peripheral 或 central）

```c
esp_ble_conn_data_t inbuff = {
    .type = BLE_CONN_UUID_TYPE_16,
    .uuid = { .uuid16 = BLE_SVC_SPP_CHR_UUID16 },
    .data = (uint8_t []){ 0x5A },
    .data_len = 1,
};
esp_ble_conn_notify(&inbuff);          // 外设发 notify（用默认连接）
// 多连接时用 esp_ble_conn_notify_by_handle(conn_handle, &inbuff);
// 中心读写：esp_ble_conn_read(&outbuf) / esp_ble_conn_write(&inbuff)
```

### 6. 周期广播（periodic advertising）— 外设侧

需 `CONFIG_BLE_CONN_MGR_EXTENDED_ADV=y` + `CONFIG_BLE_CONN_MGR_PERIODIC_ADV=y`。在 config 里填 `extended_adv_data` / `periodic_adv_data`：

```c
static uint8_t ext_adv_pattern[] = {
    0x02, 0x01, 0x06,
    0x15, 0x09, 'E','S','P','_','P','E','R','I','O','D','I','C','_','A','D','V','_','E','X','T',
};

esp_ble_conn_config_t config = {
    .device_name       = CONFIG_EXAMPLE_BLE_ADV_NAME,
    .broadcast_data    = CONFIG_EXAMPLE_BLE_SUB_ADV,
    .extended_adv_data   = (const char *)ext_adv_pattern,
    .extended_adv_len    = sizeof(ext_adv_pattern),
    .periodic_adv_data   = (const char *)ext_adv_pattern_1,
    .periodic_adv_len    = sizeof(ext_adv_pattern_1),
};
// 之后同 init → start
```

运行时也可用 `esp_ble_conn_periodic_adv_data_set(data, len)` 单独更新周期广播数据。

### 7. 周期同步（periodic sync）— 中心侧

需中心角色 + `CONFIG_BLE_CONN_MGR_PERIODIC_SYNC=y`。事件 `ESP_BLE_CONN_EVENT_PERIODIC_SYNC` / `_PERIODIC_REPORT` / `_PERIODIC_SYNC_LOST` 给出数据：

```c
case ESP_BLE_CONN_EVENT_PERIODIC_SYNC: {
    esp_ble_conn_periodic_sync_t *s = &d->periodic_sync;
    ESP_LOGI(TAG, "sync status=%d handle=%d sid=%d phy=%d interval=%d",
             s->status, s->sync_handle, s->sid, s->adv_phy, s->per_adv_ival);
    break;
}
case ESP_BLE_CONN_EVENT_PERIODIC_REPORT: {
    esp_ble_conn_periodic_report_t *r = &d->periodic_report;
    ESP_LOGI(TAG, "report handle=%d rssi=%d len=%d", r->sync_handle, r->rssi, r->data_length);
    ESP_LOG_BUFFER_HEX(TAG, r->data, r->data_length);
    break;
}
case ESP_BLE_CONN_EVENT_PERIODIC_SYNC_LOST: {
    ESP_LOGI(TAG, "sync lost handle=%d reason=%d",
             d->periodic_sync_lost.sync_handle, d->periodic_sync_lost.reason);
    break;
}
```

### 8. L2CAP Connection-Oriented Channels（高速流式传输）

先 `esp_ble_conn_l2cap_coc_mem_init()` 初始化 SDU 内存池，连接建立后在 `ESP_BLE_CONN_EVENT_CONNECTED` 里建 server（外设）或主动 connect（中心）。事件经独立回调 `esp_ble_conn_l2cap_coc_event_cb_t`（不是 `BLE_CONN_MGR_EVENTS`）。

```c
#define L2CAP_COC_PSM   CONFIG_EXAMPLE_L2CAP_COC_PSM      // 0x0001..0x00FF
#define L2CAP_COC_MTU   CONFIG_BLE_CONN_MGR_L2CAP_COC_MTU // 默认 512

static int coc_evt_cb(esp_ble_conn_l2cap_coc_event_t *evt, void *arg)
{
    switch (evt->type) {
    case ESP_BLE_CONN_L2CAP_COC_EVENT_ACCEPT:
        // 接受对端连接，提供接收缓冲
        return esp_ble_conn_l2cap_coc_accept(evt->accept.conn_handle,
                                             evt->accept.peer_sdu_size,
                                             evt->accept.chan);
    case ESP_BLE_CONN_L2CAP_COC_EVENT_DATA_RECEIVED:
        // 注意：evt->receive.sdu.data 回调返回后立即释放，需异步处理要先拷贝
        esp_ble_conn_l2cap_coc_recv_ready(evt->receive.chan, L2CAP_COC_MTU);
        return 0;
    case ESP_BLE_CONN_L2CAP_COC_EVENT_CONNECTED:
        if (evt->connect.status == 0) {
            // 发一帧
            uint8_t buf[L2CAP_COC_MTU];
            esp_ble_conn_l2cap_coc_sdu_t sdu = { .data = buf, .len = sizeof(buf) };
            esp_ble_conn_l2cap_coc_send(evt->connect.chan, &sdu);
        }
        return 0;
    default: return 0;
    }
}

// 外设：在 ESP_BLE_CONN_EVENT_CONNECTED 里
esp_ble_conn_l2cap_coc_create_server(L2CAP_COC_PSM, L2CAP_COC_MTU, coc_evt_cb, NULL);

// 中心：已知 conn_handle 时主动连接
// esp_ble_conn_l2cap_coc_connect(conn_handle, L2CAP_COC_PSM, L2CAP_COC_MTU,
//                                peer_sdu_size, coc_evt_cb, NULL);

// app_main 里（start 之前）：
ESP_ERROR_CHECK(esp_ble_conn_l2cap_coc_mem_init());
```

## 关键事件枚举（esp_ble_conn_event_t）

| 事件 | 含义 |
|---|---|
| `ESP_BLE_CONN_EVENT_STARTED` | BLE 启动完成 |
| `ESP_BLE_CONN_EVENT_STOPPED` | BLE 停止 |
| `ESP_BLE_CONN_EVENT_CONNECTED` | 新连接建立（取 `connected`） |
| `ESP_BLE_CONN_EVENT_DISCONNECTED` | 连接断开（取 `disconnected`） |
| `ESP_BLE_CONN_EVENT_DATA_RECEIVE` | notify/indicate/peripheral write（uuid_fn NULL 时） |
| `ESP_BLE_CONN_EVENT_PERIODIC_SYNC` / `_REPORT` / `_SYNC_LOST` | 周期同步系列 |
| `ESP_BLE_CONN_EVENT_CCCD_UPDATE` | CCCD 被写（取 `cccd_update`） |
| `ESP_BLE_CONN_EVENT_MTU` | MTU 协商完成（取 `mtu_update`） |
| `ESP_BLE_CONN_EVENT_SCAN_RESULT` | 中心扫描结果（取 `scan_result`） |
| `ESP_BLE_CONN_EVENT_ENC_CHANGE` | 加密/配对完成（取 `enc_change`） |

## 常见错误

| 错误 | 原因 | 解决方法 |
|---|---|---|
| `esp_ble_conn_start` 返回非 ESP_OK | 角色未在 menuconfig 开启（如中心功能没开却调中心 API） | 确认 `CONFIG_BLE_CONN_MGR_ROLE_*` 与所用 API 匹配 |
| 周期广播无数据 | 未启用扩展+周期广播 | 开 `CONFIG_BLE_CONN_MGR_EXTENDED_ADV` + `_PERIODIC_ADV`；芯片须 `SOC_BLE_50_SUPPORTED` |
| 周期同步不触发 | 用了 peripheral 角色或未开 `_PERIODIC_SYNC` | 改中心角色并开 `CONFIG_BLE_CONN_MGR_PERIODIC_SYNC=y` |
| `esp_ble_conn_add_svc` 失败 | 在 `esp_ble_conn_start()` 之后才注册 | 服务必须在 start 之前 add |
| notify 收不到 | 未先开 CCCD 或 MTU 太小 | 对端订阅后再 notify；menuconfig 调大 `CONFIG_BT_NIMBLE_ATT_PREFERRED_MTU` |
| L2CAP CoC 收数据后崩溃 | 回调里保存了 `sdu.data` 指针 | 数据回调返回即释放；异步处理须先 `memcpy` |
| 多连接写错连接 | 用了 `esp_ble_conn_write` 而非按 handle 版 | 多连接用 `esp_ble_conn_*_by_handle(conn_handle, ...)` |
| NVS 报错 | 未初始化 NVS | `app_main` 开头先 `nvs_flash_init()`（失败 erase 重试） |

## 参考项目

- 组件头文件：`components/bluetooth/ble_conn_mgr/include/esp_ble_conn_mgr.h`
- 组件 Kconfig：`components/bluetooth/ble_conn_mgr/Kconfig`
- 周期广播示例：`examples/bluetooth/ble_conn_mgr/ble_periodic_adv`
- 周期同步示例：`examples/bluetooth/ble_conn_mgr/ble_periodic_sync`
- SPP 服务端示例：`examples/bluetooth/ble_conn_mgr/ble_spp/spp_server`
- SPP 客户端示例：`examples/bluetooth/ble_conn_mgr/ble_spp/spp_client`
- L2CAP CoC 外设：`examples/bluetooth/ble_l2cap_coc/l2cap_coc_peripheral`
- L2CAP CoC 中心：`examples/bluetooth/ble_l2cap_coc/l2cap_coc_central`
- 在线文档：`docs/en/bluetooth/ble_conn_mgr.rst`
