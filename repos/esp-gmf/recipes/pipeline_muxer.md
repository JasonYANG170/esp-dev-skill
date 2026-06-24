# 用 aud_muxer 封装容器录制

> **适用摘要**: 在录音/转码流水线末尾加 `aud_muxer`，把编码后的音频封装为 TS/MP4/FLV/WAV/CAF/OGG/AVI 容器，支持流式输出或分片文件写入。

## 触发意图

- "录音封装成 mp4/ts"
- "aud_muxer 怎么用"
- "容器封装"
- "流式录制"
- "分片录制"

## 前置条件

| 条件 | 要求 |
|---|---|
| 依赖 | `gmf_audio`、`esp_muxer` |
| 参考示例 | `gmf_examples/basic_examples/pipeline_record_audio_muxer` |

## 分步说明

### 1. muxer element 配置

`esp_gmf_audio_muxer_cfg_t` 关键字段：

| 字段 | 说明 |
|---|---|
| `muxer_type` | 容器类型 `ESP_MUXER_TYPE_TS/MP4/FLV/WAV/CAF/OGG/AVI` |
| `codec` | 输入音频的 codec（对应上游 aud_enc 输出，如 `ESP_MUXER_AUDIO_CODEC_AAC`） |
| `output_type` | `ESP_GMF_AUDIO_MUXER_OUTPUT_STREAMING`（经 databus 流式输出）或 `_FILE`（直接写分片文件） |
| `slice_duration` | 仅文件模式有效，每段时长 ms，默认 60000 |
| `url_pattern` / `url_ctx` | 仅文件模式有效，决定每段写到哪个文件的回调 |

### 2. 流式输出模式（经 out_port 转发到下游 IO）

```c
#include "esp_gmf_audio_muxer.h"

esp_gmf_audio_muxer_cfg_t mcfg = {
    .muxer_type  = ESP_MUXER_TYPE_MP4,
    .codec       = ESP_MUXER_AUDIO_CODEC_AAC,
    .output_type = ESP_GMF_AUDIO_MUXER_OUTPUT_STREAMING,
};
esp_gmf_element_handle_t muxer_el = NULL;
esp_gmf_audio_muxer_init(&mcfg, &muxer_el);
```

构造好后注册进 pool，或在手工 pipeline 里串到 aud_enc 之后：

```
io_codec_dev → aud_enc → aud_muxer → io_file（写 .mp4/.ts 文件）
```

> 流式模式下 muxer 的 out_port 连到下游 io_file；element 顺序必须保持 `enc → muxer`，因为 muxer 依赖 aud_enc 上报的 `esp_gmf_info_sound_t` 初始化音频流描述符。

### 3. 分片文件模式（不占 out_port，直接按时长切文件）

```c
// 用回调决定每段文件路径（伪代码：按序号命名）
esp_gmf_audio_muxer_cfg_t mcfg = {
    .muxer_type   = ESP_MUXER_TYPE_TS,
    .codec        = ESP_MUXER_AUDIO_CODEC_AAC,
    .output_type  = ESP_GMF_AUDIO_MUXER_OUTPUT_FILE,
    .slice_duration = 60000,         // 60 秒一段
    .url_pattern  = my_slice_path_cb,
    .url_ctx      = &slice_ctx,
};
```

### 4. 完整 pool 注册 + pipeline

```c
esp_gmf_pool_handle_t pool = NULL;
esp_gmf_pool_init(&pool);
gmf_loader_setup_io_default(pool);
gmf_loader_setup_audio_codec_default(pool);
gmf_loader_setup_audio_effects_default(pool);
// muxer 默认经 gmf_loader_setup_audio_codec_default 注册（tag 为 "aud_muxer"）

const char *name[] = {"aud_enc", "aud_muxer"};
esp_gmf_pipeline_handle_t pipe = NULL;
esp_gmf_pool_new_pipeline(pool, "io_codec_dev", name, 2, "io_file", &pipe);

// 配置 enc（同录音配方：reconfig + report_info）
// 配置 muxer element 参数（取 handle 后改 cfg）
esp_gmf_element_handle_t muxer_el = NULL;
esp_gmf_pipeline_get_el_by_name(pipe, "aud_muxer", &muxer_el);
// 默认参数通常够用；如需改容器类型，可在构造时设好或用运行时方法
```

## 常见错误

| 错误 | 原因 | 解决方法 |
|---|---|---|
| muxer 无输出 | `enc → muxer` 顺序颠倒或未连 | 保持 enc 在 muxer 前，且 enc 已 report_info |
| 播放器打不开生成的文件 | `codec` 与 aud_enc 实际输出不符 | 让 `codec` 字段匹配编码格式 |
| 流式模式下 io_file 拿不到数据 | muxer `output_type` 设成了 FILE | 流式输出用 `OUTPUT_STREAMING` |
| 分片不切 | `slice_duration` 默认 60 s 太长 | 调小或确认 `url_pattern` 回调正确 |

## 参考

- `gmf_examples/basic_examples/pipeline_record_audio_muxer/main/record_audio_muxer.c`
- `docs/en/gmf-framework/gmf-elements/gmf-audio.rst`（Audio Muxer 章节）
