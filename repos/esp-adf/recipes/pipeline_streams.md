# Stream 速查与组合

> **适用摘要**: 汇总 ESP-ADF 各类 stream 的初始化宏、类型与典型用法，方便按数据源/输出口快速选型。所有 stream ���是 `audio_element_handle_t`，用统一的 register/link 模式接入 pipeline。

> Version: selected repo/component version; align with the user project and dependency manifest.
> Evidence: `repos/esp-adf/resources/`, source/examples in `repos/esp-adf/`, and this recipe path `repos/esp-adf/recipes/pipeline_streams.md`.
> Validation: draft metadata added from repository routing; verify APIs, Kconfig symbols, and component dependencies against the selected version.

## 触发意图

- "用哪个 stream"
- "raw stream 怎么用"
- "SPIFFS 播放"
- "PWM 输出音频"
- "tone stream / 提示音"

## 前置条件

| 条件 | 要求 |
|---|---|
| 参考目录 | `components/audio_stream/include/`（所有 stream 头文件） |
| 子模块 | `esp-adf-libs` 提供 tone/tts 等底层支持 |

## 分步说明

### Stream 速查表（来自 `components/audio_stream/include/*.h`）

| Stream | 初始化宏 | `_init` 函数 | 支持类型 | 典型用途 |
|---|---|---|---|---|
| i2s_stream | `I2S_STREAM_CFG_DEFAULT()` | `i2s_stream_init()` | reader / writer | I2S/PDM/ADC/DAC 接 codec |
| http_stream | `HTTP_STREAM_CFG_DEFAULT()` | `http_stream_init()` | reader / writer | HTTP/HTTPS/HLS 网络流 |
| fatfs_stream | `FATFS_STREAM_CFG_DEFAULT()` | `fatfs_stream_init()` | reader / writer | SD 卡读写 |
| spiffs_stream | `SPIFFS_STREAM_CFG_DEFAULT()` | `spiffs_stream_init()` | reader / writer | SPIFFS 读写 |
| raw_stream | `RAW_STREAM_CFG_DEFAULT()` | `raw_stream_init()` | reader / writer | element 间数据中转（无线程） |
| tcp_client_stream | （见头） | `tcp_client_stream_init()` | reader / writer | TCP 网络流 |
| tone_stream | （见头） | `tone_stream_init()` | reader only | 播放 `mk_audio_tone.py` 生成的提示音 |
| embed_flash_stream | （见头） | `embed_flash_stream_init()` | reader only | 播放 `mk_embed_flash.py` 生成的嵌入音 |
| tts_stream | （见头） | `tts_stream_init()` | reader only | 文本转语音（依赖 esp-sr） |
| pwm_stream | （见头） | `pwm_stream_init()` | writer only | 用 PWM + 滤波做低成本 DAC |
| algorithm_stream | （见头） | `algorithm_stream_init()` | reader only | AEC/AGC/NS 前端算法 |

> reader = 取数据（作 pipeline 输入），writer = 送数据（作 pipeline 输出）。设置方式：`xxx_cfg.type = AUDIO_STREAM_READER` 或 `AUDIO_STREAM_WRITER`。

### 通用接入模式

```c
// 1) 选 stream，设 type
fatfs_stream_cfg_t cfg = FATFS_STREAM_CFG_DEFAULT();
cfg.type = AUDIO_STREAM_READER;
audio_element_handle_t el = fatfs_stream_init(&cfg);

// 2) 注册 + link（顺序 = 数据流）
audio_pipeline_register(pipeline, el, "el");
// ...
```

### raw_stream：element 间数据桥（无线程）

raw stream 不创建任务，仅作 ringbuffer 读写句柄。常用于：
- reader：`[i2s] -> [filter] -> [raw]`，应用从 raw 取处理后的数据
- writer：`[raw] -> [codec-mp3] -> [i2s]`，应用向 raw 喂数据

```c
#include "raw_stream.h"
raw_stream_cfg_t raw_cfg = RAW_STREAM_CFG_DEFAULT();
raw_cfg.type = AUDIO_STREAM_READER;
audio_element_handle_t raw = raw_stream_init(&raw_cfg);
// 应用线程用 audio_element_output / audio_element_input 与之交互
```

### SPIFFS 播放

```c
#include "periph_spiffs.h"      // 先挂载 SPIFFS 外设
// ... esp_periph_set_init + periph_spiffs_init + esp_periph_start ...

spiffs_stream_cfg_t spiffs_cfg = SPIFFS_STREAM_CFG_DEFAULT();
spiffs_cfg.type = AUDIO_STREAM_READER;
audio_element_handle_t spiffs_reader = spiffs_stream_init(&spiffs_cfg);
audio_element_set_uri(spiffs_reader, "/spiffs/test.mp3");
```

### PWM 输出（无 DAC 板的低成本方案）

```c
#include "pwm_stream.h"
pwm_stream_cfg_t pwm_cfg = PWM_STREAM_CFG_DEFAULT();   // writer only
audio_element_handle_t pwm_writer = pwm_stream_init(&pwm_cfg);
// 拓扑：[fatfs/http] -> mp3_decoder -> pwm_stream -> [PWM+滤波电路]
```

> 注：PWM 数模转换信噪比较低，仅适合对音质要求不高的场景。

### tone_stream（提示音）

提示音由 `tools/audio_tone/mk_audio_tone.py` 生成 C 数组，tone_stream 读取并送入解码：
```c
#include "tone_stream.h"
// 配合 fatfs_pipeline 或直接作为 reader，再接 i2s writer
```

## 常见错误

| 错误 | 原因 | 解决方法 |
|---|---|---|
| reader/writer 搞反 | 默认宏多为 WRITER | 录音/读取显式设 `cfg.type = AUDIO_STREAM_READER` |
| SPIFFS 找不到文件 | 未挂载或挂载点不符 | 先 `periph_spiffs` 挂载，URI 前缀 `/spiffs/` |
| PWM 没声 | PWM 引脚/频率未配 | 见 `pipeline_play_mp3_with_dac_or_pwm` 示例 |
| tone/tts 编译失败 | `esp-sr` 子模块缺失 | 递归克隆子模块 |

## 参考项目

- `examples/player/pipeline_spiffs_mp3/` — SPIFFS 播放
- `examples/player/pipeline_play_mp3_with_dac_or_pwm/` — DAC / PWM 输出
- `examples/player/pipeline_flash_tone/` — tone_stream 提示音
- `examples/player/pipeline_embed_flash_tone/` — embed_flash_stream
- `examples/player/pipeline_tts_stream/` — TTS
- `examples/get-started/pipeline_tcp_client/` — TCP client stream
- `examples/advanced_examples/downmix_pipeline/` — raw stream writer 用法
