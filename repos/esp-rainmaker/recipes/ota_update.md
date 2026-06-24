# RainMaker OTA 固件升级

> **适用摘要**: 启用 RainMaker OTA（推荐 `esp_rmaker_ota_enable_default()`），监听 OTA 事件，配置 OTA 协议（HTTPS/MQTT）与回滚诊断，以及手动 fetch。

## 触发意图

- "RainMaker OTA"
- "esp_rmaker_ota_enable_default"
- "固件升级 OTA"
- "OTA 回滚 rollback"
- "OTA using topics / using params"

## 前置条件

| 条件 | 要求 |
|---|---|
| 分区表 | `partitions.csv` 含 ota 分区 |
| Kconfig | OTA 相关配置见 `Kconfig.projbuild` "ESP RainMaker OTA Config" |
| 参考示例 | 所有示例均含 `esp_rmaker_ota_enable_default()` |

## 分步说明

### 1. 启用默认 OTA（推荐）

```c
#include <esp_rmaker_ota.h>

/* 在 esp_rmaker_start() 之前调用 */
esp_rmaker_ota_enable_default();
```

默认使用 “Using the Topics”，可从 Dashboard 触发；Self Claim 节点的 Primary 用户自动成为 admin。

### 2. OTA 工作流类型

```c
typedef enum {
    OTA_USING_PARAMS = 1,  /* 通过 service/parameter 触发 */
    OTA_USING_TOPICS        /* 通过预定义 MQTT topic 触发（推荐） */
} esp_rmaker_ota_type_t;
```

### 3. 自定义 OTA 配置

```c
esp_rmaker_ota_config_t ota_cfg = {
    .ota_cb      = NULL,                            /* NULL 用内部默认回调（推荐） */
    .ota_diag    = my_post_ota_diag,                /* 可选：app rollback 诊断 */
    .server_cert = ESP_RMAKER_OTA_DEFAULT_SERVER_CERT,
    .priv        = NULL,
};
esp_rmaker_ota_enable(&ota_cfg, OTA_USING_TOPICS);
```

可选内置回调：`esp_rmaker_ota_default_cb`（默认）/ `esp_rmaker_ota_https_cb`（强制 HTTPS）/ `esp_rmaker_ota_mqtt_cb`（强制 MQTT）。

### 4. OTA 事件监听（`RMAKER_OTA_EVENT` base）

```c
ESP_ERROR_CHECK(esp_event_handler_register(RMAKER_OTA_EVENT, ESP_EVENT_ANY_ID, &event_handler, NULL));

/* 在 event_handler 中： */
if (event_base == RMAKER_OTA_EVENT) {
    switch (event_id) {
        case RMAKER_OTA_EVENT_STARTING:       ESP_LOGI(TAG, "Starting OTA."); break;
        case RMAKER_OTA_EVENT_IN_PROGRESS:    ESP_LOGI(TAG, "OTA is in progress."); break;
        case RMAKER_OTA_EVENT_SUCCESSFUL:     ESP_LOGI(TAG, "OTA successful."); break;
        case RMAKER_OTA_EVENT_FAILED:         ESP_LOGI(TAG, "OTA Failed."); break;
        case RMAKER_OTA_EVENT_REJECTED:       ESP_LOGI(TAG, "OTA Rejected."); break;
        case RMAKER_OTA_EVENT_DELAYED:        ESP_LOGI(TAG, "OTA Delayed."); break;
        case RMAKER_OTA_EVENT_REQ_FOR_REBOOT: ESP_LOGI(TAG, "Image downloaded. Reboot to apply."); break;
    }
}
```

### 5. 自定义 OTA 回调（必须 report 状态）

```c
static esp_err_t my_ota_cb(esp_rmaker_ota_handle_t handle, esp_rmaker_ota_data_t *ota_data)
{
    ESP_LOGI(TAG, "OTA url=%s size=%d", ota_data->url, ota_data->filesize);

    esp_rmaker_ota_report_status(handle, OTA_STATUS_IN_PROGRESS, "downloading");
    /* ... 执行下载逻辑 ... */
    if (download_failed) {
        esp_rmaker_ota_report_status(handle, OTA_STATUS_FAILED, "download error");
        return ESP_FAIL;
    }
    esp_rmaker_ota_report_status(handle, OTA_STATUS_SUCCESS, NULL);
    return ESP_OK;
}
```

OTA 状态枚举（`ota_status_t`）：`OTA_STATUS_IN_PROGRESS`（可多次）/ `OTA_STATUS_SUCCESS` / `OTA_STATUS_FAILED` / `OTA_STATUS_DELAYED` / `OTA_STATUS_REJECTED`。

### 6. 回滚诊断（需 `CONFIG_BOOTLOADER_APP_ROLLBACK_ENABLE`）

```c
static esp_rmaker_ota_diag_status_t my_post_ota_diag(esp_rmaker_ota_diag_priv_t *priv, void *user)
{
    if (priv->state == OTA_DIAG_STATE_INIT) {
        /* 首次启动诊断 */
        return run_self_test() ? OTA_DIAG_STATUS_SUCCESS : OTA_DIAG_STATUS_FAIL;
    }
    /* OTA_DIAG_STATE_POST_MQTT */
    return mqtt_check_ok() ? OTA_DIAG_STATUS_SUCCESS : OTA_DIAG_STATUS_PENDING;
}
```

延后判定时返回 `OTA_DIAG_STATUS_PENDING`，事后用：
```c
esp_rmaker_ota_mark_valid();    /* 标记有效 */
esp_rmaker_ota_mark_invalid();  /* 标记无效并回滚 */
```

### 7. 手动触发 OTA 拉取（Using Topics）

```c
esp_rmaker_ota_fetch();              /* 立即查询是否有 OTA */
esp_rmaker_ota_fetch_with_delay(60); /* 60 秒后查询 */
```

### 8. 关键 Kconfig（OTA）

| 符号 | 默认 | 说明 |
|---|---|---|
| `CONFIG_ESP_RMAKER_OTA_AUTOFETCH` | y | 连接后主动拉取 |
| `CONFIG_ESP_RMAKER_OTA_USE_HTTPS` | 默认 | 协议 HTTPS（或 `_USE_MQTT`） |
| `CONFIG_ESP_RMAKER_OTA_DISABLE_AUTO_REBOOT` | n | OTA 后是否自动重启 |
| `CONFIG_ESP_RMAKER_SKIP_VERSION_CHECK` | n | 跳过版本检查（仅开发） |
| `CONFIG_ESP_RMAKER_OTA_MAX_RETRIES` | 3 | 失败重试次数（1~10） |
| `CONFIG_ESP_RMAKER_OTA_ROLLBACK_WAIT_PERIOD` | 90 | 回滚等待秒数（30~600） |
| `CONFIG_ESP_RMAKER_MQTT_OTA_BLOCK_SIZE` | 4096 | MQTT OTA 块大小（仅 MQTT） |
| `CONFIG_ESP_RMAKER_HTTP_OTA_RESUMPTION` | y | HTTP OTA 断点续传 |

## 常见错误

| 错误 | 原因 | 解决方法 |
|---|---|---|
| OTA 卡在 starting | 分区表无 ota 分区 | `partitions.csv` 加 `ota_0`/`ota_1` |
| 自定义回调返回 ESP_FAIL 但状态卡住 | 未先 `esp_rmaker_ota_report_status` | 失败前先 report FAILED |
| 版本号不更新被拒绝 | 版本检查未跳过 | 生产保留检查；开发设 `SKIP_VERSION_CHECK=y` |
| 回滚未触发 | 未启用 bootloader rollback | menuconfig 启用 `BOOTLOADER_APP_ROLLBACK_ENABLE` |
| HTTPS 下载失败 | 证书/CN 校验 | 用 `ESP_RMAKER_OTA_DEFAULT_SERVER_CERT`；必要时 `SKIP_COMMON_NAME_CHECK` |
| MQTT OTA 块超时 | 网络/块大小不当 | 调 `MQTT_OTA_BLOCK_WAIT_SEC`、`_BLOCK_SIZE` |

## 参考

- `components/esp_rainmaker/include/esp_rmaker_ota.h`
- `components/esp_rainmaker/Kconfig.projbuild` — “ESP RainMaker OTA Config”
- `examples/switch/main/app_main.c` — OTA 事件处理参考
- `resources/config_reference.md`
