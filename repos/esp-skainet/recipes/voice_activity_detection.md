# 语音活动检测（VAD）

> **适用摘要**: 用 AFE 内置 VAD 判断当前帧是语音还是噪声/静音，利用 `vad_cache` 避免首字被截断，并可把语音帧落盘到 SD 卡。

## 触发意图

- "voice activity detection"
- "VAD 语音检测"
- "vad_state / vad_cache"
- "检测是否在说话"

## 前置条件

| 条件 | 要求 |
|---|---|
| 参考工程 | `examples/voice_activity_detection/` |
| 依赖组件 | `espressif/esp-sr` (^2.0.0) |
| Kconfig | `SR_VADN_WEBRTC`（默认）或 `SR_VADN_VADNET1_MEDIUM`（神经网络 VAD） |
| 硬件 | SD 卡（落盘语音片段时） |

## 分步说明

### 1. app_main：调节 VAD 参数

```c
static const esp_afe_sr_iface_t *afe_handle = NULL;

void app_main()
{
    ESP_ERROR_CHECK(esp_board_init(16000, 1, 16));
    bool sdcard_enable = true;
    if (sdcard_enable)
        ESP_ERROR_CHECK(esp_sdcard_init("/sdcard", 10));

    srmodel_list_t *models = esp_srmodel_init("model");
    afe_config_t *afe_config = afe_config_init(esp_get_input_format(), models,
                                               AFE_TYPE_SR, AFE_MODE_LOW_COST);

    // VAD 调节（来自真实示例）
    afe_config->vad_min_noise_ms   = 1000;  // 噪声/静音最短时长 ms
    afe_config->vad_min_speech_ms  = 128;   // 语音最短时长 ms
    afe_config->vad_mode           = VAD_MODE_1;   // 越大越易触发

    afe_handle = esp_afe_handle_from_config(afe_config);
    esp_afe_sr_data_t *afe_data = afe_handle->create_from_config(afe_config);
    afe_config_free(afe_config);

    xTaskCreatePinnedToCore(&feed_Task,   "feed",   8 * 1024, (void *)afe_data, 5, NULL, 0);
    xTaskCreatePinnedToCore(&detect_Task, "detect", 4 * 1024, (void *)afe_data, 5, NULL, 1);
}
```

### 2. detect 任务：读 vad_state 与 vad_cache

```c
void detect_Task(void *arg)
{
    esp_afe_sr_data_t *afe_data = (esp_afe_sr_data_t *)arg;
    printf("------------vad start------------\n");

    FILE *fp = NULL;
    fp = fopen("/sdcard/TEST.pcm", "w+");
    if (fp == NULL) printf("can not open file!\n");

    while (1) {
        afe_fetch_result_t *res = afe_handle->fetch(afe_data);
        if (!res || res->ret_value == ESP_FAIL) {
            printf("fetch error!\n");
            break;
        }

        // 当前帧状态
        printf("vad state: %s\n", res->vad_state == VAD_SILENCE ? "noise" : "speech");

        // VAD cache：处理首字截断
        // 说明（来自示例注释）：
        //  1. VAD 算法本身有 1~3 帧固有延迟；
        //  2. 为防误触发，连续触发达到 vad_min_speech_ms 才判为语音。
        // 因此首帧语音可能被截断，AFE V2.0 用 vad_cache 缓存这段，按 vad_cache_size 判断是否需要保存。
        if (fp) {
            if (res->vad_cache_size > 0) {
                printf("Save vad cache: %d\n", res->vad_cache_size);
                FatfsComboWrite(res->vad_cache, 1, res->vad_cache_size, fp);
            }
            if (res->vad_state == VAD_SPEECH) {
                FatfsComboWrite(res->data, 1, res->data_size, fp);
            }
        }
    }
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
    while (1) {
        esp_get_feed_data(true, i2s_buff, audio_chunksize * sizeof(int16_t) * feed_channel);
        afe_handle->feed(afe_data, i2s_buff);
    }
    free(i2s_buff);
    vTaskDelete(NULL);
}
```

### 4. vad_mode 取值（来自 afe_config.h）

```c
// vad_mode_t: VAD_MODE_0 .. VAD_MODE_4
// mode 越大，语音触发概率越高（也越易误触发）
```

> 若想用神经网络 VAD，menuconfig 选 `SR_VADN_VADNET1_MEDIUM`，`afe_config->vad_model_name` 会被设为对应模型名（默认 NULL = WebRTC）。

## 常见错误

| 错误 | 原因 | 解决方法 |
|---|---|---|
| 首字被切掉 | 没用 vad_cache 补帧 | 检查 `vad_cache_size > 0` 时落盘 cache |
| 一直判定为 noise | vad_mode 太低 / min_speech_ms 太大 | 调大 `vad_mode`，减小 `vad_min_speech_ms`（>32） |
| 频繁误触发 | vad_mode 太高 | 调低 mode，增大 `vad_min_speech_ms` |
| vad_cache 仍不够 | 算法延迟 + 阈值双重截断 | 增大 `vad_delay_ms`（默认 128） |

## 参考

- `examples/voice_activity_detection/main/main.c` — 本 recipe 真实来源（含 vad_cache 注释）
- `espressif-repos/esp-sr/include/esp32s3/esp_afe_config.h` — `vad_mode`、`vad_min_speech_ms`、`vad_min_noise_ms`、`vad_delay_ms`
- `espressif-repos/esp-sr/Kconfig.projbuild` — `SR_VADN_WEBRTC` / `SR_VADN_VADNET1_MEDIUM`
