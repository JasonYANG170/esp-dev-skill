# ESP-SR 状态机与数据流

> 来源：`esp_wn_iface.h`（`wakenet_state_t`）、`esp_mn_iface.h`（`esp_mn_state_t`）、`esp_vad.h`（`vad_state_t`）、`docs/en/audio_front_end/README.rst`、`docs/en/speech_command_recognition/README.rst`、`docs/en/vadnet/README.rst`。

## 1. AFE pipeline 数据流

AFE 是双任务管线：feed 任务喂多通道交错音频，fetch 任务取单声道增强音频 + 各检测器状态。

```
                  ┌─────────── feed() ───────────┐
                  ▼                               │
 多通道交错音频 (M/R/N)                            │
 (16k/16bit, interleaved)                         │
        │                                         │
        ▼                                         │
   ┌─────────── AFE pipeline ──────────────┐      │
   │  [input]                               │      │
   │    -> | AEC (可选, 需 input_format 含 R)|   │      │
   │    -> | SE/BSS 或 NS (单/多麦降噪)     │   │      │
   │    -> | VAD (WebRTC 或 VADNet)         │   │      │
   │    -> | WakeNet (wn9/wn9l/wn9s)        │   │      │
   │    -> | AGC (可选)                     │   │      │
   │  -> [output]                           │   │      │
   └────────────────────────────────────────┘   │      │
        │  print_pipeline() 可打印实际链路       │      │
        ▼                                         │      │
   fetch() 返回 afe_fetch_result_t:              │      │
     .data       单声道 16k/16bit 增强音频 ───────┼──→ 喂给 MultiNet
     .vad_state  VAD_SILENCE / VAD_SPEECH        │
     .wakeup_state WAKENET_DETECTED / NO_DETECT  │
     .wake_word_index 命中唤醒词 (从1)            │
     .vad_cache  防“吃字”补偿数据                 │
```

> 实际开启哪些模块由 `afe_config_init` 按 input_format/chip 自动决定，可用 `afe_config_print` / `print_pipeline` 查看。

## 2. WakeNet 唤醒状态

```c
typedef enum {
    WAKENET_NO_DETECT = 0,         // 未唤醒
    WAKENET_CHANNEL_VERIFIED = -1, // 双/三麦模式：通道已确认（中间态）
    WAKENET_DETECTED = 1           // 唤醒词命中
} wakenet_state_t;
```

独立 WakeNet `detect()` 返回唤醒词 index（>0 命中，0 未命中）。

```
持续 detect(WAKENET_NO_DETECT)
        │
        │ 命中唤醒词
        ▼
WAKENET_DETECTED (wake_word_index>=1)
        │
        │ 应用层接管：开始 MultiNet / 业务
        ▼
   (可选 clean 后回到 NO_DETECT)
```

双/三麦模式（`DET_MODE_2CH_*` / `DET_MODE_3CH_*`）下可能先经过 `WAKENET_CHANNEL_VERIFIED(-1)` 确认通道，再 `>0`。

## 3. MultiNet 命令词状态机（必须配合 WakeNet）

```c
typedef enum {
    ESP_MN_STATE_DETECTING = 0,  // 检测中，未命中
    ESP_MN_STATE_DETECTED  = 1,  // 命中命令词
    ESP_MN_STATE_TIMEOUT   = 2,  // 超时（duration_ms 到）
} esp_mn_state_t;
```

```
唤醒前：MultiNet 不工作
        │
        │ WAKENET_DETECTED
        ▼
ESP_MN_STATE_DETECTING ◄────────────┐
        │ 每帧 detect(res->data)     │
        │                            │
   ┌────┴────┐                       │
   ▼         ▼                       │
DETECTED  DETECTING (持续)           │
   │         │                       │
   │         │ duration_ms 到        │
   │         ▼                       │
   │      TIMEOUT                    │
   │         │                       │
   ▼         ▼                       │
取 get_results():                    │
  num>0: command_id[0]/phrase_id[0]/prob[0]/string
  num=0: 无命中（timeout 时常见）
   │                                 │
   ▼                                 │
clean() + 回到等唤醒 ─────────────────┘
```

**单次识别模式**：`DETECTED` 即退出本轮，回等唤醒。
**连续识别模式**：忽略 `DETECTED` 继续识别，直到 `TIMEOUT` 才退出。

## 4. VAD 状态与 cache

```c
typedef enum { VAD_SILENCE = 0, VAD_SPEECH = 1 } vad_state_t;
```

AFE 内 VAD 有 cache 机制，因为：
1. VAD 算法固有 1~3 帧延迟
2. 需连续触发达 `vad_min_speech_ms` 才确认语音

```
静音段 (VAD_SILENCE)
        │ 检测到语音能量，但未达 min_speech_ms
        ▼
过渡段（首字可能被截） ── AFE 把这段存入 vad_cache
        │ 达到 min_speech_ms
        ▼
VAD_SPEECH
        │ fetch 返回 res->vad_cache_size > 0
        ▼
应用层：先消费 vad_cache，再消费 data  ← 防止“吃字”
        │ 静音达 min_noise_ms
        ▼
VAD_SILENCE
```

## 5. AEC 数据格式

`aec_process` 多通道数据为**连续排布**（非交错）：`ch0 ch0 ch0..., ch1 ch1 ch1...`。
而 AFE feed 的输入是**通道交错**（interleaved）：`ch0[0] ch1[0] ch0[1] ch1[1]...`。
`afe_aec_process` 封装会自动按 `input_format` 拆分，无需手动转换。

## 6. 资源生命周期

```
esp_srmodel_init("model")
        │
        ├──> afe_config_init(fmt, models, type, mode)
        │        │
        │        ├──> esp_afe_handle_from_config(cfg) -> afe_handle
        │        └──> afe_handle->create_from_config(cfg) -> afe_data
        │                  │
        │                  ├──> feed_task / fetch_task 运行
        │                  └──> afe_handle->destroy(afe_data)
        │        afe_config_free(cfg)
        │
        ├──> (可选) multinet->create(name, dur) ... multinet->destroy()
        ├──> (可选) wakenet->create(name, mode) ... wakenet->destroy()
        └──> esp_srmodel_deinit(models)
```

`esp_srmodel_deinit_with_refcount` 适用于多模块共享 models 的场景（引用计数归零才真正释放）。

## 7. 关键 index 约定

| 概念 | 起始 index |
|---|---|
| WakeNet `get_word_name(idx)` / `set_det_threshold(md, thr, idx)` | **1** |
| AFE `set_wakenet_threshold(afe, idx, thr)` | **1 或 2** |
| `esp_mn_commands_get_from_index(i)` | **0** |
| `afe_fetch_result_t.wake_word_index` / `wakenet_model_index` / `trigger_channel_id` | **1** / **1** / **0** |
| command_id（用户定义） | **>=1**（不可为 0） |
