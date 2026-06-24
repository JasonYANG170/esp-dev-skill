# 音频处理元素：EQ / Downmix / Sonic / Resample / ALC

> **适用摘要**: 在 pipeline 中插入音频处理 element 实现均衡器（EQ）、混音（Downmix）、变速变调（Sonic）、重采样（Resample Filter）、自动电平控制（ALC）。这些 element 通常位于解码器与输出 stream 之间。

## 触发意图

- "加均衡器 / EQ"
- "混音 / 背景音乐叠加"
- "变速变调 / Sonic"
- "重采样 / resample"
- "ALC 自动音量"

## 前置条件

| 条件 | 要求 |
|---|---|
| 参考目录 | `examples/audio_processing/`、`examples/advanced_examples/` |
| 文档 | `docs/en/api-reference/audio-processing/`（equalizer、downmix、audio_sonic、filter_resample） |
| 子模块 | `esp-adf-libs`（提供处理算法） |

## 分步说明

### 处理元素速查

| 处理 | 配置宏 | init 函数 | 典型位置 |
|---|---|---|---|
| 均衡器 | `DEFAULT_EQUALIZER_CONFIG(...)` | `equalizer_init()` | decoder → **eq** → i2s |
| 混音 | `DEFAULT_DOWNMIX_CONFIG()` | `downmix_init()` | 多路输入 → **downmix** → i2s |
| 变速变调 | `DEFAULT_SONIC_CONFIG(...)` | `sonic_init()` | decoder → **sonic** → i2s |
| 重采样 | `DEFAULT_RSP_FILTER_CONFIG()` | `rsp_filter_init()` | decoder → **rsp** → i2s |
| ALC | （i2s stream 内置） | `i2s_cfg.use_alc = true` | i2s stream 配置 |

### 1. 均衡器（EQ）

```c
#include "equalizer.h"
equalizer_cfg_t eq_cfg = DEFAULT_EQUALIZER_CONFIG();   // 见头文件具体增益参数
audio_element_handle_t eq = equalizer_init(&eq_cfg);

// 拓扑：http/fatfs → mp3_decoder → equalizer → i2s_stream
audio_pipeline_register(pipeline, mp3_decoder, "mp3");
audio_pipeline_register(pipeline, eq,           "eq");
audio_pipeline_register(pipeline, i2s_writer,   "i2s");
const char *link_tag[3] = {"mp3", "eq", "i2s"};
audio_pipeline_link(pipeline, link_tag, 3);
```

### 2. 重采样（Resample Filter）

用于把源采样率转换成 codec/I2S 期望的采样率：
```c
#include "rsp_filter.h"
rsp_filter_cfg_t rsp_cfg = DEFAULT_RSP_FILTER_CONFIG();   // 按头文件设 src/dst 采样率
audio_element_handle_t rsp = rsp_filter_init(&rsp_cfg);
// 拓扑：... → mp3_decoder → rsp_filter → i2s_stream
```

### 3. 变速变调（Sonic）

```c
#include "audio_sonic.h"
sonic_cfg_t sonic_cfg = DEFAULT_SONIC_CONFIG();   // 设 speed / pitch / volume
audio_element_handle_t sonic = sonic_init(&sonic_cfg);
// 运行中可动态调整（见示例）
```

### 4. 混音（Downmix）

把多路音频（如背景音乐 + 提示音）混合。需要多输入 ringbuffer，参考 `examples/advanced_examples/downmix_pipeline/` 与 `examples/advanced_examples/audio_mixer_tone/`。

### 5. ALC（i2s stream 内置）

ALC 直接在 i2s stream ���置里开启：
```c
i2s_stream_cfg_t i2s_cfg = I2S_STREAM_CFG_DEFAULT();
i2s_cfg.use_alc = true;
i2s_cfg.volume  = 0;        // ALC 目标音量
```

## 常见错误

| 错误 | 原因 | 解决方法 |
|---|---|---|
| 处理后无声音 | 处理 element 采样率与上下文不匹配 | 用 rsp_filter 对齐；EQ 前确认采样率 |
| CPU 占用高 | 算法重 + 单核 | 降采样率 / 分核 / 关未用处理 |
| Downmix 只有一路 | 多输入 ringbuffer 未配 | 参考 downmix_pipeline 的 multi_in_rb 配置 |
| 参数编译错 | 用了未导出的字段 | 以 `components/*/include` 头文件为准 |

## 参考项目

- `examples/audio_processing/pipeline_equalizer/` — EQ
- `examples/audio_processing/pipeline_resample/` — Resample
- `examples/audio_processing/pipeline_sonic/` — Sonic 变速变调
- `examples/audio_processing/pipeline_alc/` — ALC
- `examples/advanced_examples/downmix_pipeline/` — Downmix 混音
- `examples/advanced_examples/audio_mixer_tone/` — 提示音混音
- `examples/audio_processing/pipeline_audio_forge/` — 综合处理
