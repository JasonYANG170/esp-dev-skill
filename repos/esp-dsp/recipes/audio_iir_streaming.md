# 实时音频流 IIR / 三缓冲音效处理

> **适用摘要**: 用三缓冲（triple buffer）+ `dsps_biquad_f32` 做实时音效链：从文件/I2S 读 int16 → 转 float → 串联 lowShelf（bass）与 highShelf（treble）biquad → `dsps_mulc_f32` 调音量 → 数字限幅器 → 回 int16 送 codec；运行时通过按钮重新生成 biquad 系数，而滤波器延迟线跨块保持。区别于一次性的 `iir_biquad.md` 静态测试。

> Version: selected repo/component version; align with the user project and dependency manifest.
> Evidence: `repos/esp-dsp/resources/`, source/examples in `repos/esp-dsp/`, and this recipe path `repos/esp-dsp/recipes/audio_iir_streaming.md`.
> Validation: draft metadata added from repository routing; verify APIs, Kconfig symbols, and component dependencies against the selected version.

## 触发意图

- "实时音频 IIR"
- "流式 biquad"
- "三缓冲 / triple buffer"
- "bass / treble / volume 音效"
- "LyraT 音频功放"
- "数字限幅器 / digital limiter"
- "运行时改 IIR 系数"

## 前置条件

| 条件 | 要求 |
|---|---|
| 参考应用 | `applications/lyrat_board_app/main/audio_amp_main.c`（ESP32-LyraT） |
| 头文件 | `esp_dsp.h`（含 `dsps_biquad.h`、`dsps_biquad_gen.h`、`dsps_mulc.h`） |
| Codec 驱动 | `esp_codec_dev_write`（`bsp/esp-bsp.h`，非 ESP-DSP） |
| 音频源 | WAV 文件（SPIFFS），或替换为 I2S 输入 |

## 分步说明

### 静态 vs 流式 IIR 对比

| 维度 | 静态（`iir_biquad.md`） | 流式（本 recipe） |
|---|---|---|
| 输入 | `dsps_d_gen_f32` delta 冲激 | int16 WAV/麦克风波形 |
| 数据类型 | float 全程 | int16 ↔ float 转换环绕滤波 |
| 块结构 | 单次整段 | 按块（如 64 样本）循环 |
| 系数 | 一次性生成 | **运行时按按钮重新生成** |
| 延迟线 `w[]` | 一次清零 | **跨块持续累积**，即使系数变了也不清 |
| 后处理 | 频响图 | 音量 + 数字限幅器 |
| 并发 | 单任务 | 读任务 + 处理/输出任务 + 按钮任务 |

### 三缓冲结构（decouple 读写速率）

直接取自 `audio_amp_main.c`：

```c
#define BUFFER_SIZE (64)

float    processing_audio_buffer[BUFFER_SIZE] = {0};   /* 处理用 float 缓冲 */
int16_t  triple_audio_buffer[3 * BUFFER_SIZE] = {0};   /* 3 个 int16 块：A/B/C */
int      audio_buffer_write_index = 0;   /* 处理任务写 codec 的块号 */
int      audio_buffer_read_index  = 0;   /* 读任务从源填入的块号 */
SemaphoreHandle_t sync_read_task;        /* 处理任务发信号触发读 */
```

约定（与 README 一致）：处理任务把 `write_index` 指向的块写给 codec，然后发信号让读任务把新数据填到 `read_index` 指向的块；二者各自 ++并对 3 取模。当 `write_index == read_index` 表示写比读快、缓冲耗尽，记 `ESP_LOGW` "Audio buffer overflow"。三缓冲保证：当读任务在填某块时，处理任务读写的是另外两块之一，不会踩同一内存。

### 滤波器 + 音量 + 限幅器：静态全局（跨任务共享）

关键点：**biquad 系数可被按钮任务重新生成，但延迟线 `w[]` 永远不被重置**（否则产生爆音）。

```c
/* 系数 + 延迟线：全局，w[] 跨块持续累积 */
float iir_coeffs_lpf[5];
float iir_w_lpf[5] = {0, 0};      /* lowShelf 的延迟线，永不重置 */
float iir_coeffs_hpf[5];
float iir_w_hpf[5] = {0, 0};      /* highShelf 的延迟线，永不重置 */

/* 运行时可调的滤波器参数 */
float lpf_gain = 0;     float lpf_qFactor = 0.5f;  float lpf_freq = 0.01f;  /* bass */
float hpf_gain = 0;     float hpf_qFactor = 1.0f;  float hpf_freq = 0.15f;  /* treble */

/* 音量与限幅器状态 */
float full_volume = 1;            /* 由 full_volume_db 算出 */
int   full_volume_db = -12;
float full_envelope = 0;          /* 限幅器包络，跨块保持 */
```

### 初始化：生成初始系数 + 启动任务

```c
/* 初始音量：dB -> 线性 */
full_volume = exp10f((float)full_volume_db / 20);
/* 初始 lowShelf（bass）与 highShelf（treble）系数 */
dsps_biquad_gen_lowShelf_f32 (iir_coeffs_lpf, lpf_freq, lpf_gain, lpf_qFactor);
dsps_biquad_gen_highShelf_f32(iir_coeffs_hpf, hpf_freq, hpf_gain, hpf_qFactor);

sync_read_task = xSemaphoreCreateCounting(1, 0);
xTaskCreate(audio_read_task,    "audio_read_task",    4096, NULL, 7, NULL);
xTaskCreate(buttons_process_task, "buttons_process_task", 4096, NULL, 4, NULL);
```

### 读任务：转换 → 串联滤波 → 音量 → 限幅 → 回 int16

直接取自 `audio_read_task`，这是整条音效链的核心：

```c
static void audio_read_task(void *arg)
{
    while (1) {
        if (xSemaphoreTake(sync_read_task, 100)) {
            int16_t *wav_buffer = &triple_audio_buffer[audio_buffer_read_index * BUFFER_SIZE];

            /* 从 WAV 文件读一块 int16 */
            uint32_t bytes_read = fread(wav_buffer, sizeof(int16_t), BUFFER_SIZE, play_file);

            /* int16 (Q15) -> float [-1..1] */
            convert_short2float((int16_t *)wav_buffer, processing_audio_buffer, BUFFER_SIZE);

            /* bass：lowShelf biquad（原地） */
            dsps_biquad_f32(processing_audio_buffer, processing_audio_buffer,
                            BUFFER_SIZE, iir_coeffs_lpf, iir_w_lpf);
            /* treble：highShelf biquad（原地） */
            dsps_biquad_f32(processing_audio_buffer, processing_audio_buffer,
                            BUFFER_SIZE, iir_coeffs_hpf, iir_w_hpf);
            /* 音量 */
            dsps_mulc_f32_ansi(processing_audio_buffer, processing_audio_buffer,
                               BUFFER_SIZE, full_volume, 1, 1);
            /* 数字限幅器（防止 bass/treble 抬升后 DAC 饱和） */
            digitalLimiter(processing_audio_buffer, processing_audio_buffer,
                           BUFFER_SIZE, 0.5f, 0.5f, 0.0001f, &full_envelope);

            /* float -> int16 (Q15) */
            convert_float2short(processing_audio_buffer, (int16_t *)wav_buffer, BUFFER_SIZE);

            /* 文件读完：rewind 跳过 WAV 头再续读 */
            if (bytes_read != BUFFER_SIZE) {
                rewind(play_file);
                fread((void *)&wav_header, 1, sizeof(wav_header), play_file);
                bytes_read = fread(wav_buffer, sizeof(int16_t), BUFFER_SIZE, play_file);
            }
            audio_buffer_read_index++;
            if (audio_buffer_read_index >= 3) audio_buffer_read_index = 0;
        } else {
            ESP_LOGE(TAG, "Audio timeout!");
        }
    }
}
```

int16↔float 转换的标定（取自原文件）：`multiplier_in = 1.0 / (INT16_MAX + 1)`，`multiplier_out = (INT16_MAX + 1)`。注意 `INT16_MAX + 1` 会触发 `-Wshift-count-overflow` 类警告时需显式转 `float`。

### 数字限幅器（防止 DAC 溢出）

直接取自 `audio_amp_main.c` 的 `digitalLimiter`：跟踪输入绝对值的包络（attack/release），当包络超过 `threshold` 时按 `threshold/envelope` 压缩，**`*in_envelope` 跨块保持**：

```c
void digitalLimiter(float *input_signal, float *output_signal, int signal_length,
                    float threshold, float attack_value, float release_value, float *in_envelope)
{
    float envelope = *in_envelope;
    for (int i = 0; i < signal_length; i++) {
        float abs_input = fabsf(input_signal[i]);
        if (abs_input > envelope) {
            envelope = envelope * (1 - attack_value) + attack_value * abs_input;
        } else {
            envelope = envelope * (1 - release_value) + release_value * abs_input;
        }
        if (envelope > threshold) {
            output_signal[i] = input_signal[i] * (threshold / envelope);
        } else {
            output_signal[i] = input_signal[i];
        }
    }
    *in_envelope = envelope;   /* 关键：写回跨块状态 */
}
```

参数取自原文件：`threshold=0.5, attack=0.5, release=0.0001`。

### 运行时改系数（按钮任务）

按钮任务**只写 `iir_coeffs_*`，永远不动 `iir_w_*`**：

```c
case BSP_BUTTON_VOLUP:
    if (current_set == AUDIO_BASS) {
        lpf_gain += 1;
        if (lpf_gain > 12) lpf_gain = 12;
        dsps_biquad_gen_lowShelf_f32(iir_coeffs_lpf, lpf_freq, lpf_gain, lpf_qFactor);
    }
    ...
```

这正是流式场景与静态场景的关键差异：系数可以随时换，但 `w[]` 里保存的延迟样本代表滤波器的"记忆"，清零会产出爆音 click。`dsps_biquad_f32` 本身是 无锁 的，按钮任务写系数、读任务下一块读到新系数——这是可接受的折中（系数是 5 个 float，写不满一个 cache line 也不致命；需要严格无伪读时加 `portENTER_CRITICAL`）。

## 常见错误

| 错误 | 原因 | 解决方法 |
|---|---|---|
| 切歌/调音时听到 click | 每次重新生成系数时把 `w[]` 也清零 | `iir_w_*` 只在程序开始清零一次，之后跨块持续复用 |
| 调高 bass 后爆音 | biquad 抬升后超出 ±1.0，int16 回转换饱和 | 必须接 `digitalLimiter`；或在 `convert_float2short` 前显式 clip |
| 三缓冲 overflow | 处理任务比读任务快 | 检查 `audio_buffer_write_index == audio_buffer_read_index` 时 `vTaskDelay(1)` 让出 CPU；调大 BUFFER_SIZE |
| 限幅器不生效 | `*in_envelope` 没写回 | `digitalLimiter` 末尾必须 `*in_envelope = envelope;` |
| 音量明显失真 | 用 dB 直接做乘数 | `full_volume = exp10f(db/20)`，把 dB 转成线性增益 |
| 高频/低频响应反了 | lowShelf/highShelf 颠倒 | bass = lowShelf（`lpf_freq≈0.01`），treble = highShelf（`hpf_freq≈0.15`），都是归一化频率 |
| `dsps_mulc_f32_ansi` 编译告警 | 用了后缀名 | 应用层应调 `dsps_mulc_f32`（原文件用 `_ansi` 是历史遗留；功能等价） |
| 左右声道只听到一边 | 用了单声道 `dsps_biquad_f32` 处理交错立体声 | 单声道函数处理立体声交错数据会串音；立体声用 `dsps_biquad_sf32`（见 `iir_biquad.md`） |

## 参考项目

- `applications/lyrat_board_app/main/audio_amp_main.c` — 完整三缓冲音效链：`convert_short2float` → `dsps_biquad_gen_lowShelf_f32`/`highShelf_f32` → `dsps_biquad_f32`（串联）→ `dsps_mulc_f32_ansi` → `digitalLimiter` → `convert_float2short`
- `applications/lyrat_board_app/README.md` — 三缓冲工作原理、音频处理流水线、按钮控制说明
- `applications/README.md` — 应用列表
- `modules/iir/include/dsps_biquad.h` — `dsps_biquad_f32`
- `modules/iir/include/dsps_biquad_gen.h` — `dsps_biquad_gen_lowShelf_f32`、`dsps_biquad_gen_highShelf_f32`
- `modules/math/mulc/include/dsps_mulc.h` — `dsps_mulc_f32`（应用代码中原文件用 `_ansi` 后缀）
- `recipes/iir_biquad.md` — 静态 biquad 基础（系数布局、单/立体声、级联）
