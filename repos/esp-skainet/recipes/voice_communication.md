# 语音通信（Voice Communication）数据增强

> **适用摘要**: 用 `AFE_TYPE_VC` 对通信语音做实时增强（降噪/AGC），从 `fetch` 取增强后的单声道 PCM，用于上行通话或录制。结构与深度降噪几乎相同，重点在"取数据、不强求落盘"。

> Version: selected repo/component version; align with the user project and dependency manifest.
> Evidence: `repos/esp-skainet/resources/`, source/examples in `repos/esp-skainet/`, and this recipe path `repos/esp-skainet/recipes/voice_communication.md`.
> Validation: draft metadata added from repository routing; verify APIs, Kconfig symbols, and component dependencies against the selected version.

## 触发意图

- "voice communication"
- "语音通信增强"
- "AFE_TYPE_VC 取数据"
- "通话降噪"

## 前置条件

| 条件 | 要求 |
|---|---|
| 参考工程 | `examples/voice_communication/` |
| 依赖组件 | `espressif/esp-sr` (^2.0.0) |
| AFE type | `AFE_TYPE_VC`（16kHz 输入）；8kHz 通话用 `AFE_TYPE_VC_8K` |

## 分步说明

### 1. app_main

```c
static const esp_afe_sr_iface_t *afe_handle = NULL;
static volatile int task_flag = 0;

void app_main()
{
    ESP_ERROR_CHECK(esp_board_init(16000, 1, 16));

    srmodel_list_t *models = esp_srmodel_init("model");
    afe_config_t *afe_config = afe_config_init(esp_get_input_format(), models,
                                               AFE_TYPE_VC, AFE_MODE_LOW_COST);
    afe_handle = esp_afe_handle_from_config(afe_config);
    esp_afe_sr_data_t *afe_data = afe_handle->create_from_config(afe_config);
    afe_config_free(afe_config);

    task_flag = 1;
    xTaskCreatePinnedToCore(&feed_Task,   "feed",   8 * 1024, (void *)afe_data, 5, NULL, 0);
    xTaskCreatePinnedToCore(&detect_Task, "detect", 8 * 1024, (void *)afe_data, 5, NULL, 0);
}
```

### 2. detect 任务：取增强后单声道数据

```c
void detect_Task(void *arg)
{
    esp_afe_sr_data_t *afe_data = (esp_afe_sr_data_t *)arg;
    int afe_chunksize = afe_handle->get_fetch_chunksize(afe_data);
    int16_t *buff = malloc(afe_chunksize * sizeof(int16_t));
    printf("------------detect start------------\n");

    while (task_flag) {
        afe_fetch_result_t *res = afe_handle->fetch(afe_data);
        if (res && res->ret_value != ESP_FAIL) {
            // VC 模式下 res->data 为增强后的单声道 PCM
            memcpy(buff, res->data, afe_chunksize * sizeof(int16_t));

            // 这��把 buff 送往上行的网络/编码/播放
            // send_to_uplink(buff, afe_chunksize);
        }
    }
    free(buff);
    vTaskDelete(NULL);
}
```

### 3. feed 任务

```c
void feed_Task(void *arg)
{
    esp_afe_sr_data_t *afe_data = (esp_afe_sr_data_t *)arg;
    int audio_chunksize = afe_handle->get_feed_chunksize(afe_data);
    int feed_channel = esp_get_feed_channel();
    int16_t *i2s_buff = malloc(audio_chunksize * sizeof(int16_t) * feed_channel);
    while (task_flag) {
        esp_get_feed_data(true, i2s_buff, audio_chunksize * sizeof(int16_t) * feed_channel);
        afe_handle->feed(afe_data, i2s_buff);
    }
    free(i2s_buff);
    vTaskDelete(NULL);
}
```

### 4. VC vs VC_8K 选择

- `AFE_TYPE_VC`：16 kHz 输入，主流通信场景
- `AFE_TYPE_VC_8K`：8 kHz 输入，**喂入数据必须是 8 kHz**，否则结果异常

## 常见错误

| 错误 | 原因 | 解决方法 |
|---|---|---|
| 数据有底噪 | 没启用 NS/AGC | 确认 `ns_init=true`、`agc_init=true`（VC 默认开） |
| 8K 模式音调异常 | 喂了 16k 数据给 VC_8K | 喂入前先重采样到 8k，或改回 VC |
| fetch 间隔不稳 | feed/detect 同核抢占 | feed 与 detect 分核（示例同核因负载轻，重负载需分开） |
| 增强后音量小 | linear_gain 默认 1.0 | 调 `afe_config->afe_linear_gain = 2.0` |

## 参考

- `examples/voice_communication/main/main.c` — 本 recipe 真实来源
- `examples/deep_noise_suppression/main/main.c` — 含 PCM 落盘的更完整版本
- `espressif-repos/esp-sr/include/esp32s3/esp_afe_config.h` — `AFE_TYPE_VC` / `AFE_TYPE_VC_8K` / `AFE_TYPE_FD`
