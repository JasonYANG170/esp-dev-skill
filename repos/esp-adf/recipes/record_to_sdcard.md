# 录音并编码 WAV/AMR 写入 SD 卡

> **适用摘要**: 用 i2s_stream(reader) 从 codec 麦克风取音，接编码器（wav_encoder / amrwb_encoder / amrnb_encoder），再经 fatfs_stream(writer) 写入 SD 卡。数据流：mic → i2s(reader) → encoder → fatfs(writer) → SD。

> Version: selected repo/component version; align with the user project and dependency manifest.
> Evidence: `repos/esp-adf/resources/`, source/examples in `repos/esp-adf/`, and this recipe path `repos/esp-adf/recipes/record_to_sdcard.md`.
> Validation: draft metadata added from repository routing; verify APIs, Kconfig symbols, and component dependencies against the selected version.

## 触发意图

- "录音到 SD 卡"
- "录制 WAV / AMR"
- "麦克风采集保存文件"
- "pipeline_wav_amr_sdcard 那个例子"

## 前置条件

| 条件 | 要求 |
|---|---|
| 参考示例 | `examples/recorder/pipeline_wav_amr_sdcard/main/pipeline_wav_amr_sdcard.c`、`examples/recorder/pipeline_recording_to_sdcard/` |
| 硬件 | 板载麦克风 codec（如 LyraT、Korvo）+ SD 卡 |
| 编码器 | `wav_encoder.h` / `amrwb_encoder.h` / `amrnb_encoder.h`（来自 `esp-adf-libs`） |

## 分步说明

### 1. 挂载 SD 卡 + 启动 codec（ENCODE 模式）

```c
#include "esp_peripherals.h"
#include "board.h"

esp_periph_config_t periph_cfg = DEFAULT_ESP_PERIPH_SET_CONFIG();
esp_periph_set_handle_t set = esp_periph_set_init(&periph_cfg);
audio_board_sdcard_init(set, SD_MODE_1_LINE);

audio_board_handle_t board = audio_board_init();
// 注意：录音用 ENCODE 模式（驱动 mic / ADC）
audio_hal_ctrl_codec(board->audio_hal, AUDIO_HAL_CODEC_MODE_ENCODE, AUDIO_HAL_CTRL_START);
```

### 2. 创建 i2s reader（关键：type = AUDIO_STREAM_READER）

```c
#include "i2s_stream.h"

i2s_stream_cfg_t i2s_cfg = I2S_STREAM_CFG_DEFAULT();
i2s_cfg.type = AUDIO_STREAM_READER;
i2s_cfg.multi_out_num = 1;            // 给编码器分一支输出
i2s_cfg.task_core = 1;

int sample_rate;
#ifdef CONFIG_CHOICE_AMR_WB
sample_rate = 16000;
#elif defined CONFIG_CHOICE_AMR_NB
sample_rate = 8000;
#else
sample_rate = 44100;                  // WAV
#endif

#if (ESP_IDF_VERSION >= ESP_IDF_VERSION_VAL(5, 0, 0))
i2s_cfg.chan_cfg.id = CODEC_ADC_I2S_PORT;
i2s_cfg.std_cfg.slot_cfg.slot_mode = I2S_SLOT_MODE_MONO;
i2s_cfg.std_cfg.slot_cfg.slot_mask = I2S_STD_SLOT_LEFT;
i2s_cfg.std_cfg.clk_cfg.sample_rate_hz = sample_rate;
#else
i2s_cfg.i2s_port = CODEC_ADC_I2S_PORT;
i2s_cfg.i2s_config.channel_format = I2S_CHANNEL_FMT_ONLY_LEFT;
i2s_cfg.i2s_config.sample_rate = sample_rate;
#endif
audio_element_handle_t i2s_reader = i2s_stream_init(&i2s_cfg);
```

> `CODEC_ADC_I2S_PORT` 由板级 `board.h` 定义（不同板子的 ADC I2S 端口）。

### 3. 创建编码器 + fatfs writer

```c
#include "wav_encoder.h"
#include "fatfs_stream.h"

wav_encoder_cfg_t wav_cfg = DEFAULT_WAV_ENCODER_CONFIG();
audio_element_handle_t wav_encoder = wav_encoder_init(&wav_cfg);

fatfs_stream_cfg_t fatfs_cfg = FATFS_STREAM_CFG_DEFAULT();
fatfs_cfg.type = AUDIO_STREAM_WRITER;
audio_element_handle_t fatfs_writer = fatfs_stream_init(&fatfs_cfg);

// 设置采样率信息给编码器
audio_element_info_t info = AUDIO_ELEMENT_INFO_DEFAULT();
info.sample_rates = sample_rate;
info.channels = 1;
info.bits = 16;
audio_element_setinfo(i2s_reader, &info);
```

> AMR 场景：用 `amrwb_encoder_init(&amrwb_cfg)`（`DEFAULT_AMRWB_ENCODER_CONFIG()`）或 `amrnb_encoder_init(...)`（`DEFAULT_AMRNB_ENCODER_CONFIG()`），分别对应 `.wav` / `.amr` 输出扩展名。

### 4. register + link + 设输出 URI

```c
audio_pipeline_cfg_t pipeline_cfg = DEFAULT_AUDIO_PIPELINE_CONFIG();
audio_pipeline_handle_t pipeline_wav = audio_pipeline_init(&pipeline_cfg);

audio_pipeline_register(pipeline_wav, i2s_reader,   "i2s");
audio_pipeline_register(pipeline_wav, wav_encoder,  "wav_enc");
audio_pipeline_register(pipeline_wav, fatfs_writer, "file");
const char *link_tag[3] = {"i2s", "wav_enc", "file"};
audio_pipeline_link(pipeline_wav, link_tag, 3);

audio_element_set_uri(fatfs_writer, "/sdcard/rec.wav");

audio_pipeline_run(pipeline_wav);
```

### 5. 录指定时长后停止

```c
vTaskDelay(pdMS_TO_TICKS(RECORD_TIME_SECONDS * 1000));

audio_pipeline_stop(pipeline_wav);
audio_pipeline_wait_for_stop(pipeline_wav);
audio_pipeline_terminate(pipeline_wav);
```

## 常见错误

| 错误 | 原因 | 解决方法 |
|---|---|---|
| 录出来是空文件 | i2s 用了 WRITER | 录音 `i2s_cfg.type = AUDIO_STREAM_READER` |
| codec 不采音 | 用了 DECODE 模式 | 录音用 `AUDIO_HAL_CODEC_MODE_ENCODE` |
| 文件无声音 / 速率错 | 采样率/声道未告诉编码器 | `audio_element_setinfo` 设 sample_rates/channels/bits |
| AMR 播放不出来 | 编码器与扩展名不符 | AMR-WB 写 `.amr`，WAV 写 `.wav` |
| IDF v5 编译报 i2s 字段 | 用了 v4 的 `i2s_config` 字段 | 按 `ESP_IDF_VERSION` 分支用 `std_cfg`/`chan_cfg`（见示例） |
| 录音很卡 | 单核跑全部 element | `i2s_cfg.task_core = 1` 分到核 1 |

## 参考项目

- `examples/recorder/pipeline_wav_amr_sdcard/main/pipeline_wav_amr_sdcard.c` — 同时录 WAV 和 AMR 双 pipeline
- `examples/recorder/pipeline_recording_to_sdcard/main/recording_to_sdcard_example.c` — 通用录音到 SD
- `examples/recorder/element_cb_sdcard_amr/` — 用 element 回调直写
