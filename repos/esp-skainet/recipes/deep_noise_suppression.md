# 深度降噪（AFE_TYPE_VC + NSNET2）

> **适用摘要**: 用语音通信型 AFE（`AFE_TYPE_VC`）+ 深度降噪模型做实时降噪，并把原始/降噪后 PCM 通过 ringbuf + SD 卡落盘用于评估对比。

## 触发意图

- "deep noise suppression"
- "深度降噪"
- "nsnet2"
- "保存降噪前后 PCM"

## 前置条件

| 条件 | 要求 |
|---|---|
| 参考工程 | `examples/deep_noise_suppression/` |
| 依赖组件 | `espressif/esp-sr` (^2.0.0) |
| Kconfig | `SR_NSN_NSNET2`（Select noise suppression model → Deep noise suppression v2） |
| 硬件 | SD 卡（用于 `DEBUG_SAVE_PCM` 落盘） |
| 目标芯片 | ESP32-S3 / P4 / S31（nsnet2 不支持 ESP32） |

## 分步说明

### 1. 引入头文件

```c
#include "esp_afe_sr_models.h"
#include "esp_nsn_models.h"
#include "esp_board_init.h"
#include "model_path.h"
#include "ringbuf.h"
```

### 2. app_main：选 VC 类型 + SD 卡

```c
static const esp_afe_sr_iface_t *afe_handle = NULL;
static volatile int task_flag = 0;

#define DEBUG_SAVE_PCM 1
#if DEBUG_SAVE_PCM
ringbuf_handle_t rb_debug[2] = {NULL};
FILE *file_save[2] = {NULL};
#endif

void app_main()
{
    ESP_ERROR_CHECK(esp_board_init(16000, 1, 16));
#if DEBUG_SAVE_PCM
    ESP_ERROR_CHECK(esp_sdcard_init("/sdcard", 10));
#endif

    srmodel_list_t *models = esp_srmodel_init("model");
    // 关键：用 AFE_TYPE_VC 触发非线性降噪（含 NSNET2）
    afe_config_t *afe_config = afe_config_init(esp_get_input_format(), models,
                                               AFE_TYPE_VC, AFE_MODE_LOW_COST);
    afe_handle = esp_afe_handle_from_config(afe_config);
    esp_afe_sr_data_t *afe_data = afe_handle->create_from_config(afe_config);
    afe_config_free(afe_config);

#if DEBUG_SAVE_PCM
    // 原始 feed 多通道 4 秒 ringbuf
    rb_debug[0] = rb_create(afe_handle->get_feed_channel_num(afe_data) * 4 * 16000 * 2, 1);
    file_save[0] = fopen("/sdcard/feed.pcm", "w");
    // 降噪后单通道 4 秒 ringbuf
    rb_debug[1] = rb_create(1 * 4 * 16000 * 2, 1);
    file_save[1] = fopen("/sdcard/fetch.pcm", "w");
    xTaskCreatePinnedToCore(&debug_pcm_save_Task, "debug_pcm_save", 2 * 1024, NULL, 5, NULL, 1);
#endif

    task_flag = 1;
    xTaskCreatePinnedToCore(&feed_Task,   "feed",   8 * 1024, (void *)afe_data, 5, NULL, 0);
    xTaskCreatePinnedToCore(&detect_Task, "detect", 8 * 1024, (void *)afe_data, 5, NULL, 0);
}
```

### 3. feed 任务：取音 + 落原始 PCM

```c
void feed_Task(void *arg)
{
    esp_afe_sr_data_t *afe_data = (esp_afe_sr_data_t *)arg;
    int audio_chunksize = afe_handle->get_feed_chunksize(afe_data);
    int nch = afe_handle->get_feed_channel_num(afe_data);
    int feed_channel = esp_get_feed_channel();
    assert(nch == feed_channel);
    int16_t *i2s_buff = malloc(audio_chunksize * sizeof(int16_t) * feed_channel);

    while (task_flag) {
        esp_get_feed_data(true, i2s_buff, audio_chunksize * sizeof(int16_t) * feed_channel);
        afe_handle->feed(afe_data, i2s_buff);

#if DEBUG_SAVE_PCM
        if (rb_bytes_available(rb_debug[0]) < audio_chunksize * nch * sizeof(int16_t))
            printf("ERROR! rb_debug[0] slow!!!\n");
        rb_write(rb_debug[0], (char *)i2s_buff, audio_chunksize * nch * sizeof(int16_t), 0);
#endif
    }
    free(i2s_buff);
    vTaskDelete(NULL);
}
```

### 4. detect 任务：取降噪后数据 + 落盘

```c
void detect_Task(void *arg)
{
    esp_afe_sr_data_t *afe_data = (esp_afe_sr_data_t *)arg;
    int afe_chunksize = afe_handle->get_fetch_chunksize(afe_data);
    int16_t *buff = malloc(afe_chunksize * sizeof(int16_t));

    while (task_flag) {
        afe_fetch_result_t *res = afe_handle->fetch(afe_data);
        if (res && res->ret_value != ESP_FAIL) {
            memcpy(buff, res->data, afe_chunksize * sizeof(int16_t));
#if DEBUG_SAVE_PCM
            if (rb_bytes_available(rb_debug[1]) < afe_chunksize * 1 * sizeof(int16_t))
                printf("ERROR! rb_debug[1] slow!!!\n");
            rb_write(rb_debug[1], (char *)buff, afe_chunksize * 1 * sizeof(int16_t), 0);
#endif
        }
    }
    free(buff);
    vTaskDelete(NULL);
}
```

### 5. 落盘任务（PSRAM 缓冲）

```c
void debug_pcm_save_Task(void *arg)
{
    int size = 4 * 2 * 32 * 16;   // 32ms × 4ch × 2byte ≈ 4KB
    int16_t *buf_temp = heap_caps_calloc(1, size, MALLOC_CAP_SPIRAM | MALLOC_CAP_8BIT);

    while (task_flag) {
        for (int i = 0; i < 2; i++) {
            if (file_save[i] && rb_bytes_filled(rb_debug[i]) > size) {
                int ret = rb_read(rb_debug[i], (char *)buf_temp, size, 3000 / portTICK_PERIOD_MS);
                if (ret > 0 && ret >= size)
                    FatfsComboWrite(buf_temp, size, 1, file_save[i]);
            }
        }
        vTaskDelay(1 / portTICK_PERIOD_MS);
    }
    free(buf_temp);
    vTaskDelete(NULL);
}
```

## 常见错误

| 错误 | 原因 | 解决方法 |
|---|---|---|
| 编译报 nsnet2 未启用 | Kconfig 没选深度降噪 | menuconfig 选 `SR_NSN_NSNET2` |
| 仍是线性降噪 | 用了 `AFE_TYPE_SR` | 改为 `AFE_TYPE_VC` 才含非线性 NS |
| feed.pcm 杂音 | 通道数算错 | `nch = get_feed_channel_num()`，落盘按 nch |
| SD 卡写不动 | ringbuf 满（CPU 跟不上） | 提高落盘任务优先级，或减小帧数 |
| ESP32 上不可用 | nsnet2 不支持 ESP32 | 换 S3/P4 板 |

## 参考

- `examples/deep_noise_suppression/main/main.c` — 本 recipe 真实来源
- `espressif-repos/esp-sr/Kconfig.projbuild` — `SR_NSN_NSNET2` 选择项
- `espressif-repos/esp-sr/include/esp32s3/esp_afe_config.h` — `AFE_TYPE_VC`、`afe_ns_mode_t`
