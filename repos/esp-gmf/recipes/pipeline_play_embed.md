# 基于流水线播放 Flash 内嵌音乐

> **适用摘要**: 用 GMF-Core 的 pool/pipeline 构建「embed_flash → aud_dec → 音频效果链 → codec_dev」最小播放流水线，覆盖 pool 注册、pipeline 构建、task 绑定与事件等待。是理解 ESP-GMF 工作流的入门模板。

## 触发意图

- "播放 flash 内嵌的音频"
- "用 GMF 播放提示音"
- "怎么构建 pipeline"
- "gmf 最小播放示例"
- "pool 怎么用"

## 前置条件

| 条件 | 要求 |
|---|---|
| ESP-IDF 版本 | `>= v5.4.3` (release/v5.4)、`>= v5.5.2` (release/v5.5) 或 `>= v6.0` |
| 开发板 | 任一 `esp_board_manager` 支持的音频板（如 `esp32_s3_korvo_2_3`），含 DAC 与扬声器 |
| 参考示例 | `gmf_examples/basic_examples/pipeline_play_embed_music` |

## 分步说明

### 1. 初始化板级播放外设

经 `esp_board_manager` 获取 codec_dev 句柄并打开。

```c
#include "esp_board_manager_includes.h"
#include "esp_codec_dev.h"

static int playback_peripheral_init(esp_codec_dev_handle_t *out_handle)
{
    int ret = esp_board_manager_init_device_by_name(ESP_BOARD_DEVICE_NAME_AUDIO_DAC);
    if (ret != ESP_OK) return ret;

    dev_audio_codec_handles_t *h = NULL;
    esp_board_manager_get_device_handle(ESP_BOARD_DEVICE_NAME_AUDIO_DAC, (void **)&h);
    if (h == NULL || h->codec_dev == NULL) return ESP_ERR_NOT_FOUND;

    *out_handle = h->codec_dev;
    esp_codec_dev_set_out_vol(*out_handle, 80);
    esp_codec_dev_sample_info_t fs = {
        .sample_rate      = CONFIG_GMF_AUDIO_EFFECT_RATE_CVT_DEST_RATE,
        .channel          = CONFIG_GMF_AUDIO_EFFECT_CH_CVT_DEST_CH,
        .bits_per_sample  = CONFIG_GMF_AUDIO_EFFECT_BIT_CVT_DEST_BITS,
    };
    return esp_codec_dev_open(*out_handle, &fs);
}
```

### 2. 建 pool 并用 gmf_loader 批量注册

```c
#include "esp_gmf_pool.h"
#include "gmf_loader_setup_defaults.h"
#include "esp_gmf_app_setup_peripheral.h"

esp_gmf_pool_handle_t pool = NULL;
esp_gmf_pool_init(&pool);
gmf_loader_setup_io_default(pool);            // 注册 io_file/io_http/io_embed_flash/io_codec_dev ...
gmf_loader_setup_audio_codec_default(pool);   // 注册 aud_dec / aud_enc
gmf_loader_setup_audio_effects_default(pool); // 注册 aud_rate_cvt/aud_ch_cvt/aud_bit_cvt/aud_eq ...
ESP_GMF_POOL_SHOW_ITEMS(pool);                // 打印所有已注册 element/IO 的 tag
```

### 3. 按名建 pipeline，绑定尾 IO 设备

element 名数组顺序即数据流方向；头/尾分别给 in/out 的 IO tag。

```c
#include "esp_gmf_pipeline.h"
#include "esp_gmf_io_codec_dev.h"
#include "esp_gmf_io_embed_flash.h"
#include "esp_embed_tone.h"

const char *name[] = {"aud_dec", "aud_bit_cvt", "aud_rate_cvt", "aud_ch_cvt"};
esp_gmf_pipeline_handle_t pipe = NULL;
esp_gmf_pool_new_pipeline(pool, "io_embed_flash", name, 4, "io_codec_dev", &pipe);

// 把 codec_dev 句柄喂给尾 IO
esp_gmf_io_codec_dev_set_dev(ESP_GMF_PIPELINE_GET_OUT_INSTANCE(pipe), playback_handle);

// 设置 embed URL 与资源表（URL 格式 embed://<group>/<index>_<name>.<ext>）
esp_gmf_pipeline_set_in_uri(pipe, esp_embed_tone_url[ESP_EMBED_TONE_FF_16B_1C_44100HZ_MP3]);
esp_gmf_io_handle_t in_io = NULL;
esp_gmf_pipeline_get_in(pipe, &in_io);
esp_gmf_io_embed_flash_set_context(in_io,
    (embed_item_info_t *)&g_esp_embed_tone[0], ESP_EMBED_TONE_URL_MAX);
```

### 4. 用 helper 推断 FourCC 并 reconfig 解码器

```c
#include "esp_gmf_audio_helper.h"
#include "esp_gmf_audio_dec.h"

esp_gmf_element_handle_t dec_el = NULL;
esp_gmf_pipeline_get_el_by_name(pipe, "aud_dec", &dec_el);

esp_gmf_info_sound_t info = {0};
esp_gmf_audio_helper_get_audio_type_by_uri(
    esp_embed_tone_url[ESP_EMBED_TONE_FF_16B_1C_44100HZ_MP3], &info.format_id);
esp_gmf_audio_dec_reconfig_by_sound_info(dec_el, &info);
```

### 5. task + bind + loading_jobs + event（顺序固定）

```c
#include "esp_gmf_task.h"
#include "freertos/event_groups.h"

#define PIPELINE_BLOCK_BIT  BIT(0)

esp_gmf_task_cfg_t cfg = DEFAULT_ESP_GMF_TASK_CONFIG();
cfg.name = "pipe_embed";
esp_gmf_task_handle_t task = NULL;
esp_gmf_task_init(&cfg, &task);
esp_gmf_pipeline_bind_task(pipe, task);
esp_gmf_pipeline_loading_jobs(pipe);    // 注册 open/process job，不可省

EventGroupHandle_t evt = xEventGroupCreate();
esp_gmf_pipeline_set_event(pipe, _pipeline_event, evt);
```

### 6. run 并等待终态，然后 stop + 销毁

```c
esp_gmf_pipeline_run(pipe);
xEventGroupWaitBits(evt, PIPELINE_BLOCK_BIT, pdTRUE, pdFALSE, portMAX_DELAY);
esp_gmf_pipeline_stop(pipe);

esp_gmf_task_deinit(task);
esp_gmf_pipeline_destroy(pipe);
gmf_loader_teardown_audio_effects_default(pool);
gmf_loader_teardown_audio_codec_default(pool);
gmf_loader_teardown_io_default(pool);
esp_gmf_pool_deinit(pool);
```

事件回调（识别三个终态）：

```c
esp_gmf_err_t _pipeline_event(esp_gmf_event_pkt_t *event, void *ctx)
{
    ESP_LOGI(TAG, "EVT el:%s sub:%s",
             OBJ_GET_TAG(event->from), esp_gmf_event_get_state_str(event->sub));
    if (event->sub == ESP_GMF_EVENT_STATE_STOPPED
        || event->sub == ESP_GMF_EVENT_STATE_FINISHED
        || event->sub == ESP_GMF_EVENT_STATE_ERROR) {
        xEventGroupSetBits((EventGroupHandle_t)ctx, PIPELINE_BLOCK_BIT);
    }
    return ESP_GMF_ERR_OK;
}
```

## 常见错误

| 错误 | 原因 | 解决方法 |
|---|---|---|
| pipeline 启动后无声音 | 漏调 `esp_gmf_pipeline_loading_jobs` | bind_task 之后必须 loading_jobs 注册 job |
| 报 "No _ in file name" | embed URL 缺 `index_name` 下划线 | 用 `embed://group/0_name.mp3` 格式 |
| 扬声器无声 | codec_dev 采样率与效果链目标不一致 | `esp_codec_dev_open` 的 `fs` 用 menuconfig 的 dest rate/ch/bits |
| 解码失败 | 未 reconfig 解码器，自动检测误判 | 用 `esp_gmf_audio_helper_get_audio_type_by_uri` 后 reconfig |
| 内存不足 | 同时启用太多解码器/效果 | menuconfig 仅启用实际需要的项 |

## 参考

- `gmf_examples/basic_examples/pipeline_play_embed_music/main/play_embed_music.c`
- `docs/en/gmf-framework/gmf-core/gmf-core-pipeline.rst`
- `docs/en/gmf-framework/gmf-elements/gmf-io.rst`（embed_flash 章节）
