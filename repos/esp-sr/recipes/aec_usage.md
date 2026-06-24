# AEC 回声消除

> **适用摘要**: 使用 ESP-SR AEC 消除扬声器回声，覆盖三种集成方式（独立 `aec_create`、带 input_format 的 `afe_aec_create`、经 AFE pipeline）以及 SR/FD/VOIP 三类 mode 选型。

## 触发意图

- "回声消除"
- "AEC 怎么用"
- "全双工语音"
- "aec_create / aec_process"
- "VOIP / FD / SR mode"

## 前置条件

| 条件 | 要求 |
|---|---|
| 输入 | 麦克风信号 + 扬声器播放参考信号，16kHz/16-bit |
| 缓冲区 | 16 字节对齐（`heap_caps_aligned_alloc`） |
| 参考 | `test_apps/esp-sr/main/test_afe.cpp`（含 `aec_create`/`afe_aec_create` 对比）、`docs/en/acoustic_echo_cancellation/README.rst` |

## mode 选型

| 场景 | mode | 说明 |
|---|---|---|
| 语音识别（无播放回声 / 简单回声） | `AEC_MODE_SR_LOW_COST` / `AEC_MODE_SR_HIGH_PERF` | 仅线性滤波，省内存快 |
| 全双工对话（如带屏音箱对话） | `AEC_MODE_FD_LOW_COST`（推荐默认）/ `AEC_MODE_FD_HIGH_PERF` | 线性 + 非线性 NLP |
| VoIP 通话 | `AEC_MODE_VOIP_LOW_COST` / `AEC_MODE_VOIP_HIGH_PERF` | 支持 8k/16k |

NLP 等级（仅 FD 生效）：`AEC_NLP_LEVEL_NORMAL` / `AEC_NLP_LEVEL_AGGR`（默认）/ `AEC_NLP_LEVEL_VERYAGGR`。

## 分步说明

### 方式 A：独立 aec_create（精细控制）

```c
#include "esp_aec.h"
#include "esp_heap_caps.h"

// sample_rate 当前仅支持 16000；filter_length 推荐 4（C5 推荐 2）；channel_num=麦数
aec_handle_t *aec = aec_create(16000, 4, 1, AEC_MODE_SR_LOW_COST);
int frame_size = aec_get_chunksize(aec);   // 每帧样本数

// ⚠️ mic/ref/out 必须 16 字节对齐
int16_t *mic = heap_caps_aligned_alloc(16, frame_size * sizeof(int16_t), MALLOC_CAP_8BIT);
int16_t *ref = heap_caps_aligned_alloc(16, frame_size * sizeof(int16_t), MALLOC_CAP_8BIT);
int16_t *out = heap_caps_aligned_alloc(16, frame_size * sizeof(int16_t), MALLOC_CAP_8BIT);

// 每帧处理：mic=麦克风输入，ref=送扬声器的参考，out=去回声后输出
aec_process(aec, mic, ref, out);

aec_destroy(aec);
free(mic); free(ref); free(out);
```

高级配置：

```c
aec_config_t cfg = {
    .mic_num       = 1,
    .ref_num       = 1,
    .out_num       = 1,
    .filter_length = 4,
    .sample_rate   = 16000,
    .caps          = MALLOC_CAP_PSRAM | MALLOC_CAP_8BIT,
    .mode          = AEC_MODE_FD_LOW_COST,
    .nlp_level     = AEC_NLP_LEVEL_AGGR,
};
aec_handle_t *aec = aec_create_from_config(&cfg);
```

### 方式 B：afe_aec_create（带 input_format 解析）

`esp_afe_aec.h` 提供的封装会按 `input_format` 自动分离 mic/ref 通道。**当前只支持 1 麦 + 1 参考**：

```c
#include "esp_afe_aec.h"

// input_format 同 AFE："MNR"=麦、未用、参考
afe_aec_handle_t *h = afe_aec_create("MNR", 4, AFE_TYPE_SR, AFE_MODE_LOW_COST);
int frame_size = h->frame_size;
int nch        = h->pcm_config.total_ch_num;
int mic_idx    = h->pcm_config.mic_ids[0];
int ref_idx    = h->pcm_config.ref_ids[0];

int16_t *in_data  = heap_caps_calloc(1, frame_size * sizeof(int16_t) * nch, MALLOC_CAP_SPIRAM);
int16_t *out_data = heap_caps_aligned_calloc(16, 1, frame_size * sizeof(int16_t), MALLOC_CAP_SPIRAM);

// in_data 是交错多通道；out_data 是去回声的单通道
size_t out_bytes = afe_aec_process(h, in_data, out_data);

afe_aec_destroy(h);
```

### 方式 C：经 AFE pipeline（最常用）

AFE 内部集成 AEC，只需 `input_format` 含 `R` 并设 `aec_init=true`：

```c
afe_config_t *cfg = afe_config_init("MR", models, AFE_TYPE_SR, AFE_MODE_HIGH_PERF);
// "MR" = 1 麦 + 1 参考；AEC 默认开启
cfg->aec_mode          = AEC_MODE_SR_LOW_COST;  // afe_config_t 里的 aec_mode
cfg->aec_filter_length = 4;
cfg->aec_nlp_level     = AEC_NLP_LEVEL_AGGR;

const esp_afe_sr_iface_t *afe = esp_afe_handle_from_config(cfg);
// 运行时开关：
// afe->disable_aec(afe_data); afe->enable_aec(afe_data);
```

> AFE 的 aec_mode 只接受 `AEC_MODE_SR_LOW_COST` / `AEC_MODE_SR_HIGH_PERF`；FD/VOIP 走独立 AEC 或 `AFE_TYPE_FD`/`AFE_TYPE_VC`。

### 拆分线性 / 非线性步骤（高级）

FD 模式可手动分两步：

```c
aec_linear_process(aec, mic, ref, out);   // 仅线性滤波
int n = aec_nlp_process(aec, out);        // 残余回声非线性抑制，返回输出样本数
// 运行时改 NLP：
aec_set_nlp_level(aec, AEC_NLP_LEVEL_VERYAGGR);
```

## 资源占用参考（ESP32-S3，16kHz 单声道）

来自 `docs/en/acoustic_echo_cancellation/README.rst`：

| Mode | Internal RAM | PSRAM | 每帧耗时 | CPU |
|---|---|---|---|---|
| SR_LOW_COST | 18.8 KB | 64.0 KB | 2.29ms / 32ms帧 | 7.2% |
| SR_HIGH_PERF | 8.2 KB | 100.1 KB | 4.51ms | 14.1% |
| FD_LOW_COST | 30.9 KB | 90.0 KB | 6.28ms | 19.6% |
| FD_HIGH_PERF | 20.3 KB | 126.2 KB | 8.08ms | 25.3% |
| VOIP_LOW_COST | 26.9 KB | 64.1 KB | 4.37ms / 16ms帧 | 27.3% |
| VOIP_HIGH_PERF | 69.2 KB | 66.6 KB | 5.05ms | 31.6% |

> SR/FD 帧长 32ms，VOIP 帧长 16ms。

## 常见错误

| 错误 | 原因 | 解决方法 |
|---|---|---|
| 内存对齐异常 / 崩 | mic/ref/out 未 16 字节对齐 | 全部用 `heap_caps_aligned_alloc(16, ...)` |
| 回声没消干净 | mode 太低 / 无 NLP | SR→FD；`aec_set_nlp_level(VERYAGGR)` |
| 近端人声被压扁 | NLP 太激进 | 降为 `AEC_NLP_LEVEL_NORMAL` |
| `aec_process` 多通道格式错 | 期望 `"ch0 ch0..., ch1 ch1..."` | 每通道连续排布，非交错 |
| AFE 里 AEC 没开 | input_format 无 `R` 或 `aec_init=false` | format 加 `R`；`cfg->aec_init=true` |
| VOIP 8k 输入报错 | 用了 AEC 直接 API 但 sample_rate 写 8000 | AEC 独立 API 仅支持 16000；8k 用 `AFE_TYPE_VC_8K` |
| filter_length 过大吃 CPU | 设了 8/16 | 推荐值 4（S3/P4）/ 2（C5） |

## 参考

- `test_apps/esp-sr/main/test_afe.cpp` — `TEST_CASE("test afe aec interface")` 对比 `aec_process` 与 `afe_aec_process`
- `include/esp32s3/esp_aec.h` — `aec_create/create_from_config/process/linear_process/nlp_process/set_nlp_level/get_chunksize`
- `include/esp32s3/esp_afe_aec.h` — `afe_aec_create/process`（input_format 封装）
- `include/esp32s3/esp_afe_config.h` — `afe_config_t` 的 `aec_*` 字段
- `docs/en/acoustic_echo_cancellation/README.rst` — 三类 mode、NLP、资源占用表
- 配套 recipe：`recipes/afe_sr_pipeline.md`
