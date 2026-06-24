# ESP-SR Configuration Reference

> 来源：`Kconfig.projbuild`、`idf_component.yml`、`CMakeLists.txt`、`docs/en/flash_model/README.rst`。所有符号、依赖、分区规则均来自仓库。

## 组件依赖（idf_component.yml）

| 依赖 | 版本 |
|---|---|
| `idf` | `>= 5.0` |
| `espressif/esp-dsp` | `1.8.0` |
| `espressif/dl_fft` | `>= 0.2.0` |
| `espressif/cjson` | `^1.7.19` |
| 组件版本（自身） | `2.4.6` |

## 支持芯片（CMakeLists.txt + README）

`esp32`、`esp32s2`、`esp32s3`、`esp32s31`、`esp32p4`、`esp32c3`、`esp32c5`、`esp32c6`

> ESP32-P4 在 IDF v5.5 之前用 `esp32p4_less_v3` 库（rev < v3）；IDF v6 不再支持 P4 rev<3。

## menuconfig 顶层（ESP Speech Recognition）

### 模型数据路径

| Kconfig | 说明 |
|---|---|
| `CONFIG_MODEL_IN_FLASH` | 模型存 flash 的 `model` 分区（默认，ESP-IDF 自动烧录） |
| `CONFIG_MODEL_IN_SDCARD` | 模型存 SD 卡 |
| `CONFIG_ESP_SR_EXTERNAL_MODEL_PATH` | 外部模型路径（自定义加载） |

### 噪声抑制 NS

| Kconfig | 说明 |
|---|---|
| `CONFIG_SR_NSN_WEBRTC` | 用 WebRTC NS（传统） |
| `CONFIG_SR_NSN_NSNET` | 用 NSNet（深度降噪） |

对应 `afe_config_t.ns_model_name` / `afe_ns_mode`：`AFE_NS_MODE_WEBRTC` / `AFE_NS_MODE_NET`。

### VAD

| Kconfig | 说明 |
|---|---|
| `CONFIG_SR_VADN_WEBRTC` | WebRTC VAD |
| `CONFIG_SR_VADN_VADNET` | VADNet（神经网络，`vadnet1 medium`） |

对应 `afe_config_t.vad_model_name`；NULL 时用 menuconfig 选的。

### WakeNet（大量 `CONFIG_SR_WN_WN*`，每个对应一个唤醒词模型）

menuconfig 在 `Select WakeNet` 下列出，如：
- `wn9_hilexin`（Hi,乐鑫）、`wn9_hiesp`（Hi,ESP）
- `wn9_alexa`、`wn9_jarvis_tts`、`wn9_computer_tts`、`wn9_mycroft_tts` 等
- `wn9l_*`（高配长版，提升极快语速响应）
- `wn9s_hilexin`、`wn9s_hiesp`、`wn9s_nihaoxiaozhi` 等（C3/C5/C6 专用）

另有 `Load Multiple Wake Words (WakeNet9 / WakeNet9s)` 子菜单，可同时加载多个唤醒词（AFE 支持 `AFE_MAX_WAKEWORD_NUM=3` 个）。

### MultiNet

| Kconfig（中文） | 模型名 | 芯片 |
|---|---|---|
| `CONFIG_SR_MN_CN_MULTINET2_SINGLE_RECOGNITION` | `mn2_cn` | ESP32 |
| mn5q8_cn / mn6_cn / mn7_cn | 同名 | S3 / P4 / S31 |

| Kconfig（英文） | 模型名 | 芯片 |
|---|---|---|
| mn5q8_en / mn6_en / mn7_en | 同名 | S3 / P4 / S31 |

menuconfig 子菜单：`Chinese Speech Commands Model` / `English Speech Commands Model`。

### 命令词（menuconfig 内逐条）

`Add Chinese speech commands` / `Add English speech commands`：可填命令 ID 与命令串，运行时用 `esp_mn_commands_update_from_sdkconfig()` 加载。命令 ID 从 1 起，同 ID 可多条。

## 分区表配置

ESP-SR 的 CMake 在 `partitions.csv` 含 `model` 分区时自动打包烧录模型（`model/movemodel.py`）。

### 必须项：model 分区

```csv
model,    data,         ,        , 6000K
```

- **Name** 固定 `model`
- **Type** = `data`
- **Size** 按所选模型总和（见 `docs/benchmark`），常见 4~8MB

未配置时编译报：`Failed to find model in partition table file`。

### 可选项：voice_data 分区（仅 TTS）

```csv
voice_data,  data, fat,         , 1500K
```

`esp_partition_find_first(ESP_PARTITION_TYPE_DATA, ESP_PARTITION_SUBTYPE_ANY, "voice_data")`。

## afe_config_t 关键字段默认值与取值

| 字段 | 取值 / 默认 | 说明 |
|---|---|---|
| `aec_init` | bool | 是否启用 AEC |
| `aec_mode` | `AEC_MODE_SR_LOW_COST`/`_HIGH_PERF` | AFE 内只接受 SR 两档；FD/VOIP 走独立 AEC 或 `AFE_TYPE_FD/VC` |
| `aec_filter_length` | 4（S3/P4）/ 2（C5） | 越大越耗资源 |
| `aec_nlp_level` | `AEC_NLP_LEVEL_AGGR` | NLP 等级，仅 FD 生效 |
| `se_init` | bool | 麦克风阵列处理（BSS/MISO） |
| `ns_init` / `ns_model_name` / `afe_ns_mode` | bool / 名 / `AFE_NS_MODE_*` | 降噪 |
| `vad_init` | true | 是否启用 VAD |
| `vad_mode` | `VAD_MODE_0`~`VAD_MODE_4` | 越大越严格 |
| `vad_min_speech_ms` | 128（>32） | 触发语音最短时长 |
| `vad_min_noise_ms` | 1000（>64） | 触发静音最短时长 |
| `vad_delay_ms` | 128 | 首帧语音延迟；cache 覆盖不全时增大 |
| `vad_mute_playback` | false | VAD 检测时静音播放 |
| `vad_enable_channel_trigger` | false | VAD 选通道 |
| `wakenet_init` | true | 是否启用 WakeNet |
| `wakenet_model_name` / `_2` | 模型名 | 第二个可选唤醒词 |
| `wakenet_mode` | `DET_MODE_90` 等 | 唤醒检测模式 |
| `agc_init` | bool | 自动增益 |
| `agc_mode` | `AFE_AGC_MODE_WEBRTC`/`_WAKENET` | |
| `agc_compression_gain_db` | 9 | 压缩增益 dB |
| `agc_target_level_dbfs` | 3 | 目标电平 -dBFS |
| `pcm_config` | 由 input_format 解析 | 通道布局 |
| `afe_mode` | `AFE_MODE_LOW_COST`/`_HIGH_PERF` | 性能/成本权衡 |
| `afe_type` | `AFE_TYPE_SR`/`VC`/`VC_8K`/`FD` | 场景类型 |
| `afe_perferred_core` / `_priority` | int | AFE 内部 SE 任务核与优先级 |
| `afe_ringbuf_size` | int | ringbuf 帧数 |
| `memory_alloc_mode` | `AFE_MEMORY_ALLOC_*` | 内存分配倾向 |
| `afe_linear_gain` | [0.1, 10.0] | 输出线性增益 |
| `fixed_first_channel` | bool | 首次唤醒后固定通道为麦 |
| `fixed_output_channel` | bool | 输出固定为麦通道 |
| `output_playback_channel` | bool | fetch 输出播放参考通道 |

## 阈值范围速查

| 模型 | 设置接口 | 范围 |
|---|---|---|
| WakeNet（独立） | `wakenet->set_det_threshold(md, thr, word_idx)` | 0.4 ~ 0.9999 |
| WakeNet（AFE） | `afe_handle->set_wakenet_threshold(afe, index, thr)` | 0.4 ~ 0.9999（index=1/2） |
| MultiNet | `multinet->set_det_threshold(md, thr)` | 0.0 ~ 0.9999 |
| VADNet | `vadnet->set_det_threshold(md, thr)` | 0.5 ~ 0.9999 |

## 常量宏

```c
// esp_mn_iface.h
ESP_MN_RESULT_MAX_NUM  5
ESP_MN_MAX_PHRASE_NUM  400
ESP_MN_MAX_PHRASE_LEN  63
ESP_MN_MIN_PHRASE_LEN  2

// esp_afe_config.h
AFE_MAX_WAKEWORD_NUM   3

// model_path.h
SRMODEL_STRING_LENGTH  32
MODEL_NAME_MAX_LENGTH  64

// esp_vad.h
SAMPLE_RATE_HZ         16000
VAD_FRAME_LENGTH_MS    30
```

## 构建命令

| 命令 | 作用 |
|---|---|
| `idf.py set-target <chip>` | 设定目标芯片 |
| `idf.py menuconfig` | 选模型、配分区路径、加命令词 |
| `idf.py build` | 编译（自动跑 movemodel.py 打包 srmodels.bin） |
| `idf.py flash` | 烧录应用 + 模型（到 model 分区） |
| `idf.py app-flash` | 只烧应用，不重烧模型（调试加速） |
| `idf.py partition-table` | 查看分区表与 model 分区 offset |
| `python model/movemodel.py -d1 sdkconfig -d2 esp-sr -d3 build` | 手动打包（Arduino/手动场景） |

## 参考文档路径（仓库内）

- `docs/en/flash_model/README.rst` — 模型选择与加载
- `docs/en/benchmark/README.rst` — 各模型资源占用
- `docs/en/audio_front_end/README.rst` — AFE 字段说明
- `Kconfig.projbuild` — 全部 Kconfig 符号源
