# OTA 固件升级

> **适用摘要**: 通过 HTTPS 下载新固件并切换分区启动，覆盖高层 `esp_https_ota`（推荐）与底层 `esp_ota_begin/write/end` 两种用法（适配自 simple_ota_example���。

> Version: ESP-IDF version used by the project.
> Validation: draft metadata added from repository routing; verify APIs, Kconfig symbols, and component dependencies against the selected version.

## 触发意图

- "OTA 升级"
- "远程更新固件"
- "esp_https_ota"
- "双分区切换"
- "app_update"

## 前置条件

| 条件 | 要求 |
|---|---|
| 组件 | `app_update`、`esp_http_client`、`esp_https_ota`、`esp_tls` |
| 头文件 | `esp_ota_ops.h`、`esp_https_ota.h` |
| 分区表 | 含 `factory` + `ota_0` + `ota_1` + `otadata` |
| 参考 | `examples/system/ota/simple_ota_example`、`native_ota_example`、`advanced_https_ota` |

## 分步说明

### 1. 分区表（必备 OTA 分区）

```csv
# Name,   Type, SubType, Offset, Size
nvs,      data, nvs,     ,       0x4000
otadata,  data, ota,     ,       0x2000
factory,  app,  factory, ,       1M
ota_0,    app,  ota_0,   ,       1M
ota_1,    app,  ota_1,   ,       1M
```
menuconfig 选 `Factory app, two OTA`，或 `Custom CSV` 指向上面文件。

### 2. 高层 API（推荐）：`esp_https_ota`（适配自 simple_ota_example.c）

```c
#include "esp_https_ota.h"
#include "esp_http_client.h"
#include "esp_log.h"

static const char *TAG = "ota";

void ota_task(void *arg)
{
    esp_http_client_config_t http_cfg = {
        .url = "https://example.com/firmware.bin",
        .cert_pem = (const char *)server_cert_pem_start,   /* 内嵌 PEM */
        .timeout_ms = 10000,
        .keep_alive_enable = true,
    };
    esp_https_ota_config_t ota_cfg = {
        .http_config = &http_cfg,
    };

    esp_err_t ret = esp_https_ota(&ota_cfg);
    if (ret == ESP_OK) {
        ESP_LOGI(TAG, "OTA OK, rebooting...");
        esp_restart();
    } else {
        ESP_LOGE(TAG, "OTA failed: %s", esp_err_to_name(ret));
    }
    vTaskDelete(NULL);
}

void app_main(void)
{
    /* NVS + Wi-Fi 连接（见 wifi_sta recipe）... */
    xTaskCreate(ota_task, "ota", 8192, NULL, 5, NULL);
}
```

> PEM 证书常通过组件 `EMBED_TXTFILES` 内嵌，或在 menuconfig 设 `skip_cert_common_name_check`（仅测试用）。

### 3. 底层 API：`esp_ota_*` 分步写（自定义来源时）

```c
#include "esp_ota_ops.h"

const esp_partition_t *next = esp_ota_get_next_update_partition(NULL);
esp_ota_handle_t handle;
ESP_ERROR_CHECK(esp_ota_begin(next, OTA_SIZE_UNKNOWN, &handle));

/* 循环从你的来源（串口/蓝牙/HTTP）拿数据 chunk */
while (have_data) {
    ESP_ERROR_CHECK(esp_ota_write(handle, chunk, chunk_len));
}
ESP_ERROR_CHECK(esp_ota_end(handle));
ESP_ERROR_CHECK(esp_ota_set_boot_partition(next));   /* 设置下次启动分区 */
esp_restart();
```

### 关键 API

```c
const esp_partition_t* esp_ota_get_next_update_partition(const esp_partition_t *start_from);
const esp_partition_t* esp_ota_get_running_partition(void);
const esp_partition_t* esp_ota_get_boot_partition(void);
esp_err_t esp_ota_begin(const esp_partition_t *partition, size_t image_size, esp_ota_handle_t *out_handle);
esp_err_t esp_ota_write(esp_ota_handle_t handle, const void *data, size_t size);
esp_err_t esp_ota_end(esp_ota_handle_t handle);
esp_err_t esp_ota_abort(esp_ota_handle_t handle);
esp_err_t esp_ota_set_boot_partition(const esp_partition_t *partition);
esp_err_t esp_https_ota(const esp_https_ota_config_t *ota_config);
```

`OTA_SIZE_UNKNOWN`（0xffffffff）用于流式未知大小镜像（适配自 `esp_ota_ops.h`）。

### 关键 Kconfig

- `CONFIG_PARTITION_TABLE_TYPE` = `Factory app, two OTA`
- `CONFIG_ESP_HTTPS_OTA_DECRYPT_CB`：启用解密回调（加密镜像，可选）
- HTTP client 相关：`CONFIG_ESP_HTTP_CLIENT_ENABLE_HTTPS`

## 常见错误

| 错误 | 原因 | 解决方法 |
|---|---|---|
| `no next partition` | 分区表无 ota_0/ota_1 | 加 OTA 分区并选对 Partition Table Type |
| HTTPS 握手失败 | 证书不匹配 | 内嵌正确 PEM；测试可 `skip_cert_common_name_check` |
| `image invalid` | 镜像目标不符/损坏 | 镜像须为同目标编译；勿传错文件 |
| 升级后不切换 | 未 `set_boot_partition` | `esp_ota_end` 后调 `esp_ota_set_boot_partition` 再 `esp_restart` |
| 升级中重启回旧版 | otadata 未更新或校验失败 | 等到 `esp_ota_set_boot_partition` 返回 ESP_OK 再重启 |
| 网络中断 | HTTP 超时 | 增大 `timeout_ms`、断点续传或改用 advanced_https_ota |

## 参考

- `examples/system/ota/simple_ota_example` — HTTPS OTA（高层 API）
- `examples/system/ota/native_ota_example` — 底层 `esp_ota_*` 完整流程
- `examples/system/ota/advanced_https_ota` — 带进度/断点续传
- ESP-IDF `components/app_update/include/esp_ota_ops.h`、`components/esp_https_ota/include/esp_https_ota.h`
