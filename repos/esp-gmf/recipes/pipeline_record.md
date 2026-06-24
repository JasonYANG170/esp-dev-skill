# 录音并编码到 SD 卡

> **适用摘要**: 构建「io_codec_dev → aud_enc → io_file」录音流水线，从麦克风采集 PCM 并编码为 AAC/AMR/OPUS 等格式写入 SD 卡，含编码器 reconfig、主动上报格式信息与录音栈配置。

## 触发意图

- "录音到 SD 卡"
- "麦克风采集编码"
- "aud_enc 怎么用"
- "录音 pipeline"
- "AMR/AAC/OPUS 编码录音"

## 前置条件

| 条件 | 要求 |
|---|---|
| 硬件 | 含 ADC（麦克风）与 SD 卡的音频板 |
| 编码格式 | menuconfig 中 `gmf_loader` 启用对应编码器（AAC/AMR/OPUS...） |
| 参考示例 | `gmf_examples/basic_examples/pipeline_record_sdcard` |

## 分步说明

### 1. 初始化录音外设（ADC + SD 卡）

```c
#include "esp_board_manager_includes.h"
#include "esp_codec_dev.h"

#define REC_SR   48000
#define REC_CH   1
#define REC_BITS 16
#define REC_GAIN 32

static int record_peripheral_init(esp_codec_dev_handle_t *rec)
{
    esp_board_manager_init_device_by_name(ESP_BOARD_DEVICE_NAME_FS_SDCARD);
    esp_board_manager_init_device_by_name(ESP_BOARD_DEVICE_NAME_AUDIO_ADC);

    dev_audio_codec_handles_t *h = NULL;
    esp_board_manager_get_device_handle(ESP_BOARD_DEVICE_NAME_AUDIO_ADC, (void **)&h);
    esp_codec_dev_handle_t handle = h->codec_dev;
    esp_codec_dev_set_in_gain(handle, REC_GAIN);
    esp_codec_dev_sample_info_t fs = {
        .sample_rate = REC_SR, .channel = REC_CH, .bits_per_sample = REC_BITS,
    };
    esp_codec_dev_open(handle, &fs);
    *rec = handle;
    return ESP_OK;
}
```

### 2. 建 pool 与录音 pipeline（头 codec_dev，尾 io_file）

```c
#include "esp_gmf_pool.h"
#include "esp_gmf_pipeline.h"
#include "esp_gmf_io_codec_dev.h"
#include "gmf_loader_setup_defaults.h"

esp_gmf_pool_handle_t pool = NULL;
esp_gmf_pool_init(&pool);
gmf_loader_setup_io_default(pool);
gmf_loader_setup_audio_codec_default(pool);
gmf_loader_setup_audio_effects_default(pool);

const char *name[] = {"aud_enc"};
esp_gmf_pipeline_handle_t pipe = NULL;
// 方向：codec_dev（采集）→ aud_enc → io_file（写文件）
esp_gmf_pool_new_pipeline(pool, "io_codec_dev", name, 1, "io_file", &pipe);
esp_gmf_io_codec_dev_set_dev(ESP_GMF_PIPELINE_GET_IN_INSTANCE(pipe), record_handle);

esp_gmf_pipeline_set_out_uri(pipe, "/sdcard/esp_gmf_rec001.aac");
```

### 3. reconfig 编码器并主动上报格式信息

录音方向无解码器解析文件头，必须由应用主动 `report_info` 告知编码器目标格式。

```c
#include "esp_gmf_audio_enc.h"
#include "esp_gmf_audio_helper.h"
#include "esp_amrnb_enc.h"   // AMR bitrate 常量（如使用 AMR）
#include "esp_amrwb_enc.h"

esp_gmf_element_handle_t enc_el = NULL;
esp_gmf_pipeline_get_el_by_name(pipe, "aud_enc", &enc_el);

esp_gmf_info_sound_t info = {
    .sample_rates = REC_SR,
    .channels     = REC_CH,
    .bits         = REC_BITS,
    .bitrate      = 90000,   // AAC 目标码率
};
esp_gmf_audio_helper_get_audio_type_by_uri("/sdcard/esp_gmf_rec001.aac", &info.format_id);

// AMR 需用其专用 bitrate 常量
if (info.format_id == ESP_AUDIO_TYPE_AMRWB)      info.bitrate = ESP_AMRWB_ENC_BITRATE_MD885;
else if (info.format_id == ESP_AUDIO_TYPE_AMRNB) info.bitrate = ESP_AMRNB_ENC_BITRATE_MR122;

esp_gmf_audio_enc_reconfig_by_sound_info(enc_el, &info);
esp_gmf_pipeline_report_info(pipe, ESP_GMF_INFO_SOUND, &info, sizeof(info));
```

### 4. task 栈调大（AMR/AAC 需要 40 KB，可放 PSRAM）

```c
#include "esp_gmf_task.h"

esp_gmf_task_cfg_t cfg = DEFAULT_ESP_GMF_TASK_CONFIG();
cfg.thread.stack       = 40 * 1024;   // AMR 编码栈需求大
cfg.thread.stack_in_ext = true;       // 放 PSRAM 节省 SRAM
cfg.name = "gmf_rec";
esp_gmf_task_handle_t task = NULL;
esp_gmf_task_init(&cfg, &task);
esp_gmf_pipeline_bind_task(pipe, task);
esp_gmf_pipeline_loading_jobs(pipe);
esp_gmf_pipeline_set_event(pipe, _pipeline_event, NULL);

esp_gmf_pipeline_run(pipe);

// 录 10 秒后停止
vTaskDelay(10000 / portTICK_PERIOD_MS);
esp_gmf_pipeline_stop(pipe);

esp_gmf_task_deinit(task);
esp_gmf_pipeline_destroy(pipe);
/* teardown + pool_deinit + 板级 deinit（ADC/SD） */
```

### 5. 运行时改码率（编码器已创建后直接生效）

```c
#include "esp_gmf_audio_enc.h"
esp_gmf_audio_enc_set_bitrate(enc_el, 128000);   // 直接作用于运行中的编码器
esp_gmf_audio_enc_get_bitrate(enc_el, &cur);     // 读当前码率
```

> `esp_gmf_audio_enc_reconfig` / `_by_sound_info` 仅在状态 < `OPENING`（NONE/INITIALIZED）时可替换完整配置。

## 常见错误

| 错误 | 原因 | 解决方法 |
|---|---|---|
| 编码器未初始化即 process | 应用未 `report_info`，aud_enc 不知目标格式 | 显式 `esp_gmf_pipeline_report_info(ESP_GMF_INFO_SOUND,...)` |
| 录音栈溢出/复位 | 默认 4 KB 栈不够 | `stack=40*1024` 且 `stack_in_ext=true` |
| AMR bitrate 非法 | 用了任意数值 | 用 `ESP_AMRWB_ENC_BITRATE_*` / `ESP_AMRNB_ENC_BITRATE_*` 常量 |
| 录音文件无声音 | ADC 增益 0 或 codec 未 open | `esp_codec_dev_set_in_gain` + `esp_codec_dev_open` |
| 单声道板子录单声道失败 | 部分 LYRAT 板只有立体声 ADC | 把 channel 设 2 并配 `channel_mask`（见示例 `#ifdef CONFIG_BOARD_LYRAT_MINI_V1_1`） |

## 参考

- `gmf_examples/basic_examples/pipeline_record_sdcard/main/play_record_sdcard.c`
- `docs/en/gmf-framework/gmf-elements/gmf-audio.rst`（aud_enc 章节）
