# 批量固件 OTA 升级

> **适用摘要**: initiator 从 HTTP 下载固件并写入本地升级分区，再扫描 responder、分发包并支持断点续传；responder 启动升级接收并写 flash（参考 `examples/ota`）。

## 触发意图

- "ESP-NOW OTA"
- "批量升级"
- "固件分发"
- "断点续传"
- "espnow_ota"

## 前置条件

| 条件 | 要求 |
|---|---|
| 头文件 | `espnow_ota.h`, `esp_ota_ops.h`, `esp_partition.h` |
| 依赖 | initiator 需联网（`protocol_examples_common` / `example_connect`） |
| 参考示例 | `examples/ota/main/app_main.c` |

## 分步说明

### 1. 公共初始化（initiator 需联网）

```c
#include "espnow.h"
#include "espnow_ota.h"

void app_main(void)
{
    espnow_storage_init();
#ifdef CONFIG_APP_ESPNOW_OTA_INITIATOR
    app_wifi_init();   // 内部 example_connect()，需联网
#else
    app_wifi_init();   // responder 仅 STA
#endif

    espnow_config_t cfg = ESPNOW_INIT_CONFIG_DEFAULT();
    espnow_init(&cfg);

#ifdef CONFIG_APP_ESPNOW_OTA_INITIATOR
    /* initiator 流程 */
#elif CONFIG_APP_ESPNOW_OTA_RESPONDER
    /* responder 流程 */
#endif
}
```

### 2. Initiator：从 HTTP 下载固件到本地升级分区

```c
#define OTA_DATA_PAYLOAD_LEN 1024

static size_t app_firmware_download(const char *url)
{
    esp_http_client_config_t config = { .url = url, .transport_type = HTTP_TRANSPORT_UNKNOWN };
    esp_http_client_handle_t client = esp_http_client_init(&config);

    // 重试连接
    esp_err_t ret;
    do {
        ret = esp_http_client_open(client, 0);
        if (ret != ESP_OK) vTaskDelay(pdMS_TO_TICKS(1000));
    } while (ret != ESP_OK);

    size_t total_size = esp_http_client_fetch_headers(client);
    const esp_partition_t *p = esp_ota_get_next_update_partition(NULL);
    esp_ota_handle_t ota_handle = 0;
    esp_ota_begin(p, total_size, &ota_handle);

    uint8_t *data = ESP_MALLOC(OTA_DATA_PAYLOAD_LEN);
    for (ssize_t size = 0, recv = 0; recv < total_size; recv += size) {
        size = esp_http_client_read(client, (char *)data, OTA_DATA_PAYLOAD_LEN);
        if (size > 0) esp_ota_write(ota_handle, data, OTA_DATA_PAYLOAD_LEN);
    }
    esp_ota_end(ota_handle);

    ESP_FREE(data);
    esp_http_client_close(client);
    esp_http_client_cleanup(client);
    return total_size;
}
```

### 3. Initiator：分发固件回调（从本地分区读出）

```c
esp_err_t app_ota_initiator_data_cb(size_t src_offset, void *dst, size_t size)
{
    static const esp_partition_t *p = NULL;
    if (!p) p = esp_ota_get_next_update_partition(NULL);
    return esp_partition_read(p, src_offset, dst, size);
}
```

### 4. Initiator：扫描 → 发送 → 释放结果

```c
static void app_firmware_send(size_t firmware_size, uint8_t sha[ESPNOW_OTA_HASH_LEN])
{
    espnow_ota_responder_t *info_list = NULL;
    size_t num = 0;
    espnow_ota_initiator_scan(&info_list, &num, pdMS_TO_TICKS(3000));
    ESP_LOGW(TAG, "wait ota num: %u", num);
    if (!num) { goto EXIT; }

    espnow_addr_t *dest = ESP_MALLOC(num * ESPNOW_ADDR_LEN);
    for (size_t i = 0; i < num; i++)
        memcpy(dest[i], info_list[i].mac, ESPNOW_ADDR_LEN);
    espnow_ota_initiator_scan_result_free();

    espnow_ota_result_t result = {0};
    esp_err_t ret = espnow_ota_initiator_send(dest, num, sha, firmware_size,
                                              app_ota_initiator_data_cb, &result);
    ESP_ERROR_GOTO(ret != ESP_OK, EXIT, "initiator_send");

    ESP_LOGI(TAG, "successed %u, unfinished %u",
             result.successed_num, result.unfinished_num);
EXIT:
    ESP_FREE(dest);
    espnow_ota_initiator_result_free(&result);
}

// 在 app_main 里:
uint8_t sha_256[32] = {0};
const esp_partition_t *p = esp_ota_get_next_update_partition(NULL);
size_t firmware_size = app_firmware_download(CONFIG_APP_ESPNOW_FIRMWARE_UPGRADE_URL);
esp_partition_get_sha256(p, sha_256);   // 用于断点续传比对（取前 16 字节 ESPNOW_OTA_HASH_LEN）
app_firmware_send(firmware_size, sha_256);
```

### 5. Responder：启动升级

```c
#ifdef CONFIG_APP_ESPNOW_OTA_RESPONDER
espnow_ota_config_t ota_config = {
    .skip_version_check       = true,   // 跳过版本检查
    .progress_report_interval = 10,     // 每进 10% 上报一次状态
};
espnow_ota_responder_start(&ota_config);
#endif
```

### 6. OTA 事件（可选，监听进度）

```c
// 事件 ID（来自 espnow_ota.h）
// ESP_EVENT_ESPNOW_OTA_STARTED        开始升级
// ESP_EVENT_ESPNOW_OTA_STATUS         主动上报进度
// ESP_EVENT_ESPNOW_OTA_FINISH         升级完成，重启后跑新固件
// ESP_EVENT_ESPNOW_OTA_STOPED         停止升级
// ESP_EVENT_ESPNOW_OTA_FIRMWARE_DOWNLOAD 开始写 flash
// ESP_EVENT_ESPNOW_OTA_SEND_FINISH    initiator 分发完成
esp_event_handler_register(ESP_EVENT_ESPNOW, ESP_EVENT_ANY_ID, ota_event_cb, NULL);
```

> 关键尺寸：`ESPNOW_OTA_HASH_LEN=16`，`ESPNOW_OTA_PACKET_MAX_SIZE = ((ESPNOW_DATA_LEN-4) - (ESPNOW_DATA_LEN-4)%16)`，`ESPNOW_OTA_PROGRESS_MAX_SIZE = ESPNOW_DATA_LEN - 30`。

## 常见错误

| 错误 | 原因 | 解决方法 |
|---|---|---|
| initiator 分发空 | 本地未先下载固件 | 先 `app_firmware_download` 写到 next update partition |
| 断点续传失效 | SHA-256 不一致 | 用 `esp_partition_get_sha256` 取分区真实 SHA |
| responder 不接收 | 未 `espnow_ota_responder_start` | responder 启动升级，必要时 `skip_version_check=true` |
| 内存泄漏 | 未 `*_result_free` | `espnow_ota_initiator_result_free(&result)` + `scan_result_free` + `ESP_FREE(dest)` |
| 升级超时 | `CONFIG_ESPNOW_OTA_WAIT_RESPONSE_TIMEOUT` 太小 | 默认 10000ms，可调；同时看 `OTA_RETRY_COUNT`(默认 50) |
| 重传不稳 | 信道差异 | 调 `CONFIG_ESPNOW_OTA_RETRANSMISSION_TIMES`(默认 2) |

## 参考

- `examples/ota/main/app_main.c` — initiator 下载+分发、responder 接收完整示例
- `src/ota/include/espnow_ota.h` — 全部结构与错误码
- Kconfig：`CONFIG_ESPNOW_OTA_RETRANSMISSION_TIMES` / `_RETRY_COUNT` / `_SEND_FORWARD_TTL` / `_WAIT_RESPONSE_TIMEOUT`
