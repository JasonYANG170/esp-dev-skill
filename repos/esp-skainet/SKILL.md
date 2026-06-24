---
name: esp-skainet-skill
description: >-
  AI Skill for ESP-Skainet intelligent voice assistant development. Used when users need to
  create, modify, or debug ESP-Skainet firmware projects based on Espressif ESP32 / ESP32-S3 / ESP32-P4,
  including wake word detection (WakeNet), speech commands recognition (MultiNet), the Audio Front-End
  (AFE: AEC / VAD / BSS / NS / AGC), voice activity detection, direction of arrival, and Chinese TTS.
  Trigger words: "ESP-Skainet", "esp-skainet", "Skainet", "WakeNet", "唤醒词", "MultiNet", "命令词", "AFE", "语音识别", "ESP-SR", "乐鑫语音", "TTS", "语音合成", "ESP32-S3", "Korvo"
tags:
  - embedded
  - esp32
  - esp32-s3
  - esp32-p4
  - voice
  - speech-recognition
  - wake-word
  - wakenet
  - multinet
  - audio-front-end
  - AFE
  - espressif
  - firmware
license: Apache-2.0
compatibility: ESP32 / ESP32-S3 (recommended) / ESP32-P4; build via ESP-IDF v4.4 or v5.x; depends on espressif/esp-sr component (^2.0.0) and a model partition (SPIFFS)
metadata:
  author: Community
  version: "1.1.0"
---

# esp-skainet-skill

AI Skill for ESP-Skainet (乐鑫智能语音助手) 固件开发。提供场景化 recipes、真实 API 参考、配置项速查与高频陷阱。涵盖 WakeNet 唤醒词引擎、MultiNet 命令词识别、Audio Front-End (AFE: AEC/VAD/BSS/NS/AGC)、VAD、DOA 与中文 TTS。所有 API、结构体、宏、Kconfig 符号、文件路径均取自 `esp-skainet` 仓库及其依赖 `esp-sr` 组件的真实头文件与示例，绝不臆造。

## Core Principles

1. **AFE 是核心运行容器** — `esp_afe_sr_iface_t` 以接口结构体形式暴露所有方法（`feed`/`fetch`/`enable_wakenet`/`disable_aec` 等），通过 `esp_afe_handle_from_config()` 获取句柄，再 `create_from_config()` 创建实例。不要凭空调用底层算法函数。
2. **双任务模型固定为 feed + detect(fetch)** — feed 任务调 `afe_handle->feed()`，detect 任务调 `afe_handle->fetch()`；两者用 `xTaskCreatePinnedToCore` 分别绑定到不同核。fetch 返回 `afe_fetch_result_t*`，必须判空并检查 `res->ret_value`。
3. **模型必须先 `esp_srmodel_init` 再传入 AFE** — `esp_srmodel_init("model")` 中的 `"model"` 必须对应 `partitions.csv` 里 SPIFFS 分区的 label。WakeNet/MultiNet 句柄由 `esp_wn_handle_from_name()` / `esp_mn_handle_from_name()` 按模型名获取。
4. **input_format 字符串决定通道编排** — `M`=麦克风、`R`=回采参考、`N`=未知/未用，例如 `"MR"`(单麦+AEC)、`"MM"`(双麦 BSS)、`"MMNR"`。由 `esp_get_input_format()` 返回，直接传给 `afe_config_init()`。通道数必须与 `esp_get_feed_channel()` 一致，否则 `assert` 失败。
5. **AFE_TYPE_SR vs AFE_TYPE_VC 区分场景** — `AFE_TYPE_SR` 用于语音识别（不含非线性降噪）；`AFE_TYPE_VC` 用于语音通信（16 kHz，含非线性降噪 NS）；另有 `AFE_TYPE_VC_8K` / `AFE_TYPE_FD`。降噪深度演示用 `AFE_TYPE_VC`。
6. **MultiNet 命令需 add → update 两步** — `esp_mn_commands_add(id, str)` 加入链表后，必须调用 `esp_mn_commands_update()` 才会刷新语言模型；返回 `esp_mn_error_t*` 列出无法解析的短语。`esp_mn_commands_update_from_sdkconfig()` 则从 Kconfig `CN/EN_SPEECH_COMMAND_IDx` 批量导入。
7. **唤醒后状态机三态** — `detect()` 返回 `ESP_MN_STATE_DETECTING`（继续）/ `ESP_MN_STATE_DETECTED`（命中，取 `get_results()`）/ `ESP_MN_STATE_TIMEOUT`（超时，重新 `enable_wakenet` 进入待唤醒）。唤醒事件由 fetch 结果 `wakeup_state` 字段给出。
8. **多通道唤醒需等 CHANNEL_VERIFIED** — `raw_data_channels == 1` 时直接 `WAKENET_DETECTED` 即可进入命令模式；`raw_data_channels > 1`（多麦 BSS）时必须等到 `WAKENET_CHANNEL_VERIFIED`，再读取 `trigger_channel_id`。
9. **chunksize 必须对齐** — `afe_handle->get_fetch_chunksize()` 与 `multinet->get_samp_chunksize()` 必须相等（示例中 `assert(mu_chunksize == afe_chunksize)`）；feed 帧大小 = `get_feed_chunksize()` × `feed_channel` × sizeof(int16_t)。
10. **WakeNet 阈值 0.4–0.9999** — `afe_handle->set_wakenet_threshold(afe_data, model_index, threshold)`，model_index 仅 1 或 2（AFE 最多同时运行两个 WakeNet 模型）。低于范围或超出索引会失败。
11. **板级初始化走 hardware_driver** — `esp_board_init(16000, channel, 16)` 统一初始化 I2S；`esp_get_feed_data()` 取音；`esp_audio_play()` 播放；`esp_sdcard_init()` 挂载 SD。板子由 `Audio Media HAL` Kconfig 选择（`CONFIG_ESP32_S3_KORVO_1_V4_0_BOARD` 等）。
12. **PSRAM/Flash/分区三件套** — ESP32-S3 需 `CONFIG_SPIRAM=y` + `CONFIG_SPIRAM_MODE_OCT=y` + `CONFIG_SPIRAM_SPEED_80M=y`；Flash ≥ 8 MB（推荐 16 MB QIO）；`partitions.csv` 需 `model,data,spiffs` 分区存放模型。

## When to Use

**Applicable (适用):**
- 在 ESP32 / ESP32-S3 / ESP32-P4 上开发离线语音唤醒（WakeNet）
- 开发离线命令词识别（MultiNet，中/英文，最多 200 条）
- 配置或裁剪 Audio Front-End（AEC 回采、BSS 多麦、NS 降噪、VAD、AGC）
- 集成 VAD（WebRTC / vadnet1）、方向角估计（DOA）、中文 TTS
- 移植/适配新音频开发板（编写 `bsp_board.c`，设置 `input_format`）
- 性能调优（perf_tester 控制台，调整 wakenet 阈值、AFE mode、内存分配模式）

**Not applicable (不适用):**
- 非 ESP 系列芯片（STM32、RK 等）的语音方案
- 云端/在线 ASR（如调用百度/讯飞云端 API）—— 本 skill 仅覆盖本地离线识别
- esp-adf（音频应用框架）整体框架问题 —— esp-skainet 仅借用其部分硬件抽象，不依赖完整 ADF
- 纯 PCB / 音频电路设计（麦克风摆位、电源去耦等硬件设计）

---

## Scenario Quick Reference (Recipes)

当用户意图匹配下列场景时，**先读对应 recipe** —— 它包含完整调用链、分步说明、真实代码与常见错误表。

### 基础 / 唤醒词

| recipe | 场景 |
|---|---|
| `recipes/wake_word_afe.md` | 基于 AFE 的实时唤醒词检测（feed/detect 双任务，含双模型与阈值调节） |
| `recipes/wake_word_raw.md` | 不走 AFE，直接对 PCM/WAV 数据调用 WakeNet（离线文件推理） |

### 命令词识别

| recipe | 场景 |
|---|---|
| `recipes/cn_speech_commands.md` | 中文命令词识别（MultiNet7 + 唤醒后进入命令模式） |
| `recipes/en_speech_commands.md` | 英文命令词识别（MultiNet7 en） |
| `recipes/customize_commands.md` | 运行时自定义命令词（add/update/modify/remove + sdkconfig 导入） |

### AFE / 语音通信 / 降噪

| recipe | 场景 |
|---|---|
| `recipes/afe_config_tuning.md` | AFE 配置项调优（aec/se/ns/vad/agc 开关、mode、内存分配） |
| `recipes/deep_noise_suppression.md` | 深度降噪（AFE_TYPE_VC + NSNET2，含 PCM 落盘） |
| `recipes/voice_communication.md` | 语音通信场景（VC 取增强后单声道数据） |

### 进阶模块

| recipe | 场景 |
|---|---|
| `recipes/voice_activity_detection.md` | VAD 语音活动检测（vad_state / vad_cache / min_speech_ms） |
| `recipes/direction_of_arrival.md` | 双麦方向角估计（esp_doa_create / esp_doa_process） |
| `recipes/chinese_tts.md` | 中文 TTS 语音合成（esp_tts_create / parse_chinese / stream_play） |

### 测试与移植

| recipe | 场景 |
|---|---|
| `recipes/perf_benchmarking.md` | 性能基准与报告生成（perf_tester 控制台 + WakeNet RAR/FAR + pytest CI） |
| `recipes/custom_board_porting.md` | 自定义音频板移植（编写 esp_custom_board.h + bsp_board.c，配置 input_format） |

---

## Board Support (硬件板支持)

板子由 `hardware_driver` 组件的 Kconfig `Audio Media HAL → Audio hardware board` 选择，决定 I2S 引脚与 `input_format`：

| Kconfig symbol | 板子 | 目标芯片 | 典型 input_format |
|---|---|---|---|
| `CONFIG_ESP32_KORVO_V1_1_BOARD` | ESP32-Korvo | ESP32 | `MR`（单麦 + 回采） |
| `CONFIG_ESP32_S3_KORVO_1_V4_0_BOARD` | ESP32-S3-Korvo-1 | ESP32-S3 | `MR`（默认） |
| `CONFIG_ESP32_S3_KORVO_2_V3_0_BOARD` | ESP32-S3-Korvo-2 | ESP32-S3 | 多麦 |
| `CONFIG_ESP32_S3_BOX_BOARD` | ESP32-S3-BOX | ESP32-S3 | 双麦 |
| `CONFIG_ESP32_S3_BOX_3_BOARD` | ESP32-S3-BOX-3 | ESP32-S3 | 双麦 |
| `CONFIG_ESP32_S3_EYE_BOARD` | ESP32-S3-EYE | ESP32-S3 | 单麦 |
| `CONFIG_ESP32_P4_FUNCTION_EV_BOARD` | ESP32-P4-Function-EV | ESP32-P4 | 多麦 |

> 切换板子后必须 `idf.py set-target` + `idf.py menuconfig` 重新选板；`esp_get_input_format()` 的返回值会随之改变，进而影响 `pcm_config.total_ch_num` / `mic_ids` / `ref_ids`。

## Key AFE Configuration (afe_config_t 关键字段)

来自 `esp-sr/include/esp32s3/esp_afe_config.h`：

| 字段 | 类型 | 说明 |
|---|---|---|
| `aec_init` | bool | 是否初始化 AEC（回声消除），需回采通道 `R` |
| `aec_mode` | aec_mode_t | `AEC_MODE_SR_LOW_COST` / `AEC_MODE_SR_HIGH_PERF` |
| `se_init` | bool | 是否初始化 SE（多麦 BSS 盲源分离） |
| `ns_init` | bool | 是否初始化 NS（降噪） |
| `ns_model_name` | char* | NS 模型名，如 `"nsnet2"`；WebRTC 时为 NULL |
| `vad_init` | bool | 是否初始化 VAD |
| `vad_mode` | vad_mode_t | `VAD_MODE_0..4`，越大越易触发 |
| `vad_min_speech_ms` | int | 最短语音时长 ms（>32，默认 128） |
| `vad_min_noise_ms` | int | 最短噪声/静音时长 ms（>64，默认 1000） |
| `wakenet_init` | bool | 是否启用 WakeNet |
| `wakenet_model_name` | char* | WakeNet 模型 1 名 |
| `wakenet_model_name_2` | char* | WakeNet 模型 2 名（可选） |
| `wakenet_mode` | det_mode_t | `DET_MODE_90` / `DET_MODE_95` 等 |
| `agc_init` | bool | 是否初始化 AGC（自动增益） |
| `afe_mode` | afe_mode_t | `AFE_MODE_LOW_COST` / `AFE_MODE_HIGH_PERF` |
| `afe_type` | afe_type_t | `AFE_TYPE_SR` / `AFE_TYPE_VC` / `AFE_TYPE_VC_8K` / `AFE_TYPE_FD` |
| `memory_alloc_mode` | afe_memory_alloc_mode_t | `AFE_MEMORY_ALLOC_MORE_INTERNAL` / `_INTERNAL_PSRAM_BALANCE` / `_MORE_PSRAM` |
| `pcm_config` | afe_pcm_config_t | 通道布局（total_ch_num / mic_ids / ref_ids / sample_rate） |

## Wake / Detect State Machine (唤醒与命令状态机)

```
                     feed task                        detect (fetch) task
mic/I2S ──► esp_get_feed_data() ──► afe_handle->feed() ──► [AFE pipeline]
                                                              │
                          ┌───────────────────────────────────┘
                          ▼
                  afe_handle->fetch()  →  afe_fetch_result_t*
                          │
            ┌─────────────┴───────────────┐
   wakeup_state == WAKENET_DETECTED    wakeup_state == WAKENET_CHANNEL_VERIFIED
   (单通道 / raw_data_channels==1)       (多通道 / raw_data_channels>1，读 trigger_channel_id)
            │
            ▼
   multinet->detect(res->data)  →  ESP_MN_STATE_DETECTING
                                   ESP_MN_STATE_DETECTED  → get_results() 取 command_id/phrase_id/string/prob
                                   ESP_MN_STATE_TIMEOUT   → enable_wakenet() 回到待唤醒
```

**关键枚举值（来自真实头文件）：**
- `wakenet_state_t`: `WAKENET_NO_DETECT=0`, `WAKENET_CHANNEL_VERIFIED=-1`, `WAKENET_DETECTED=1`
- `esp_mn_state_t`: `ESP_MN_STATE_DETECTING=0`, `ESP_MN_STATE_DETECTED=1`, `ESP_MN_STATE_TIMEOUT=2`
- `det_mode_t`: `DET_MODE_90`, `DET_MODE_95`, `DET_MODE_2CH_90`, `DET_MODE_2CH_95`, `DET_MODE_3CH_90`, `DET_MODE_3CH_95`
- `vad_state_t` (来自 fetch 结果): `VAD_SILENCE` / `VAD_SPEECH`

---

## Critical Pitfalls (Must Read)

下面是最常见的错误。违反任何一条都会导致固件无法正常工作。

### 1. feed 帧大小必须用 feed_chunksize × feed_channel

```c
// ❌ WRONG —— 只算了单通道 chunksize，多麦时数据不足导致 AFE 崩溃
int audio_chunksize = afe_handle->get_feed_chunksize(afe_data);
int16_t *i2s_buff = malloc(audio_chunksize * sizeof(int16_t));
esp_get_feed_data(true, i2s_buff, audio_chunksize * sizeof(int16_t));

// ✅ CORRECT —— 乘以真实 feed 通道数
int audio_chunksize = afe_handle->get_feed_chunksize(afe_data);
int feed_channel = esp_get_feed_channel();
int16_t *i2s_buff = malloc(audio_chunksize * sizeof(int16_t) * feed_channel);
esp_get_feed_data(true, i2s_buff, audio_chunksize * sizeof(int16_t) * feed_channel);
```

### 2. esp_srmodel_init 的分区 label 必须与 partitions.csv 一致

```c
// ❌ WRONG —— label 写错，models 为 NULL，AFE 找不到 WakeNet/MultiNet
srmodel_list_t *models = esp_srmodel_init("models");   // partitions.csv 里是 "model"

// ✅ CORRECT —— 与 partitions.csv 中 spiffs 分区的 Name 一致
// partitions.csv:  model, data, spiffs, , 5168K
srmodel_list_t *models = esp_srmodel_init("model");
```

### 3. fetch 必须判空并检查 ret_value

```c
// ❌ WRONG —— 直接解引用 res，偶发 NULL/ESP_FAIL 直接 crash
afe_fetch_result_t *res = afe_handle->fetch(afe_data);
if (res->wakeup_state == WAKENET_DETECTED) { ... }

// ✅ CORRECT
afe_fetch_result_t *res = afe_handle->fetch(afe_data);
if (!res || res->ret_value == ESP_FAIL) {
    printf("fetch error!\n");
    break;
}
if (res->wakeup_state == WAKENET_DETECTED) { ... }
```

### 4. 多通道必须等 CHANNEL_VERIFIED 才进命令模式

```c
// ❌ WRONG —— 双麦板在 WAKENET_DETECTED 就直接跑命令词，trigger_channel_id 还没定
if (res->wakeup_state == WAKENET_DETECTED) {
    wakeup_flag = 1;   // 多麦下此时 channel 未确认
}

// ✅ CORRECT —— 按通道数分支
if (res->raw_data_channels == 1 && res->wakeup_state == WAKENET_DETECTED) {
    wakeup_flag = 1;
} else if (res->raw_data_channels > 1 && res->wakeup_state == WAKENET_CHANNEL_VERIFIED) {
    printf("channel index: %d\n", res->trigger_channel_id);
    wakeup_flag = 1;
}
```

### 5. MultiNet 命令 add 后必须 update

```c
// ❌ WRONG —— 只 add 不 update，模型语言图未刷新，永远识别不到
esp_mn_commands_add(1, "turn on the light");
esp_mn_commands_add(2, "turn off the light");
multinet->detect(model_data, res->data);   // 命令未生效

// ✅ CORRECT —— add 之后 update，并检查返回的错误短语
esp_mn_commands_clear();
esp_mn_commands_add(1, "turn on the light");
esp_mn_commands_add(2, "turn off the light");
esp_mn_error_t *err = esp_mn_commands_update();
if (err != NULL) {
    printf("%d phrases failed to parse\n", err->num);
}
multinet->print_active_speech_commands(model_data);
```

### 6. create_from_config 之前必须先拿到 afe_handle

```c
// ❌ WRONG —— 直接调 afe_config_init 返回值的方法
afe_config_t *cfg = afe_config_init(...);
cfg->create_from_config(cfg);   // afe_config_t 没有方法表

// ✅ CORRECT —— handle 从 config 取，data 从 handle 创建
afe_config_t *afe_config = afe_config_init(esp_get_input_format(), models, AFE_TYPE_SR, AFE_MODE_LOW_COST);
afe_handle = esp_afe_handle_from_config(afe_config);
esp_afe_sr_data_t *afe_data = afe_handle->create_from_config(afe_config);
afe_config_free(afe_config);
```

### 7. set_wakenet_threshold 的 model_index 只能是 1 或 2

```c
// ❌ WRONG —— index 从 0 开始或超过 2，返回 -1 失败
afe_handle->set_wakenet_threshold(afe_data, 0, 0.6);
afe_handle->set_wakenet_threshold(afe_data, 3, 0.6);

// ✅ CORRECT —— index=1 对应 wakenet_model_name，index=2 对应 wakenet_model_name_2
afe_handle->set_wakenet_threshold(afe_data, 1, 0.6);
afe_handle->reset_wakenet_threshold(afe_data, 1);   // 恢复默认
```

### 8. TIMEOUT 后要重新 enable_wakenet 并清状态

```c
// ❌ WRONG —— 超时后不重新使能唤醒，设备再也唤不醒
if (mn_state == ESP_MN_STATE_TIMEOUT) {
    continue;
}

// ✅ CORRECT —— 超时后清 flag、重新使能唤醒词
if (mn_state == ESP_MN_STATE_TIMEOUT) {
    esp_mn_results_t *r = multinet->get_results(model_data);
    printf("timeout, string:%s\n", r->string);
    afe_handle->enable_wakenet(afe_data);
    wakeup_flag = 0;
    continue;
}
```

### 9. DOA 需要关闭 AEC 并手动拆左右声道

```c
// ❌ WRONG —— 直接把多通道交错数据塞给 esp_doa_process
afe_config->aec_init = true;            // AEC 会吃掉一通道
fdoa = esp_doa_process(doa_handle, i2s_buff, i2s_buff);   // 左右声道相同

// ✅ CORRECT —— 关 AEC，按 input_format 里 'M' 的位置拆声道
afe_config->aec_init = false;
char *str = esp_get_input_format();
int pos[10], cnt = 0;
for (int i = 0; str[i]; i++) if (str[i] == 'M') pos[cnt++] = i;
for (int i = 0; i < audio_chunksize; i++) {
    ileft[i]  = i2s_buff[i * feed_channel + pos[0]];
    iright[i] = i2s_buff[i * feed_channel + pos[1]];
}
fdoa = esp_doa_process(doa_handle, ileft, iright);
```

### 10. TTS voice_data 必须从分区 mmap 加载

```c
// ❌ WRONG —— 直接用 &esp_tts_voice_template 当 voice，缺真实发音数据
esp_tts_handle_t *tts = esp_tts_create(&esp_tts_voice_template);

// ✅ CORRECT —— 从 voice_data 分区 mmap 出发音集，再 init voice set
const esp_partition_t *part = esp_partition_find_first(
    ESP_PARTITION_TYPE_DATA, ESP_PARTITION_SUBTYPE_ANY, "voice_data");
const void *voicedata;
esp_partition_mmap_handle_t mmap;
esp_partition_mmap(part, 0, part->size, ESP_PARTITION_MMAP_DATA, &voicedata, &mmap);
esp_tts_voice_t *voice = esp_tts_voice_set_init(&esp_tts_voice_template, (int16_t *)voicedata);
esp_tts_handle_t *tts_handle = esp_tts_create(voice);
```

### 11. ESP32-S3 必须开 PSRAM octal + flash ≥16MB QIO

```makefile
# ❌ WRONG —— 没开 PSRAM，MultiNet/NSNET2 模型加载失败或运行极慢
# sdkconfig.defaults 缺失以下项

# ✅ CORRECT —— sdkconfig.defaults.esp32s3 至少包含
CONFIG_IDF_TARGET="esp32s3"
CONFIG_SPIRAM=y
CONFIG_SPIRAM_MODE_OCT=y
CONFIG_SPIRAM_SPEED_80M=y
CONFIG_ESPTOOLPY_FLASHMODE_QIO=y
CONFIG_ESPTOOLPY_FLASHSIZE_16MB=y
CONFIG_ESP_DEFAULT_CPU_FREQ_MHZ_240=y
CONFIG_ESP32S3_INSTRUCTION_CACHE_32KB=y
CONFIG_ESP32S3_DATA_CACHE_64KB=y
CONFIG_ESP32S3_DATA_CACHE_LINE_64B=y
```

### 12. Kconfig 选模型后才会被编译进固件

```makefile
# ❌ WRONG —— menuconfig 没选 WakeNet/MultiNet，flash 里没有模型，esp_srmodel_filter 返回 NULL
# （未在 menuconfig 中勾选任何 SR_WN_* / SR_MN_* 项）

# ✅ CORRECT —— sdkconfig.defaults.esp32s3 显式指定
CONFIG_SR_VADN_VADNET1_MEDIUM=y
CONFIG_SR_WN_WN9_HIESP=y               # 唤醒词 Hi,ESP
CONFIG_SR_MN_CN_MULTINET7_QUANT=y      # 中文命令词 MultiNet7
# 英文场景：CONFIG_SR_MN_EN_MULTINET7_QUANT=y
```

---

## Execution Workflow

| Step | 名称 | 说明 |
|------|------|------|
| 1 | Plan | 确定场景（唤醒/命令/降噪/VAD/DOA/TTS）、目标芯片、音频板 |
| 2 | Recipe | 在 `recipes/` 找匹配 recipe，按其调用链与代码骨架起步 |
| 3 | Query | recipe 未覆盖的 API，查 `resources/api_reference.md`；配置项查 `resources/config_reference.md` |
| 4 | Validate | 核对：板子 Kconfig、input_format、模型 Kconfig、partitions.csv 的 model 分区、PSRAM 配置 |
| 5 | Confirm | 向用户呈现方案：includes、afe_config、双任务绑定核、命令词列表 |
| 6 | Execute | **新项目**：从最接近的 example（`examples/<name>/`）整体复制再改；**已有项目**：原地编辑 |
| 7 | Check | 核对 feed 帧大小、fetch 判空、多通道 CHANNEL_VERIFIED、命令 update、阈值范围 |
| 8 | Build | `idf.py set-target esp32s3` → `idf.py menuconfig`（选板+模型）→ `idf.py flash monitor` |
| 9 | Debug | 串口监控（115200/退出 Ctrl-]），用 `afe_handle->print_pipeline()` 打印流水线，`print_active_speech_commands()` 看命令 |

### Step 6 Detail — 项目创建策略

**目标目录无现成工程（首次创建）：**

1. 按场景选最接近的 example：
   - 唤醒（实时）→ `examples/wake_word_detection/afe/`
   - 唤醒（离线文件推理）→ `examples/wake_word_detection/wakenet/`
   - 中文命令词 → `examples/cn_speech_commands_recognition/`
   - 英文命令词 → `examples/en_speech_commands_recognition/`
   - 深度降噪 → `examples/deep_noise_suppression/`
   - 语音通信 → `examples/voice_communication/`
   - VAD → `examples/voice_activity_detection/`
   - DOA → `examples/direction_of_arrival/`
   - 中文 TTS → `examples/chinese_tts/`

2. 整体复制该 example 目录（保留 `main/`、`partitions.csv`、`sdkconfig.defaults*`、`CMakeLists.txt`）。

3. 在复制件上改：换板子 Kconfig、改 `sdkconfig.defaults.<target>` 中的模型符号、调 `afe_config`、改命令词。

4. 说明复制来源与改动点。

**目标目录已有工程：** 原地编辑，不要覆盖已有 `sdkconfig`/`partitions.csv` 除非用户要求。

---

## Failure Strategies

| Situation | Action |
|---|---|
| `esp_srmodel_init` 返回 NULL 或 `esp_srmodel_filter` 返回 NULL | 检查 partitions.csv 有 `model` spiffs 分区；检查 Kconfig 是否勾选对应 SR_WN_*/SR_MN_* 模型 |
| `assert(nch == feed_channel)` 失败 | 板子 Kconfig 与 `esp_get_feed_channel()` 不匹配，重新 `set-target`+`menuconfig` 选板 |
| fetch 一直返回 NULL / ESP_FAIL | 检查 feed 任务是否在运行、feed 帧大小是否含通道数、PSRAM 是否真的初始化 |
| 唤醒灵敏度过低/误触发多 | `set_wakenet_threshold(afe_data, 1, x)` 调高(更严)或调低(更灵)，范围 0.4–0.9999 |
| MultiNet 永远识别不到 | 确认 `esp_mn_commands_update()` 已调用；用 `print_active_speech_commands()` 确认；检查 `esp_mn_error_t` |
| 编译报 model not found | 该芯片不支持该模型（如 mn2_cn 仅 ESP32，mn7_* 需 S3/P4/S31），换适配模型 |
| 内存不足 / PSRAM OOM | 调 `memory_alloc_mode = AFE_MEMORY_ALLOC_MORE_PSRAM`；MultiNet 用 `ESP_MN_LOAD_FROM_FLASH` |
| DOA 角度跳变 | 确认 `aec_init=false`、`d_mics` 与实际麦距一致、分辨率参数合理 |

## References

- 场景 recipes → `recipes/` 目录
- API 速查 → `resources/api_reference.md`
- 配置项速查 → `resources/config_reference.md`
- 陷阱汇总 → `resources/pitfalls.md`
- 示例工程索引 → `resources/example_list.md`
- 真实源码：`espressif-repos/esp-skainet/examples/` 与 `espressif-repos/esp-sr/include/esp32s3/`
