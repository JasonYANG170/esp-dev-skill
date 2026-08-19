# 多流混音渲染（esp_audio_render）

> **适用摘要**: 用 `esp_audio_render` 把多路 PCM 输入（如「音乐 + TTS + 提示音」或 4 轨钢琴合成）混音为统一输出格式后由 writer 回调送出（I2S/BT sink/网络皆可）。每路可独立挂 per-stream 处理链（ALC/EQ/Sonic/Fade），混音后还可挂 post-mix 处理（ALC/limiter）。区别于单流的 `simple_player.md`/`esp_player.md`，本包面向「多轨合成 + 统一输出」。

> Version: selected repo/component version; align with the user project and dependency manifest.
> Evidence: `repos/esp-gmf/resources/`, source/examples in `repos/esp-gmf/`, and this recipe path `repos/esp-gmf/recipes/esp_audio_render.md`.
> Validation: draft metadata added from repository routing; verify APIs, Kconfig symbols, and component dependencies against the selected version.

## 触发意图

- "多路混音"
- "esp_audio_render"
- "音乐 + 提示音叠加"
- "TTS 叠加背景音"
- "多轨道实时合成"
- "运行时切输出采样率"

## 前置条件

| 条件 | 要求 |
|---|---|
| 硬件 | 含 DAC 的音频板（S3-Korvo2 / P4-Function-EV） |
| 组件 | `espressif/esp_audio_render`（依赖 `gmf_core`、`gmf-audio` 效果 element、`esp_codec_dev`） |
| 处理器来源 | `ALC/EQ/Sonic/Fade/ENC` 来自 `cfg.pool`；需提前把对应 element 注册进 pool（用 `gmf_loader_setup_audio_effects_default` 或手工注册） |
| 参考头文件 | `packages/esp_audio_render/include/esp_audio_render.h`、`esp_audio_render_types.h` |
| 参考示例 | `packages/esp_audio_render/examples/audio_render`（open/write/close + 8 流混音）、`simple_piano`（4 轨钢琴实时合成） |

## 分步说明

`esp_audio_render` 把整个系统抽象为一个 render（含多条 stream 与一个 mixed processor）。数据流：**应用 `stream_write(PCM)` → per-stream 处理链（可选）→ ring_fifo → mixer 线程取帧混音 → post-mix 处理链（可选）→ `out_writer` 回调**。`max_stream_num==1` 时跳过 FIFO/混音，直接同步处理输出。

### 1. 注册处理 element 到 pool

render 的处理器按 FourCC 在 `cfg.pool` 里查；至少要注册会用到的 element（ch/bit/rate_cvt 是基础转换，会被自动加，但 ALC/EQ/Sonic/Fade 需显式注册）。

```c
#include "esp_gmf_pool.h"
#include "esp_gmf_ch_cvt.h"
#include "esp_gmf_bit_cvt.h"
#include "esp_gmf_rate_cvt.h"
#include "esp_gmf_alc.h"
#include "esp_gmf_sonic.h"

esp_gmf_pool_handle_t pool = NULL;
esp_gmf_pool_init(&pool);

esp_gmf_element_handle_t el = NULL;
esp_ae_ch_cvt_cfg_t   ch_cfg   = DEFAULT_ESP_GMF_CH_CVT_CONFIG();    esp_gmf_ch_cvt_init(&ch_cfg, &el);   esp_gmf_pool_register_element(pool, el, NULL);
esp_ae_bit_cvt_cfg_t  bit_cfg  = DEFAULT_ESP_GMF_BIT_CVT_CONFIG();   esp_gmf_bit_cvt_init(&bit_cfg, &el);  esp_gmf_pool_register_element(pool, el, NULL);
esp_ae_rate_cvt_cfg_t rate_cfg = DEFAULT_ESP_GMF_RATE_CVT_CONFIG();  esp_gmf_rate_cvt_init(&rate_cfg, &el);esp_gmf_pool_register_element(pool, el, NULL);
esp_ae_alc_cfg_t      alc_cfg  = DEFAULT_ESP_GMF_ALC_CONFIG();       esp_gmf_alc_init(&alc_cfg, &el);     esp_gmf_pool_register_element(pool, el, NULL);
esp_ae_sonic_cfg_t    sonic_cfg= DEFAULT_ESP_GMF_SONIC_CONFIG();     esp_gmf_sonic_init(&sonic_cfg, &el);  esp_gmf_pool_register_element(pool, el, NULL);
```

> 也可改用 `gmf_loader_setup_audio_effects_default(pool)` 批量注册。

### 2. create render（含 writer 回调与输出格式）

writer 回调是最终的 PCM 落点：写到 codec_dev、`esp_bt_audio` 上行 sink、网络推流均可。`out_sample_info` 必须与下游硬件实际配置一致。

```c
#include "esp_audio_render.h"
#include "esp_codec_dev.h"

static int out_writer(uint8_t *pcm, uint32_t size, void *ctx) {
    esp_codec_dev_handle_t dev = (esp_codec_dev_handle_t)ctx;
    return esp_codec_dev_write(dev, pcm, size);   // 返回 0 表成功
}

esp_audio_render_cfg_t cfg = {
    .max_stream_num   = 4,            // 同时混音的最大流数（含 solo 单流场景也 ≥1）
    .out_writer       = out_writer,
    .out_ctx          = codec_dev,
    .out_sample_info  = { .sample_rate = 16000, .bits_per_sample = 16, .channel = 2 },
    .pool             = pool,
    .process_period   = 20,           // ms，混音周期；短则响应快但要求时序精，长则延迟大
    .process_buf_align= 16,           // 硬件加速对齐，0 时默认 16
};
esp_audio_render_handle_t render = NULL;
esp_audio_render_create(&cfg, &render);

// 可选：在 open 任何流之前还能改输出格式 / 重配 task
esp_audio_render_sample_info_t fixed = { .sample_rate = 48000, .bits_per_sample = 16, .channel = 2 };
esp_audio_render_set_out_sample_info(render, &fixed);

esp_gmf_task_config_t task_cfg = { .stack = 6 * 1024, .prio = 5, .core = 0 };
esp_audio_render_task_reconfigure(render, &task_cfg);   // 默认 stack_in_ext=true
```

### 3. 取 stream 句柄、（可选）加 per-stream 处理链

stream 句柄在 `create` 后即可通过 `stream_get` 取得；处理链必须在 `stream_open` 前用 `stream_add_proc` 加。

```c
esp_audio_render_stream_handle_t music = NULL;
esp_audio_render_stream_get(render, ESP_AUDIO_RENDER_FIRST_STREAM, &music);

esp_audio_render_proc_type_t per_stream[] = { ESP_AUDIO_RENDER_PROC_ALC, ESP_AUDIO_RENDER_PROC_SONIC };
esp_audio_render_stream_add_proc(music, per_stream, sizeof(per_stream) / sizeof(per_stream[0]));
```

> 处理类型（`esp_audio_render_proc_type_t`，FourCC 值）：`ALC`/`SONIC`/`EQ`/`FADE`/`ENC`（`ENC` 只能放 mixed 处理器上）。基础 rate/ch/bit 转换由框架自动加，不用 `add_proc`。

### 4. （可选）加 post-mix 处理链（混音后统一处理）

`add_mixed_proc` 必须在任何 stream open 前调；多流时是混音后的统一链，单流时等价于 `stream_add_proc`。

```c
esp_audio_render_proc_type_t post[] = { ESP_AUDIO_RENDER_PROC_ALC };  // 混音后做一次自动电平控制
esp_audio_render_add_mixed_proc(render, post, sizeof(post) / sizeof(post[0]));
```

### 5. open stream（按真实输入格式）、配处理器、write 数据

`stream_open` 触发框架按输入与输出格式自动构造转换链；open 后用 `stream_get_element` 取回 ALC/EQ 句柄做运行时设置。

```c
esp_audio_render_sample_info_t in_info = { .sample_rate = 44100, .bits_per_sample = 16, .channel = 2 };
esp_audio_render_stream_open(music, &in_info);

// 拿到 per-stream ALC 设增益
esp_gmf_element_handle_t alc_el = NULL;
esp_audio_render_stream_get_element(music, ESP_AUDIO_RENDER_PROC_ALC, &alc_el);
if (alc_el) {
    esp_gmf_alc_set_gain(alc_el, 0, -3);
    esp_gmf_alc_set_gain(alc_el, 1, -3);
}

// 喂数据：解码后或合成的 PCM，循环写
while (have_pcm) {
    // 多流时 write 会阻塞直到 ring_fifo 有空位（受 process_period 控制）
    esp_audio_render_stream_write(music, pcm_buf, pcm_size);
}
```

> writer 单流（`max_stream_num==1`）：`stream_write` 同步执行处理链并立即调 `out_writer`；多流时只写到 FIFO，由内部混音线程消费。

### 6. 运行时控制：pause/flush/fade/solo/speed/latency

```c
esp_audio_render_stream_pause(music, true);          // 暂停该流（仅多流有效；暂停后 write 可能阻塞）
esp_audio_render_stream_flush(music);                // 清空该流缓冲
esp_audio_render_stream_set_fade(music, true);       // true=fade in（→target_gain），false=fade out（→initial_gain）

// solo：让指定流独占输出，绕过混音器（运行中随时可切）
esp_audio_render_set_solo_stream(render, ESP_AUDIO_RENDER_STREAM_ID(2));
// 取消 solo、恢复混音：
esp_audio_render_set_solo_stream(render, ESP_AUDIO_RENDER_ALL_STREAM);

// 变速（需先 stream_add_proc 加 SONIC）
esp_audio_render_stream_set_speed(music, 1.2f);

// 查延迟（仅多流）
uint32_t latency_ms = 0;
esp_audio_render_stream_get_latency(music, &latency_ms);
```

### 7. （可选）设每路 mixer gain（必须在 open 前）

默认每路 gain 为 `[0, sqrt(1/max_stream_num)]`，避免混音后削顶。可自定义初始/目标 gain 与过渡时间实现淡入淡出。

```c
esp_audio_render_mixer_gain_t g = {
    .initial_gain   = 0.0f,
    .target_gain    = 0.5f,    // 注意叠加后不要超过 1.0 否则削顶
    .transition_time= 500,     // ms
};
esp_audio_render_stream_set_mixer_gain(music, &g);   // 必须在任何 stream open 之前
```

### 8. 事件回调（用于节能：所有流关掉时挂起设备）

```c
static int on_render_event(esp_audio_render_event_type_t ev, void *ctx) {
    if (ev == ESP_AUDIO_RENDER_EVENT_TYPE_CLOSED) {
        // 所有流已 close，可挂起 codec/进入低功耗
    } else if (ev == ESP_AUDIO_RENDER_EVENT_TYPE_OPENED) {
        // 至少一路 open，需保证设备工作
    }
    return 0;
}
esp_audio_render_set_event_cb(render, on_render_event, NULL);
```

### 9. close 各 stream、destroy render

```c
esp_audio_render_stream_close(music);
/* 其他 stream 同样 close */
esp_audio_render_destroy(render);   // 之后不能再调任何 render API
esp_gmf_pool_deinit(pool);
```

## 常见错误

| 错误 | 原因 | 解决方法 |
|---|---|---|
| `stream_open` 返回 `INVALID_STATE` | 该流已 open | open 前检查状态；同一流不能重复 open，需先 close |
| `add_proc`/`add_mixed_proc` 返回 `INVALID_STATE` | 在 stream open 之后才加处理器 | 所有处理器必须在 open 之前 add |
| `add_mixed_proc` 返回 `NOT_FOUND` | pool 里没有对应 element（如没注册 ALC） | 先 `gmf_loader_setup_audio_effects_default(pool)` 或手工 `pool_register_element` |
| `stream_set_speed` 返回 `NOT_SUPPORTED` | 该流没加 `SONIC` 处理器 | open 前 `stream_add_proc(..., ESP_AUDIO_RENDER_PROC_SONIC, ...)` |
| 混音后削顶/爆音 | 多路 target_gain 之和 > 1.0 | 用默认 `sqrt(1/max_stream_num)` 或手动调小每路 `target_gain` |
| `set_out_sample_info` 返回 `INVALID_STATE` | 已有 stream open | 改输出格式必须在任何 stream open 之前 |
| `task_reconfigure` 返回 `INVALID_STATE` | 已有 stream open | 同上，必须在 open 之前 |
| write 一直阻塞 | 该流被 pause 且 FIFO 满 | 先 `stream_pause(stream, false)` 或 `stream_flush` |
| writer 无输出 | codec_dev 未按 `out_sample_info` open 或单流模式下 write 未发生 | 用 `esp_codec_dev_open` 与 `out_sample_info` 对齐；确认 write 被调用 |
| 多核解码分配不均 | 单核跑多路解码过载 | 像示例一样按 `i % 2` 把解码任务分散到两核 |

## 参考

- `packages/esp_audio_render/examples/audio_render/main/main.c`（单流 + 8 流混音两个 case，per-stream + post-mix ALC）
- `packages/esp_audio_render/examples/simple_piano/main/piano_example.c`（4 轨实时合成混音）
- `packages/esp_audio_render/include/esp_audio_render.h`、`esp_audio_render_types.h`
- `docs/en/gmf-framework/gmf-package/esp-audio-render.rst`、`docs/zh_CN/gmf-framework/gmf-package/esp-audio-render.rst`
