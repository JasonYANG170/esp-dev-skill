# AFE 配置项调优

> **适用摘要**: 理解并调整 `afe_config_t` 各字段，按场景开启/关闭 AEC、SE(BSS)、NS、VAD、AGC，选择 AFE type/mode 与内存分配策略。所有字段取自 `esp_afe_config.h`。

## 触发意图

- "AFE 配置"
- "afe_config_init 参数"
- "开关 AEC / NS / VAD"
- "AFE_MODE_LOW_COST vs HIGH_PERF"
- "AFE_TYPE_SR vs VC"

## 前置条件

| 条件 | 要求 |
|---|---|
| 参考工程 | 任意带 AFE 的 example（如 `examples/wake_word_detection/afe/`） |
| 头文件 | `esp_afe_config.h` |
| 关键概念 | `afe_config_init()` 会根据 input_format 与 models 给出"尽可能全开"的默认值，再手动微调 |

## 分步说明

### 1. 四种 AFE type（场景）

```c
// 来自 esp_afe_config.h 的 afe_type_t
AFE_TYPE_SR    = 0,  // 语音识别：不含非线性降噪（最常用，WakeNet+MultiNet）
AFE_TYPE_VC    = 1,  // 语音通信：16kHz，含非线性降噪 NS
AFE_TYPE_VC_8K = 2,  // 语音通信：8kHz 输入（必须 8kHz 喂入）
AFE_TYPE_FD    = 3,  // 全双工：含非线性降噪

afe_config_t *cfg = afe_config_init(esp_get_input_format(), models,
                                    AFE_TYPE_SR, AFE_MODE_LOW_COST);
```

### 2. 两种 AFE mode（性能/成本）

```c
AFE_MODE_LOW_COST = 0,  // 低成本，占用少
AFE_MODE_HIGH_PERF = 1, // 高性能，算法更重
```

### 3. AEC（回声消除）—— 需要回采通道 R

```c
// input_format 含 'R' 才有意义，如 "MR"（单麦+回采）
cfg->aec_init   = true;
cfg->aec_mode   = AEC_MODE_SR_LOW_COST;   // 或 AEC_MODE_SR_HIGH_PERF
cfg->aec_filter_length = ...;              // 滤波器长度
cfg->aec_nlp_level = AEC_NLP_LEVEL_AGGR;   // 回声抑制等级
```

### 4. SE（多麦盲源分离 BSS）与 NS（降噪）

```c
cfg->se_init = true;          // 双麦及以上时启用 BSS
cfg->ns_init = true;
cfg->ns_model_name  = "nsnet2";   // 深度降噪；WebRTC 时留 NULL
cfg->afe_ns_mode    = AFE_NS_MODE_NET;   // 或 AFE_NS_MODE_WEBRTC
```

> 二选一规则（来自 `afe_config_check` 注释）：双通道输入时优先 SE(BSS)，若关闭 SE 则只用第一路麦克风。

### 5. VAD（语音活动检测）

```c
cfg->vad_init         = true;
cfg->vad_mode         = VAD_MODE_1;   // VAD_MODE_0..4，越大越易触发
cfg->vad_model_name   = NULL;         // NULL=WebRTC VAD；或填 vadnet1 模型名
cfg->vad_min_speech_ms = 128;         // 最短语音 ms（>32）
cfg->vad_min_noise_ms  = 1000;        // 最短噪声/静音 ms（>64）
cfg->vad_delay_ms      = 128;         // 首帧语音延迟 ms
cfg->vad_mute_playback = false;       // true 时 VAD 检测期间静音回放
cfg->vad_enable_channel_trigger = false;
```

### 6. AGC（自动增益）

```c
cfg->agc_init               = true;
cfg->agc_mode               = AFE_AGC_MODE_WAKENET; // 增益由 WakeNet 算（若启用）
cfg->agc_compression_gain_db = 9;   // 压缩增益 dB
cfg->agc_target_level_dbfs   = 3;   // 目标电平 -dBfs
```

另有 fetch 输出峰值 AGC 模式（作用于识别后音频）：
```c
// afe_mn_peak_agc_mode_t: AFE_MN_PEAK_AGC_MODE_1(-9dB)/_2(-6dB)/_3(-3dB)/AFE_MN_PEAK_NO_AGC(0)
```

### 7. 内存分配模式（PSRAM 策略）

```c
cfg->memory_alloc_mode = AFE_MEMORY_ALLOC_MORE_INTERNAL;          // 尽量内部 RAM
// 或
cfg->memory_alloc_mode = AFE_MEMORY_ALLOC_INTERNAL_PSRAM_BALANCE; // 均衡
// 或
cfg->memory_alloc_mode = AFE_MEMORY_ALLOC_MORE_PSRAM;             // 尽量 PSRAM（无 PSRAM 板禁用）
```

### 8. 其他实用字段

```c
cfg->pcm_config.total_ch_num = ...;   // 总通道（含 M/R/N）
cfg->pcm_config.mic_ids      = ...;   // 麦克风通道索引数组
cfg->pcm_config.ref_ids      = ...;   // 回采通道索引数组
cfg->pcm_config.sample_rate  = 16000;
cfg->afe_perferred_core      = 1;     // SE 任务优选核
cfg->afe_perferred_priority  = 5;
cfg->afe_ringbuf_size        = 50;    // 环形缓冲帧数
cfg->afe_linear_gain         = 1.0;   // 线性增益 [0.1, 10.0]
cfg->fixed_first_channel     = false; // true=首次唤醒后通道固定到原始麦
cfg->fixed_output_channel    = false; // true=输出固定到麦克风通道
cfg->output_playback_channel = false; // true=fetch 输出回采参考通道
```

### 9. 运行时开关（无需重建 AFE）

```c
afe_handle->disable_wakenet(afe_data);  afe_handle->enable_wakenet(afe_data);
afe_handle->disable_aec(afe_data);      afe_handle->enable_aec(afe_data);
afe_handle->disable_se(afe_data);       afe_handle->enable_se(afe_data);
afe_handle->disable_vad(afe_data);      afe_handle->enable_vad(afe_data);
afe_handle->disable_ns(afe_data);       afe_handle->enable_ns(afe_data);
afe_handle->disable_agc(afe_data);      afe_handle->enable_agc(afe_data);
afe_handle->reset_vad(afe_data);
afe_handle->print_pipeline(afe_data);   // 打印流水线，如 [input]->|AEC|->|WakeNet(wn9_hiesp)|->[output]
```

## 常见错误

| 错误 | 原因 | 解决方法 |
|---|---|---|
| AEC 无效果 | input_format 没有 `R` 回采通道 | 板子需接回采，input_format 设为 `MR` |
| 双麦仍用 NS | 双通道默认走 SE(BSS)，NS 被忽略 | 关 SE 才会退回单麦+NS |
| 内存不足崩溃 | 全开算法 + 无 PSRAM | `memory_alloc_mode=_MORE_PSRAM`，或关 SE/NS |
| `afe_config_check` 改了我的配置 | 存在算法冲突被自动修正 | 调用后 `afe_config_print(cfg)` 看实际值 |
| AGC 把唤醒搞失真 | `agc_mode` 选错 | 识别场景用 `AFE_AGC_MODE_WAKENET` |

## 参考

- `espressif-repos/esp-sr/include/esp32s3/esp_afe_config.h` — `afe_config_t` 全字段、`afe_config_init/check/print/free`
- `espressif-repos/esp-sr/include/esp32s3/esp_afe_sr_iface.h` — 运行时 enable/disable 方法表
- `examples/wake_word_detection/afe/main/main.c` — `afe_config_init` 真实用法
- `examples/voice_activity_detection/main/main.c` — vad_mode / vad_min_speech_ms 调节示例
