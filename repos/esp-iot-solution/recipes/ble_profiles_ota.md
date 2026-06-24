# BLE GATT Profiles 与服务（OTA 固件升级、HTP 健康体温）

> **适用摘要**: 在 `esp_ble_conn_mgr` 之上使用 `ble_profiles` 组件的��准 GATT 服务与 profile：`esp_ble_ota_raw` 提供基于扇区 CRC 校验的 BLE OTA 固件升级（服务 UUID `0x8018`）；`esp_ble_htp` 提供健康体温计 profile（服务 UUID `0x1809`）。profile 内部注册服务和特征值，应用只需 init + 注册回调。

## 触发意图

- "BLE OTA 固件升级"
- "esp_ble_ota_raw"
- "ble_ota_raw profile"
- "BLE 健康体温计 / Health Thermometer"
- "esp_ble_htp"
- "BLE GATT 标准服务"

## 前置条件

| 条件 | 要求 |
|---|---|
| 目标芯片 | ESP32 / ESP32-C2 / ESP32-C3 / ESP32-S3（带 BLE 的芯片） |
| IDF 环境 | ESP-IDF v5.3+ |
| 组件依赖 | `espressif/ble_conn_mgr` + `espressif/ble_profiles`（或对应子组件） |
| 分区表（OTA） | 自定义 `partitions.csv` 含 `ota_0` / `ota_1` / `otadata` |
| menuconfig（OTA） | `CONFIG_BT_NIMBLE_ATT_PREFERRED_MTU=498`、`CONFIG_BLE_OTA_RAW_PROFILE=y` |
| 参考示例 | `examples/bluetooth/ble_profiles/ble_ota`、`examples/bluetooth/ble_profiles/ble_htp` |

## 分步说明

### 1. 添加组件依赖

```bash
idf.py add-dependency "espressif/ble_conn_mgr"
idf.py add-dependency "espressif/ble_profiles"
```

### 2. OTA — 分区表与 sdkconfig.defaults

OTA 需双 OTA 分区。`examples/bluetooth/ble_profiles/ble_ota/partitions.csv`：

```csv
# Name,   Type, SubType, Offset,  Size, Flags
nvs,      data, nvs,     ,        0x4000,
otadata,  data, ota,     ,        0x2000,
phy_init, data, phy,     ,        0x1000,
ota_0,    app,  ota_0,   ,        1500K,
ota_1,    app,  ota_1,   ,        1500K,
```

`sdkconfig.defaults` 关键项：

```
CONFIG_BT_ENABLED=y
CONFIG_BT_NIMBLE_ENABLED=y
CONFIG_BLE_OTA_RAW_PROFILE=y
CONFIG_BT_NIMBLE_ATT_PREFERRED_MTU=498
CONFIG_ESPTOOLPY_FLASHSIZE_4MB=y
CONFIG_PARTITION_TABLE_CUSTOM=y
CONFIG_PARTITION_TABLE_CUSTOM_FILENAME="partitions.csv"
```

### 3. OTA — 初始化 ble_conn_mgr + ota_raw profile

`esp_ble_ota_raw_init()` 注册 OTA 服务（`0x8018`）和 DIS 服务，**ble_conn_mgr 的 init/start 由应用自己完成**。流程：建 ringbuf → conn init → ota_raw init → 注册固件回调 → 启 OTA worker → conn start。

```c
#include "esp_ble_conn_mgr.h"
#include "esp_ble_ota_raw.h"
#include "ble_ota_raw.h"     // 示例辅助 API（ringbuf / task）

#define BLE_OTA_RAW_ADV_UUID16 0x8018

static const uint8_t s_manu_data[] = {
    0x0b, 0xff, 0x01, 0x01, 0x27, 0x95, 0x01, 0x00, 0x00, 0x00, 0xff, 0xff
};

// 固件扇区到达回调（一个 4096 字节扇区通过 CRC 后调用一次）
static void app_recv_fw_cb(uint8_t *buf, uint32_t length)
{
    if (!ble_ota_raw_recv_fw_cb((const uint8_t *)buf, length)) {
        ESP_LOGE(TAG, "Failed to queue firmware chunk");
    }
}

void app_main(void)
{
    esp_ble_conn_config_t config = {0};
    strlcpy((char *)config.device_name, CONFIG_EXAMPLE_BLE_OTA_RAW_DEVICE_NAME,
            sizeof(config.device_name));
    memcpy(config.broadcast_data, s_manu_data, sizeof(s_manu_data));
    config.include_service_uuid = 1;
    config.adv_uuid_type = BLE_CONN_UUID_TYPE_16;
    config.adv_uuid16 = BLE_OTA_RAW_ADV_UUID16;

    ESP_ERROR_CHECK(nvs_flash_init());   // 失败时 erase 重试略

    // 1) 固件环形缓冲（0 = 用 BLE_OTA_RAW_RINGBUF_DEFAULT_SIZE = 12KB）
    if (!ble_ota_raw_ringbuf_init(BLE_OTA_RAW_RINGBUF_DEFAULT_SIZE)) {
        ESP_LOGE(TAG, "ringbuf init fail"); return;
    }

    esp_event_loop_create_default();
    esp_event_handler_register(BLE_CONN_MGR_EVENTS, ESP_EVENT_ANY_ID,
                               app_ble_conn_event_handler, NULL);

    ESP_ERROR_CHECK(esp_ble_conn_init(&config));
    ESP_ERROR_CHECK(esp_ble_ota_raw_init());                              // 注册 0x8018 服务
    ESP_ERROR_CHECK(esp_ble_ota_raw_recv_fw_data_callback(app_recv_fw_cb));// 固件回调
    if (!ble_ota_raw_task_init()) {                                       // OTA 写 flash 任务
        ESP_LOGE(TAG, "ota task init fail"); return;
    }
    ESP_ERROR_CHECK(esp_ble_conn_start());
}
```

### 4. OTA — 可选 flash begin 钩子与发送窗口

```c
// 可选：在 START 命令到达时调用 esp_ota_begin（默认实现已含；自定义时注册）
esp_ble_ota_raw_set_ota_begin_cb(my_ota_begin_hook);

// 可选：按 ringbuf 容量推导每扇区发送窗口（ringbuf/4096，clamp 1..64）
esp_ble_ota_raw_set_sector_send_window_for_ringbuf(BLE_OTA_RAW_RINGBUF_DEFAULT_SIZE);

// 查询固件总长度（START 前为 UINT32_MAX）
uint32_t total = esp_ble_ota_raw_get_fw_length();
```

OTA worker 内部用 `esp_ota_write` 写扇区，全部写完后 `esp_ota_set_boot_partition` 切换并可在下次启动验证。

### 5. HTP — 初始化健康体温计 profile

`esp_ble_htp_init()` 注册服务 `0x1809` 与特征值（Temperature Measurement `0x2A1C`、Temperature Type `0x2A1D`、Intermediate Temperature `0x2A1E`、Measurement Interval `0x2A21`）。温度数据事件经 `BLE_HTP_EVENTS` 投递。

```c
#include "esp_htp.h"

static void app_htp_event_handler(void *arg, esp_event_base_t base,
                                  int32_t id, void *event_data)
{
    if (base != BLE_HTP_EVENTS) return;

    esp_ble_htp_data_t *d = (esp_ble_htp_data_t *)event_data;
    switch (id) {
    case BLE_HTP_CHR_UUID16_TEMPERATURE_MEASUREMENT:
    case BLE_HTP_CHR_UUID16_INTERMEDIATE_TEMPERATURE:
        if (d->flags.temperature_unit) {       // 1 = 华氏
            ESP_LOGI(TAG, "Temp %lu°F", (unsigned long)d->temperature.fahrenheit);
        } else {                                // 0 = 摄氏
            ESP_LOGI(TAG, "Temp %lu°C", (unsigned long)d->temperature.celsius);
        }
        if (d->flags.time_stamp) {
            ESP_LOGI(TAG, "  timestamp %04u-%02u-%02u %02u:%02u:%02u",
                     d->timestamp.year, d->timestamp.month, d->timestamp.day,
                     d->timestamp.hours, d->timestamp.minutes, d->timestamp.seconds);
        }
        if (d->flags.temperature_type) {
            ESP_LOGI(TAG, "  location %d", d->location);  // 1=Armpit..9=Tympanum
        }
        break;
    default: break;
    }
}

// app_main 里（conn init/start 之后）：
esp_ble_htp_init();
esp_event_handler_register(BLE_HTP_EVENTS, ESP_EVENT_ANY_ID,
                           app_htp_event_handler, NULL);
```

### 6. HTP — 读/写特征值

```c
uint8_t  temp_type = 0;
uint16_t interval  = 0;
esp_ble_htp_get_temp_type(&temp_type);              // 读 Temperature Type
esp_ble_htp_get_measurement_interval(&interval);    // 读 Measurement Interval
esp_ble_htp_set_measurement_interval(0);            // 0 = 停止周期测量
```

> 其它标准 profile（HRS 心率、BAS 电池、ANP 提醒、MIDI、OTP 对象传输）位于 `components/bluetooth/ble_services/*` 与 `components/bluetooth/ble_profiles/std/*`，用法与 HTP 类似（init + 事件 base 监听）。

## 常见错误

| 错误 | 原因 | 解决方法 |
|---|---|---|
| OTA 写 flash 失败 / 分区找不到 | 没有自定义分区表或缺少 `ota_0`/`ota_1` | 复制示例 `partitions.csv`；开 `CONFIG_PARTITION_TABLE_CUSTOM=y` |
| `esp_ble_ota_raw_init` 链接报错 | 未开 `CONFIG_BLE_OTA_RAW_PROFILE=y` | sdkconfig 启用该宏 |
| OTA 传输很慢 | MTU 默认 23 字节 | menuconfig 调 `CONFIG_BT_NIMBLE_ATT_PREFERRED_MTU=498` |
| 固件回调收不到数据 | 没调 `esp_ble_ota_raw_recv_fw_data_callback` | 注册固件接收回调，否则扇区被丢弃 |
| HTP 事件不触发 | 没监听 `BLE_HTP_EVENTS` base | HTP 用独立事件 base，不是 `BLE_CONN_MGR_EVENTS` |
| HTP 温度单位乱 | 没看 `flags.temperature_unit` | bit0=1 是华氏，=0 是摄氏；取对应联合体成员 |
| OTA 升级后不启动 | 没切 boot partition | 让 OTA worker 跑完（写完 + `esp_ota_set_boot_partition`） |
| `esp_ble_ota_raw_get_fw_length` 返回 `UINT32_MAX` | START 命令未到 | 主机未发 START，或 MTU 太小导致握手失败 |

## 参考项目

- OTA 头文件：`components/bluetooth/ble_profiles/esp/ble_ota_raw/include/esp_ble_ota_raw.h`
- OTA 示例辅助：`examples/bluetooth/ble_profiles/ble_ota/main/include/ble_ota_raw.h`
- HTP 头文件：`components/bluetooth/ble_profiles/std/ble_htp/include/esp_htp.h`
- OTA 完整示例：`examples/bluetooth/ble_profiles/ble_ota`（含 `partitions.csv`、`sdkconfig.defaults`）
- HTP 完整示例：`examples/bluetooth/ble_profiles/ble_htp`（含 console 命令 `app_htp.c`）
- HRP 心率示例：`examples/bluetooth/ble_profiles/ble_hrp`
- 在线文档：`docs/en/bluetooth/ble_ota.rst`、`docs/en/bluetooth/ble_htp.rst`、`docs/en/bluetooth/ble_profiles.rst`
