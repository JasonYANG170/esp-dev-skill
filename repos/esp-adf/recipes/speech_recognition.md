# 语音识别与录音器：WakeNet 唤醒 + MultiNet 命令词 + VAD + audio_recorder

> **适用摘要**: 用 `audio_recorder` 高层录音器把 AFE（AEC/AGC/NS）、WakeNet 唤醒词、MultiNet 命令词识别、VAD 与编码整合进一个事件驱动的 recorder 管线，通过 `rec_engine_cb` 回调收 `AUDIO_REC_WAKEUP_START` / `AUDIO_REC_VAD_START` / `AUDIO_REC_COMMAND_DECT` 等事件。数据流：mic → i2s(reader) → rsp_filter(16kHz) → raw_stream(reader) → AFE → WakeNet → MultiNet → (可选 encoder) → `audio_recorder_data_read`。底层 VAD（`esp_vad`）也可独立用于纯「是否有人说话」检测。

> Version: selected repo/component version; align with the user project and dependency manifest.
> Evidence: `repos/esp-adf/resources/`, source/examples in `repos/esp-adf/`, and this recipe path `repos/esp-adf/recipes/speech_recognition.md`.
> Validation: draft metadata added from repository routing; verify APIs, Kconfig symbols, and component dependencies against the selected version.

## 触发意图

- "语音唤醒 / 唤醒词 / WakeNet / hi le xin"
- "命令词识别 / MultiNet / 语音命令"
- "audio_recorder / 录音器 / 智能音箱录音"
- "VAD / 语音活动检测 / 判断是否在说话"
- "speech_recognition/wwe 那个例子"

## 前置条件

| 条件 | 要求 |
|---|---|
| 参考示例 | `examples/speech_recognition/wwe/main/main.c`、`examples/speech_recognition/vad/main/example_vad_main.c` |
| 子模块 | `esp-sr`（必须 `git clone --recursive`，提供 WakeNet/MultiNet/AFE/`esp_vad.h` 预编译库） |
| 硬件 | 带麦克风 codec 的板子；WakeNet+MultiNet 默认板 `ESP32-S3-Korvo-2 v3`（`CONFIG_ESP32_S3_KORVO2_V3_BOARD`）。ESP32 不支持 MultiNet |
| Flash 分区 | ESP32-S3 上需在分区表加 `model, data, spiffs, , 4152K,`（存模型，大小看编译日志 `Recommended model partition size`） |
| 头文件 | `components/audio_recorder/include/audio_recorder.h`、`recorder_sr.h`、`recorder_encoder.h`；`esp_vad.h`、`esp_afe_sr_iface.h`、`esp_mn_models.h`（来自 esp-sr） |

## 分步说明

### 1. 两种用法（先选清楚）

| 用法 | 适用 | 核心 API |
|---|---|---|
| 高层 `audio_recorder`（推荐） | 唤醒词 + 命令词 + VAD + 编码 一站式 | `audio_recorder_create` + `rec_engine_cb` |
| 底层 `esp_vad` 独立 | 只判断「是否有人说话」 | `vad_create` + `vad_process` |

### 2. 方式 A：高层 audio_recorder（唤醒 + 命令词 + VAD）

数据流（来自 `docs/en/api-reference/speech-recognition/audio_recorder.rst`）：

```
mic → i2s_stream(reader) → resample(48k→16k) → raw_stream(reader)
             → AFE → WakeNet → MultiNet → (可选 resample+encoder) → audio_recorder_data_read
```

**a. 建一条「喂给 AFE」的录音 pipeline（i2s→filter→raw）**

```c
#include "audio_pipeline.h"
#include "i2s_stream.h"
#include "filter_resample.h"
#include "raw_stream.h"
#include "board.h"

static audio_element_handle_t raw_read;   // AFE 从这里取数据

audio_pipeline_handle_t pipeline;
audio_pipeline_cfg_t pipeline_cfg = DEFAULT_AUDIO_PIPELINE_CONFIG();
pipeline = audio_pipeline_init(&pipeline_cfg);

// 48kHz 采集（板相关宏 CODEC_ADC_I2S_PORT / CODEC_ADC_BITS_PER_SAMPLE 在 board.h）
i2s_stream_cfg_t i2s_cfg = I2S_STREAM_CFG_DEFAULT_WITH_PARA(
    CODEC_ADC_I2S_PORT, 48000, CODEC_ADC_BITS_PER_SAMPLE, AUDIO_STREAM_READER);
audio_element_handle_t i2s_reader = i2s_stream_init(&i2s_cfg);

// 重采样到 16kHz（AFE 要求）
rsp_filter_cfg_t rsp_cfg = DEFAULT_RESAMPLE_FILTER_CONFIG();
rsp_cfg.src_rate = 48000;
rsp_cfg.dest_rate = 16000;
audio_element_handle_t filter = rsp_filter_init(&rsp_cfg);

raw_stream_cfg_t raw_cfg = RAW_STREAM_CFG_DEFAULT();
raw_cfg.type = AUDIO_STREAM_READER;
raw_read = raw_stream_init(&raw_cfg);

audio_pipeline_register(pipeline, i2s_reader, "i2s");
audio_pipeline_register(pipeline, filter,    "filter");
audio_pipeline_register(pipeline, raw_read,  "raw");
const char *link_tag[3] = {"i2s", "filter", "raw"};
audio_pipeline_link(pipeline, link_tag, 3);
audio_pipeline_run(pipeline);
```

> 双麦板（如 Korvo-2 v3 双麦模式）需设 `rsp_cfg.mode = RESAMPLE_UNCROSS_MODE; rsp_cfg.src_ch = 4; rsp_cfg.dest_ch = 4; rsp_cfg.max_indata_bytes = 1024;`。

**b. 给 AFE 喂数据的回调（audio_recorder 通过它读 raw）**

```c
static int input_cb_for_afe(int16_t *buffer, int buf_sz, void *user_ctx, TickType_t ticks)
{
    return raw_stream_read(raw_read, (char *)buffer, buf_sz);
}
```

**c. 配 recorder_sr（AFE + WakeNet + MultiNet）**

```c
#include "recorder_sr.h"
#include "audio_recorder.h"
#include "model_path.h"   // 模型分区 label

char *sr_input_fmt = AUDIO_ADC_INPUT_CH_FORMAT;   // 板相关声道顺序宏

recorder_sr_cfg_t sr_cfg = DEFAULT_RECORDER_SR_CFG(
    sr_input_fmt, "model", AFE_TYPE_SR, AFE_MODE_HIGH_PERF);
sr_cfg.afe_cfg->memory_alloc_mode = AFE_MEMORY_ALLOC_MORE_PSRAM;
sr_cfg.afe_cfg->wakenet_init  = true;     // 关闭则不检测唤醒词
sr_cfg.afe_cfg->vad_mode      = VAD_MODE_4;
sr_cfg.afe_cfg->aec_init      = false;    // 硬件 AEC 由板宏 RECORD_HARDWARE_AEC 决定
sr_cfg.afe_cfg->agc_mode      = AFE_MN_PEAK_NO_AGC;
sr_cfg.multinet_init          = true;
sr_cfg.mn_language            = ESP_MN_CHINESE;   // 或 ESP_MN_ENGLISH
```

> 单麦板（如 Korvo-2 v3 单麦）还需：`sr_cfg.afe_cfg->pcm_config.mic_num = 1; .ref_num = 1; .total_ch_num = 2; sr_cfg.afe_cfg->wakenet_mode = DET_MODE_90;`

**d. 创建 audio_recorder 并注册事件回调**

```c
static audio_rec_handle_t recorder;

static esp_err_t rec_engine_cb(audio_rec_evt_t *event, void *user_data)
{
    if (event->type == AUDIO_REC_WAKEUP_START) {
        recorder_sr_wakeup_result_t *r = event->event_data;
        ESP_LOGI(TAG, "WAKEUP_START vol=%.1f model=%d word=%d",
                 r->data_volume, r->wakenet_model_index, r->wake_word_index);
        // 通常这里播一个 "ding" 提示音
    } else if (event->type == AUDIO_REC_VAD_START) {
        ESP_LOGI(TAG, "VAD_START (开始说话)");
    } else if (event->type == AUDIO_REC_VAD_END) {
        ESP_LOGI(TAG, "VAD_STOP (说完了)");
    } else if (event->type == AUDIO_REC_WAKEUP_END) {
        ESP_LOGI(TAG, "WAKEUP_END (本次唤醒会话结束)");
    } else if (event->type >= AUDIO_REC_COMMAND_DECT) {
        // type 本身就是命令词 id（从 0 起）
        recorder_sr_mn_result_t *r = event->event_data;
        ESP_LOGW(TAG, "command %d, phrase_id=%d, prob=%.2f, str=%s",
                 event->type, r->phrase_id, r->prob, r->str);
    }
    return ESP_OK;
}

audio_rec_cfg_t cfg = AUDIO_RECORDER_DEFAULT_CFG();
cfg.read       = (recorder_data_read_t)input_cb_for_afe;
cfg.sr_handle  = recorder_sr_create(&sr_cfg, &cfg.sr_iface);
cfg.event_cb   = rec_engine_cb;
cfg.vad_off    = 1000;   // 静音超过 1000ms 判定 VAD_END（默认 300ms）
recorder = audio_recorder_create(&cfg);
```

> 关键：`AUDIO_REC_COMMAND_DECT = 0`，命令词事件用 `event->type >= AUDIO_REC_COMMAND_DECT` 捕获，`event->type` 的数值即命令 id。

**e. 动态改命令词（运行时）**

```c
char err[200];
recorder_sr_reset_speech_cmd(cfg.sr_handle,
    "da kai dian deng,kai dian deng;guan bi dian deng,guan dian deng;guan deng;", err);
```

> 命令字符串格式见 `recorder_sr.h` 注释里的 esp-sr 链接：分号分句、逗号列同义词。

**f. 取录音数据（编码后的语音）**

当不需要唤醒词、想手动按键录音时，用 `audio_recorder_trigger_start` 强制开始，VAD 静音或 `trigger_stop` 结束：

```c
// 按下 REC 键
audio_recorder_trigger_start(recorder);
// 主循环或单独 task 里读
uint8_t buf[2048];
int n = audio_recorder_data_read(recorder, buf, sizeof(buf), portMAX_DELAY);
// 松开 REC 键 或 VAD 自动结束
audio_recorder_trigger_stop(recorder);
```

> 若启用了 encoder（见下），`data_read` 返回的就是编码后的 AMR/WAV 字节，可直接写文件。

**g. 可选：接编码器（录成 AMR/WAV）**

```c
#include "recorder_encoder.h"
#include "amrnb_encoder.h"

recorder_encoder_cfg_t enc_cfg = { 0 };
amrnb_encoder_cfg_t amr_cfg = DEFAULT_AMRNB_ENCODER_CONFIG();
amr_cfg.contain_amrnb_header = true;
enc_cfg.encoder = amrnb_encoder_init(&amr_cfg);

cfg.encoder_handle = recorder_encoder_create(&enc_cfg, &cfg.encoder_iface);
```

> AMR-NB 需 8kHz，所以编码器前还要再接一个 resample（16k→8k），见 `wwe/main/main.c` 的 `ENC_2_AMRNB` 分支。

### 3. 方式 B：底层 esp_vad（只判断有人说话）

数据流：`mic → i2s(reader) → rsp_filter(→16kHz mono) → raw → [vad_process]`

```c
#include "esp_vad.h"
#include "raw_stream.h"
#include "filter_resample.h"

#define VAD_SAMPLE_RATE_HZ  16000
#define VAD_FRAME_LENGTH_MS 30
#define VAD_BUFFER_LENGTH   (VAD_FRAME_LENGTH_MS * VAD_SAMPLE_RATE_HZ / 1000)

vad_handle_t vad_inst = vad_create(VAD_MODE_4);   // MODE_0..4，越大越严格
int16_t *vad_buff = malloc(VAD_BUFFER_LENGTH * sizeof(short));

while (1) {
    raw_stream_read(raw_read, (char *)vad_buff, VAD_BUFFER_LENGTH * sizeof(short));
    vad_state_t st = vad_process(vad_inst, vad_buff, VAD_SAMPLE_RATE_HZ, VAD_FRAME_LENGTH_MS);
    if (st == VAD_SPEECH) {
        ESP_LOGI(TAG, "Speech detected");
    }
}
vad_destroy(vad_inst);
```

> `esp_vad.h` 不在 ADF `components/` 下，而在 `esp-sr/include/<chip>/esp_vad.h`。支持采样率 8000/16000/32000，帧长 10/20/30ms。

### 4. 按键触发（periph 回调里调 trigger）

```c
// 注册外设按键回调
esp_periph_set_register_callback(set, periph_callback, NULL);

esp_err_t periph_callback(audio_event_iface_msg_t *event, void *context) {
    if (event->source_type == PERIPH_ID_ADC_BTN
        && (int)event->data == get_input_rec_id()) {
        if (event->cmd == PERIPH_ADC_BUTTON_PRESSED) {
            audio_recorder_trigger_start(recorder);
        } else if (event->cmd == PERIPH_ADC_BUTTON_RELEASE
                || event->cmd == PERIPH_ADC_BUTTON_LONG_RELEASE) {
            audio_recorder_trigger_stop(recorder);
        }
    }
    return ESP_OK;
}
```

## 常见错误

| 错误 | 原因 | 解决方法 |
|---|---|---|
| 找不到 `audio_recorder.h` / `recorder_sr.h` | 未启用 `audio_recorder` 组件 | 工程依赖里加 `REQUIRES audio_recorder` |
| 找不到 `esp_vad.h` / `esp_afe_sr_iface.h` | esp-sr 子模块未拉取 | `git submodule update --init --recursive` |
| ESP32-S3 编译报模型分区不够 / 找不到模型 | 缺 `model` 分区 | 分区表加 `model, data, spiffs, , 4152K,`，大小按编译日志提示 |
| 唤醒不灵 / 误触发 | `wakenet_mode` 灵敏度不合适 | 调 `DET_MODE_90` / `DET_MODE_95` 等；`vad_mode` 调 0-4 |
| MultiNet 命令识别差 | 命令词拼音写错或语言不对 | `mn_language` 用 `ESP_MN_CHINESE`/`ENGLISH`；拼音按 esp-sr 规范 |
| 命令词事件收不到 | 回调里没判 `>=` | 用 `event->type >= AUDIO_REC_COMMAND_DECT`（type 本身是 id） |
| ESP32 上无 MultiNet | MultiNet 仅 S3/C3 等支持 | ESP32 选板时 MultiNet 自动禁用，只能用 WakeNet+VAD |
| AEC 回声没消掉 | `aec_init=false` 或硬件 AEC 未配 | 设 `sr_cfg.afe_cfg->aec_init = true`；参考 voip 例子的 `DEBUG_AEC_INPUT` 调延迟 |
| 任务看门狗超时 | ESP32 跑语音算法算力紧 | wwe README 已知问题；用 S3 或开 PSRAM、`memory_alloc_mode = AFE_MEMORY_ALLOC_MORE_PSRAM` |

## 参考项目

- `examples/speech_recognition/wwe/main/main.c` — 完整唤醒词+命令词+VAD+AMR 编码（默认 ESP32-S3-Korvo-2 v3）
- `examples/speech_recognition/vad/main/example_vad_main.c` — 底层 esp_vad 独立检测（默认 ESP32-LyraT V4.3）
- `docs/en/api-reference/speech-recognition/audio_recorder.rst` — recorder 数据路径图与 API
- `docs/en/api-reference/speech-recognition/esp_vad.rst`、`esp_wn_iface.rst` — VAD / WakeNet 管线说明
- `components/audio_recorder/include/audio_recorder.h`、`recorder_sr.h` — 真实结构与事件枚举
