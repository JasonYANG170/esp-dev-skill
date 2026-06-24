---
name: esp-sr-skill
description: >-
  AI Skill for Espressif ESP-SR speech recognition framework (WakeNet wake word,
  MultiNet command word, AFE audio front-end, VADNet, AEC, Chinese TTS). Used when
  users need to create, modify, or debug offline speech recognition / voice
  applications on ESP32 / ESP32-S3 / ESP32-P4 / ESP32-S31 / ESP32-C3 / ESP32-C5 /
  ESP32-C6 built with ESP-IDF.
  Trigger words: "ESP-SR", "WakeNet", "MultiNet", "AFE", "VADNet", "AEC", "esp-tts",
  "wake word", "speech command", "语音识别", "唤醒词", "命令词", "乐鑫语音", "嗨乐鑫", "Hi ESP"
tags:
  - embedded
  - esp32
  - esp-idf
  - speech-recognition
  - wake-word
  - wakenet
  - multinet
  - audio-front-end
  - vad
  - tts
  - espressif
license: ESPRESSIF MIT
compatibility: ESP32 / ESP32-S2 / ESP32-S3 / ESP32-S31 / ESP32-P4 / ESP32-C3 / ESP32-C5 / ESP32-C6 ; ESP-IDF >= 5.0 (component also depends on espressif/esp-dsp, espressif/dl_fft, espressif/cjson)
metadata:
  author: Community
  version: "1.1.0"
---

# esp-sr-skill

面向 Espressif **ESP-SR** 离线语音识别框架的 AI Skill。提供场景化 recipe、完整的 AFE / WakeNet / MultiNet / VADNet / AEC / TTS API 参考、模型与分区配置说明，以及实战中最常见的踩坑点。所有 API、结构体、宏、模型名、分区配置均来自 `esp-sr` 仓库的 `include/`、`src/include/`、`docs/` 与 `test_apps/`，不臆造任何接口。

> 本仓库是 **component**（ESP-IDF 组件），通常随 [esp-skainet](https://github.com/espressif/esp-skainet) 工程一同下载使用。完整示例工程在 esp-skainet 仓库的 `examples/` 下，本 skill 在 `resources/example_list.md` 中列出仓库自带的 `test_apps/`。

## Core Principles

1. **绝不臆造 API** — 先查 `resources/api_reference.md`；查不到即视为不存在，不得编造函数名或参数。
2. **AFE 是统一入口** — V2.0 起 WakeNet、VAD、NS、AEC、AGC、SE 全部由 AFE pipeline 统一编排，通过 `feed()`/`fetch()` 双任务驱动；单独跑模型只是 `test_apps` 里的测试用法。
3. **配置统一用 `afe_config_init`** — 旧的 `AFE_CONFIG_DEFAULT()`、`ESP_AFE_SR_HANDLE`、`ESP_AFE_VC_HANDLE` 已在 V2.0 移除，必须改用 `afe_config_init(input_format, models, type, mode)` + `esp_afe_handle_from_config()`。
4. **`input_format` 字符串定义通道排列** — `M`=麦克风，`R`=播放参考，`N`=未知/未用；数据必须是 **通道交错（interleaved）** 排布，例如 `MMNR`。
5. **feed/fetch 帧长必须用 `get_*_chunksize` 查询** — 帧长由所选模型与算法决定，写死数值会随配置变化而错位。
6. **MultiNet 帧长必须等于 AFE fetch 帧长** — 把 AFE `fetch` 返回的单声道 16k/16bit 数据直接喂给 `multinet->detect()`。
7. **模型必须在 menuconfig 选择并烧入 `model` 分区** — `partitions.csv` 需含 `model, data, , , 6000K`（大小按实际模型调），`idf.py flash` 自动打包烧录；改模型后重新 `idf.py flash`，调试代码可用 `idf.py app-flash`。
8. **`esp_srmodel_init("model")` 加载模型** — partition_label 必须与分区表里的 `model` 标签一致；用 `esp_srmodel_filter(models, ESP_WN_PREFIX, NULL)` 取模型名。
9. **唤醒与命令词必须配合** — MultiNet 只在唤醒后运行；单次识别在 `ESP_MN_STATE_DETECTED` 退出，连续识别在 `ESP_MN_STATE_TIMEOUT` 退出。
10. **命令词修改后必须调用 `esp_mn_commands_update()`** — add/remove/modify/clear 只是缓存，不调用 update 语言模型不会更新。
11. **音频格式固定为 16k/16bit/单声道（喂给检测器）** — WakeNet/MultiNet/VADNet 输入采样率 16kHz、有符号 16-bit；AFE 内部会处理重采样与通道抽取。
12. **缓冲区 16 字节对齐** — AEC / 独立算法的输入输出缓冲区用 `heap_caps_aligned_alloc(16, ...)` 分配，否则可能触发对齐异常。

## When to Use

**适用：**
- 在 ESP32 / ESP32-S3 / ESP32-P4 / ESP32-S31 上构建离线语音唤醒 + 命令词识别（AFE + WakeNet + MultiNet）
- 配置 AFE pipeline（AEC、NS、VAD、SE/BSS、AGC）
- 单独使用 WakeNet / VADNet / AEC / NS / DOA 模块
- 自定义中文 / 英文命令词（MultiNet5/6/7）
- 集成中文 TTS（esp-tts）
- 从 ESP-SR V1.* 迁移到 V2.*
- 配置模型分区、烧录 `srmodels.bin`、OTA 更新模型

**不适用：**
- 非 ESP 芯片的语音识别（其他厂商 SDK）
- 云端 ASR / 大模型语音对话（ESP-SR 是纯离线、嵌入式方案）
- PCB 设计、麦克风阵列硬件选型（仅给出仓库 `Microphone_Design_Guidelines` 文档指引）
- 在不支持 PSRAM 的芯片上跑 WakeNet9（高配版）；C3/C5/C6 应使用 WakeNet9s

---

## Scenario Quick Reference (Recipes)

匹配到下列场景时，**先读对应 recipe** — 内含完整调用链、分步说明、常见错误与可复制代码。

### 框架与模型配置

| recipe | 场景 |
|---|---|
| `recipes/afe_sr_pipeline.md` | AFE 语音识别主线：menuconfig 选模型 → 分区 → AFE 初始化 → feed/fetch 双任务 |
| `recipes/model_partition.md` | 模型选择、`partitions.csv` 配置、`srmodels.bin` 生成与烧录 |
| `recipes/migration_v1_v2.md` | 从 ESP-SR V1.* 迁移到 V2.0（`afe_config_init` / `esp_afe_handle_from_config`） |

### 唤醒与命令词

| recipe | 场景 |
|---|---|
| `recipes/wakenet_standalone.md` | 单独运行 WakeNet（不经过 AFE，用于测试/低延迟场景） |
| `recipes/multinet_commands.md` | MultiNet 命令词识别：加载、add/update、detect、get_results、单/连续模式 |
| `recipes/custom_commands.md` | 自定义中英文命令词：commands_cn/en.txt、API 增删改、g2p 工具 |

### 算法模块

| recipe | scenario |
|---|---|
| `recipes/vadnet.md` | VADNet 语音活动检测（独立或经 AFE）+ VAD cache 防“吃字” |
| `recipes/aec_usage.md` | AEC 回声消除：独立 `aec_create` / `afe_aec_create` 与各 mode 选型 |
| `recipes/doa_sound_localization.md` | DOA 双麦声源定位（SRP-PHAT）：独立 `esp_doa_*` 与 AFE-aware `afe_doa_*`（input_format 自动解交错） |
| `recipes/afe_vc_pipeline.md` | AFE 语音通信/全双工主线（`AFE_TYPE_VC`/`VC_8K`/`FD`）：pipeline 差异、NS/BSS 选型、8kHz 约束 |
| `recipes/chinese_tts.md` | 中文 TTS：voice_data 分区、`esp_tts_create`、流式合成播放 |

---

## 芯片与模型支持矩阵

来源：仓库 `README.md` 与各模块 `Supported Targets` 标注。

| 模块 | ESP32 | ESP32-S2 | ESP32-S3 | ESP32-S31 | ESP32-P4 | ESP32-C3 | ESP32-C5 | ESP32-C6 |
|---|:--:|:--:|:--:|:--:|:--:|:--:|:--:|:--:|
| WakeNet | wn9 | wn9 | wn9 / wn9l / wn9s | wn9 / wn9l | wn9 / wn9l | wn9s | wn9s | wn9s |
| MultiNet(中) | mn2_cn | — | mn5q8/mn6/mn7 | mn7_cn | mn7_cn | — | — | — |
| MultiNet(英) | — | — | mn5q8/mn6/mn7 | mn7_en | mn7_en | — | — | — |
| AFE | ✓ | ✓ | ✓ | ✓ | ✓ | ✓ | ✓ | ✓ |
| AEC | ✓ | ✓ | ✓ | ✓ | ✓ | ✓ | ✓ | ✓ |
| TTS(中文) | ✓ | ✓ | ✓ | ✓ | ✓ | — | — | — |

> WakeNet9 需要芯片支持 SIMD 且通常需 PSRAM；WakeNet9s 面向无 PSRAM / 无 SIMD 的 C 系列芯片。`wn9l` 在 `wn9` 基础上提升极快语速下的响应率，CPU/内存约为 wn9 的 1.3 倍。

## AFE 模式 / 类型速查

| 枚举 | 含义 |
|---|---|
| `AFE_TYPE_SR` | 语音识别场景，不含非线性降噪 |
| `AFE_TYPE_VC` | 语音通信场景，16kHz 输入，含非线性降噪 |
| `AFE_TYPE_VC_8K` | 语音通信场景，**输入必须为 8kHz** |
| `AFE_TYPE_FD` | 全双工场景，含非线性降噪 |
| `AFE_MODE_LOW_COST` | 低成本模式 |
| `AFE_MODE_HIGH_PERF` | 高性能模式 |

## AEC 模式速查

| 枚举 | 场景 | 说明 |
|---|---|---|
| `AEC_MODE_SR_LOW_COST` / `AEC_MODE_SR_HIGH_PERF` | 语音识别 | 仅线性滤波，内存小、速度快 |
| `AEC_MODE_FD_LOW_COST` / `AEC_MODE_FD_HIGH_PERF` | 全双工对话 | 线性 + 非线性，推荐默认 `FD_LOW_COST` |
| `AEC_MODE_VOIP_LOW_COST` / `AEC_MODE_VOIP_HIGH_PERF` | VoIP 通话 | 支持 8k/16k |

NLP 等级（仅对 FD 模式生效）：`AEC_NLP_LEVEL_NORMAL` / `AEC_NLP_LEVEL_AGGR`（默认）/ `AEC_NLP_LEVEL_VERYAGGR`。

## VAD 模式

`VAD_MODE_0`（Normal）~ `VAD_MODE_4`（Very Very Very Aggressive）。模式越大对语音触发越严格（误触少但可能漏触发）；想让 VAD 触发更多语音请选低模式。

---

## Critical Pitfalls (Must Read)

下面是最常见的错误。违反任何一条都会导致识别不工作或编译/运行失败。

### 1. 不要用已废弃的 V1.* 接口

```c
// ❌ WRONG — V2.0 已移除这些符号，编译报 undefined reference
afe_config_t *afe_config = AFE_CONFIG_INIT("MMNR", models, AFE_INTERNAL);
const esp_afe_sr_iface_t *afe_handle = ESP_AFE_SR_HANDLE;
esp_afe_sr_data_t *afe_data = afe_handle->create(afe_config);

// ✅ CORRECT
srmodel_list_t *models = esp_srmodel_init("model");
afe_config_t *afe_config = afe_config_init("MMNR", models, AFE_TYPE_SR, AFE_MODE_HIGH_PERF);
const esp_afe_sr_iface_t *afe_handle = esp_afe_handle_from_config(afe_config);
esp_afe_sr_data_t *afe_data = afe_handle->create_from_config(afe_config);
```

### 2. 必须先建 model 分区并烧录模型

```makefile
# ❌ WRONG — partitions.csv 没有 model 分区，运行时 esp_srmodel_init 返回空模型列表
# Name,   Type, SubType, Offset, Size
nvs,      data, nvs
phy_init, data, phy

# ✅ CORRECT — 增加 model 分区（label 固定为 "model"）
# Name,   Type, SubType, Offset, Size
nvs,      data, nvs
phy_init, data, phy
model,    data,         ,        ,       6000K
```

烧录：`idf.py flash`（自动打包 srmodels.bin 并烧入 model 分区）；改完代码只调试用 `idf.py app-flash`。

### 3. feed 的帧长/通道数必须查询得到，不能写死

```c
// ❌ WRONG — 帧长随配置变化，写死 512 会错位
int16_t buff[512];
afe_handle->feed(afe_data, buff);

// ✅ CORRECT — 用 get_feed_chunksize / get_feed_channel_num 查询
int feed_chunksize = afe_handle->get_feed_chunksize(afe_data);
int feed_nch       = afe_handle->get_feed_channel_num(afe_data);
int16_t *buff = malloc(feed_chunksize * sizeof(int16_t) * feed_nch);
afe_handle->feed(afe_data, buff);
```

### 4. MultiNet 必须吃 AFE fetch 出来的单声道数据

```c
// ❌ WRONG — 把多通道/未采样率转换的原始 I2S 数据喂给 MultiNet
multinet->detect(model_data, multichannel_raw_i2s);

// ✅ CORRECT — 用 fetch 结果（单声道 16k/16bit）喂给 MultiNet
afe_fetch_result_t *res = afe_handle->fetch(afe_data);
// mu_chunksize == fetch_chunksize
esp_mn_state_t st = multinet->detect(model_data, res->data);
```

### 5. 命令词增删改后必须调用 esp_mn_commands_update()

```c
// ❌ WRONG — 只 add，没 update，模型语言模型没重建，识别不到
esp_mn_commands_add(1, "da kai kong tiao");

// ✅ CORRECT
esp_mn_commands_add(1, "da kai kong tiao");
esp_mn_commands_add(2, "guan bi kong tiao");
esp_mn_error_t *err = esp_mn_commands_update();   // err==NULL 表示全部解析成功
if (err != NULL) { /* err->phrases 是无法解析的错误短语 */ }
```

### 6. 唤醒态判断要用 fetch 结果字段

```c
// ❌ WRONG — 忽略 fetch 返回，无法知道是否唤醒
afe_fetch_result_t *res = afe_handle->fetch(afe_data);

// ✅ CORRECT — 通过 wakeup_state / wake_word_index 判断
afe_fetch_result_t *res = afe_handle->fetch(afe_data);
if (res->wakeup_state == WAKENET_DETECTED) {
    printf("wake word #%d triggered on channel %d\n",
           res->wake_word_index, res->trigger_channel_id);
    // 此后开始把 res->data 喂给 MultiNet
}
```

### 7. WakeNet 阈值范围是 0.4~0.9999，MultiNet 是 0.0~0.9999

```c
// ❌ WRONG — WakeNet 阈值写成 0.1，低于下限不生效
wakenet->set_det_threshold(model_data, 0.1, 1);

// ✅ CORRECT — WakeNet 用 0.4~0.9999
wakenet->set_det_threshold(model_data, 0.8, 1);
// AFE 里则用 set_wakenet_threshold(afe_data, index, 0.8); index=1 或 2
```

### 8. AEC 输入输出缓冲区必须 16 字节对齐

```c
// ❌ WRONG — 普通 malloc，可能未对齐
int16_t *out = malloc(frame_size * sizeof(int16_t));
aec_process(aec, mic, ref, out);

// ✅ CORRECT — 16 字节对齐分配（mic/ref/out 都要对齐）
int16_t *out = heap_caps_aligned_alloc(16,
        frame_size * sizeof(int16_t), MALLOC_CAP_8BIT);
```

### 9. VAD 首帧可能“吃字”，必须处理 vad_cache

```c
// ❌ WRONG �� 直接用 fetch 的 data 录音，第一句话被截掉
fwrite(res->data, 1, res->data_size, fp);

// ✅ CORRECT — 有 vad_cache 时先写 cache 再写 data
if (res->vad_cache_size > 0) {
    fwrite(res->vad_cache, 1, res->vad_cache_size, fp);
}
fwrite(res->data, 1, res->data_size, fp);
```

### 10. menuconfig 不选模型 → 拿不到 handle

```c
// ❌ WRONG — menuconfig 里 WakeNet/MultiNet 都没勾，esp_srmodel_filter 返回 NULL
char *name = esp_srmodel_filter(models, ESP_WN_PREFIX, NULL);
// name == NULL，下面 esp_wn_handle_from_name(NULL) 崩溃

// ✅ CORRECT — 先在 menuconfig > ESP Speech Recognition 里选好模型再烧录，
// 并判空
char *name = esp_srmodel_filter(models, ESP_WN_PREFIX, NULL);
if (name == NULL) {
    ESP_LOGE(TAG, "No wakenet model found, check menuconfig & flash model partition");
    return;
}
```

### 11. AFE 实例用完要 destroy，模型要 deinit

```c
// ❌ WRONG — 反复 create 不 destroy，内存泄漏
while (1) {
    afe_data = afe_handle->create_from_config(afe_config);
    /* ... */
}

// ✅ CORRECT
afe_data = afe_handle->create_from_config(afe_config);
/* ... 运行 ... */
afe_handle->destroy(afe_data);
afe_config_free(afe_config);
esp_srmodel_deinit(models);
```

### 12. C 系列芯片别选 wn9，要选 wn9s

```text
# ❌ WRONG — ESP32-C5 选了 wn9_hilexin，需要 SIMD/PSRAM，跑不动
idf.py menuconfig -> ESP Speech Recognition -> Select WakeNet -> wn9_hilexin

# ✅ CORRECT — C3/C5/C6 选 wn9s 系列（Depthwise Separable Conv，无 PSRAM/SIMD 可跑）
idf.py menuconfig -> ESP Speech Recognition -> Select WakeNet -> wn9s_hilexin
```

---

## Execution Workflow

| Step | Name | Description |
|------|------|-------------|
| 1 | Plan | 明确芯片型号、是否需 PSRAM、唤醒词语言、命令词语言、是否需要 AEC/全双工 |
| 2 | Recipe | 匹配 `recipes/` 场景，按其调用链与代码推进 |
| 3 | Query | 未覆盖的 API 查 `resources/api_reference.md`，配置项查 `resources/config_reference.md` |
| 4 | Validate | 校验 include 路径（`esp_afe_sr_iface.h`、`esp_afe_config.h`、`model_path.h` 等）、函数签名、模型名 |
| 5 | Confirm | 向用户确认：芯片、menuconfig 模型选择、`partitions.csv`、input_format、AFE type/mode |
| 6 | Execute | 在用户 ESP-IDF 工程里改/加文件；新工程可参照 `test_apps/esp-sr/main/` 的结构 |
| 7 | Check | 核对：model 分区存在、模型已烧录、feed/fetch 帧长查询而非写死、命令词已 update、缓冲区对齐 |
| 8 | Build | `idf.py set-target <chip>` → `idf.py menuconfig`（选模型）→ `idf.py flash` |
| 9 | Debug | 串口看 AFE 打印的 pipeline（`afe_config_print` / `print_pipeline`）；用 `esp_mn_active_commands_print` 确认命令词已加载 |

### Step 6 Detail — 参照 test_apps 组织代码

ESP-SR 本身是 component，自身 `test_apps/` 是最贴近真实用法的可编译参考。新工程建议：

1. **主调用骨架**参考 `test_apps/esp-sr/main/test_afe.cpp` 的 `test_feed_Task` / `test_fetch_Task`（feed/fetch 双任务 + `afe_task_into_t`）。
2. **独立模型用法**参考 `test_wakenet.cpp` / `test_multinet.cpp` / `test_vadnet.cpp`。
3. **TTS 用法**参考 `test_apps/esp-tts/main/test_chinese_tts.cpp`。
4. **完整产品级示例**（含 I2S 驱动、LED、OTA）在 esp-skainet 仓库 `examples/`，本 skill 不重复其代码，仅在 `resources/example_list.md` 给出索引。

---

## Failure Strategies

| Situation | Action |
|---|---|
| API 在 `resources/api_reference.md` 查不到 | 停下，告知用户该接口可能不存在或版本不符，不得编造 |
| `esp_srmodel_filter` 返回 NULL | 检查 menuconfig 是否选了对应模型、model 分区是否已烧录、`idf.py flash` 是否执行 |
| 编译报 `undefined reference to AFE_CONFIG_DEFAULT` | 处于 V1.* 写法，按 `recipes/migration_v1_v2.md` 迁移到 `afe_config_init` |
| 唤醒率低 / 漏唤醒 | 调高 `set_wakenet_threshold` 上限内的阈值或换 `DET_MODE_95`；检查麦克风设计（见 docs Microphone Guidelines） |
| 命令词识别不到 | 确认 `esp_mn_commands_update()` 已调用且返回 NULL；用 `print_active_speech_commands` 确认；命令词不含数字/特殊字符 |
| AEC 后仍有回声 | 切换 mode（SR→FD）；调 `aec_set_nlp_level` 到 `VERYAGGR`；检查 ref 通道接线 |
| VAD 截断首字 | 检查并写入 `res->vad_cache`；调大 `vad_delay_ms` |
| 内存不足 | 用 `AFE_MODE_LOW_COST`；MultiNet 用 `ESP_MN_LOAD_FROM_FLASH`；模型选 Q8 版本 |

## References

- 场景 recipe → `recipes/` 目录
- AFE / WakeNet / MultiNet / VADNet / AEC / TTS API → `resources/api_reference.md`
- menuconfig / Kconfig / 分区 / 模型配置 → `resources/config_reference.md`
- 汇总踩坑 → `resources/pitfalls.md`
- 仓库自带可编译示例（test_apps）索引 → `resources/example_list.md`
- 唤醒/识别/VAD 状态机 → `resources/state_machine.md`
