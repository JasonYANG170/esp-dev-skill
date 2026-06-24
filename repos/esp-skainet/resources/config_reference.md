# 配置项速查（Kconfig / sdkconfig / partitions）

> 全部 Kconfig 符号取自 `espressif-repos/esp-sr/Kconfig.projbuild` 与 `espressif-repos/esp-skainet/components/hardware_driver/Kconfig.projbuild`。

## 1. 板子选择（hardware_driver / Audio Media HAL）

| Kconfig symbol | 板子 | 依赖 |
|---|---|---|
| `CONFIG_ESP32_KORVO_V1_1_BOARD` | ESP32-Korvo | `IDF_TARGET_ESP32` |
| `CONFIG_ESP32_S3_BOX_BOARD` | ESP32-S3-BOX | `IDF_TARGET_ESP32S3` |
| `CONFIG_ESP32_S3_BOX_3_BOARD` | ESP32-S3-BOX-3 | `IDF_TARGET_ESP32S3` |
| `CONFIG_ESP32_S3_KORVO_1_V4_0_BOARD` | ESP32-S3-Korvo-1（默认） | `IDF_TARGET_ESP32S3` |
| `CONFIG_ESP32_S3_KORVO_2_V3_0_BOARD` | ESP32-S3-Korvo-2 | `IDF_TARGET_ESP32S3` |
| `CONFIG_ESP32_S3_EYE_BOARD` | ESP32-S3-EYE | `IDF_TARGET_ESP32S3` |
| `CONFIG_ESP32_P4_FUNCTION_EV_BOARD` | ESP32-P4-Function-EV | `IDF_TARGET_ESP32P4` |

> 选择路径：`Audio Media HAL → Audio hardware board`。默认值 `ESP32_S3_KORVO_1_V4_0_BOARD`。

## 2. 模型数据路径

| symbol | 含义 | 依赖 |
|---|---|---|
| `CONFIG_MODEL_IN_FLASH` | 从 flash 读模型（默认） | S3 / P4 / S31 |
| `CONFIG_MODEL_IN_SDCARD` | 从 SD 卡读模型 | S3 / P4 / S31 |

## 3. 降噪 / VAD 模型选择

| symbol | 含义 |
|---|---|
| `CONFIG_SR_NSN_WEBRTC` | 噪声抑制 WebRTC（默认） |
| `CONFIG_SR_NSN_NSNET2` | 深度降噪 nsnet2（仅 S3/P4/S31） |
| `CONFIG_SR_VADN_WEBRTC` | VAD WebRTC（默认） |
| `CONFIG_SR_VADN_VADNET1_MEDIUM` | VAD 神经网络 vadnet1 medium（仅 S3/P4/S31） |

## 4. AFE 接口

| symbol | 含义 |
|---|---|
| `CONFIG_AFE_INTERFACE_V1` | afe interface v1（默认，当前唯一选项） |

## 5. WakeNet 模型（WakeNet9 系列，节选）

主选择 `ESP Speech Recognition → Select wake words`（单选），并可经 `Load Multiple Wake Words` 菜单多选。常用：

| symbol | 唤醒词 |
|---|---|
| `CONFIG_SR_WN_WN9_HILEXIN` | Hi,乐鑫 (wn9_hilexin) |
| `CONFIG_SR_WN_WN9_HIESP` | Hi,ESP (wn9_hiesp) |
| `CONFIG_SR_WN_WN9_ALEXA` | Alexa (wn9_alexa) |
| `CONFIG_SR_WN_WN9_XIAOAITONGXUE` | 小爱同学 (wn9_xiaoaitongxue) |
| `CONFIG_SR_WN_WN9_NIHAOXIAOZHI_TTS` | 你好小智 (wn9_nihaoxiaozhi_tts) |
| `CONFIG_SR_WN_WN9_JARVIS_TTS` | Jarvis (wn9_jarvis_tts) |
| `CONFIG_SR_WN_WN9_COMPUTER_TTS` | computer (wn9_computer_tts) |
| `CONFIG_SR_WN_WN9_HEYWILLOW_TTS` | Hey,Willow (wn9_heywillow_tts) |
| `CONFIG_SR_WN_WN9S_HILEXIN` | Hi,乐鑫 (wn9s_hilexin) — wn9s 系列 |
| `CONFIG_SR_WN_WN9S_HIESP` | Hi,ESP (wn9s_hiesp) |
| `CONFIG_SR_WN_WN9S_NIHAOXIAOZHI` | 你好小智 (wn9s_nihaoxiaozhi) |

> AFE 最多同时运行两个 WakeNet 模型。完整列表见 `esp-sr/Kconfig.projbuild`。

## 6. MultiNet 命令词模型

中文（`Chinese Speech Commands Model`）：

| symbol | 模型 | 依赖 |
|---|---|---|
| `CONFIG_SR_MN_CN_NONE` | 无（默认） | — |
| `CONFIG_SR_MN_CN_MULTINET2_SINGLE_RECOGNITION` | mn2_cn | ESP32 |
| `CONFIG_SR_MN_CN_MULTINET5_RECOGNITION_QUANT8` | mn5q8_cn | ESP32-S3 |
| `CONFIG_SR_MN_CN_MULTINET6_QUANT` | mn6_cn | ESP32-S3 |
| `CONFIG_SR_MN_CN_MULTINET6_AC_QUANT` | mn6_cn_ac（空调） | ESP32-S3 |
| `CONFIG_SR_MN_CN_MULTINET7_QUANT` | mn7_cn | S3 / P4 / S31 |
| `CONFIG_SR_MN_CN_MULTINET7_AC_QUANT` | mn7_cn_ac（空调） | S3 / P4 / S31 |

英文（`English Speech Commands Model`）：

| symbol | 模型 | 依赖 |
|---|---|---|
| `CONFIG_SR_MN_EN_NONE` | 无（默认） | — |
| `CONFIG_SR_MN_EN_MULTINET5_SINGLE_RECOGNITION_QUANT8` | mn5q8_en | ESP32-S3 |
| `CONFIG_SR_MN_EN_MULTINET6_QUANT` | mn6_en | ESP32-S3 |
| `CONFIG_SR_MN_EN_MULTINET7_QUANT` | mn7_en | S3 / P4 / S31 |

## 7. 中文默认命令词（仅 mn2 / mn5，menuconfig `Add Chinese speech commands`）

`CONFIG_CN_SPEECH_COMMAND_ID0 .. ID31`，类型 `string`。仅当选择了 mn2/mn5 时可见。默认值示例：

| symbol | 默认值 |
|---|---|
| `CN_SPEECH_COMMAND_ID0` | `da kai kong tiao` |
| `CN_SPEECH_COMMAND_ID1` | `guan bi kong tiao` |
| `CN_SPEECH_COMMAND_ID2` | `zeng da feng su` |
| `CN_SPEECH_COMMAND_ID3` | `jian xiao feng su` |
| `CN_SPEECH_COMMAND_ID4` | `sheng gao yi du` |
| `CN_SPEECH_COMMAND_ID5` | `jiang di yi du` |
| `CN_SPEECH_COMMAND_ID6` | `zhi re mo shi` |
| `CN_SPEECH_COMMAND_ID7` | `zhi leng mo shi` |
| `CN_SPEECH_COMMAND_ID8` | `song feng mo shi` |
| `CN_SPEECH_COMMAND_ID9` | `jie neng mo shi` |
| `CN_SPEECH_COMMAND_ID10` | `chu shi mo shi` |
| `CN_SPEECH_COMMAND_ID11` | `jian kang mo shi` |
| `CN_SPEECH_COMMAND_ID12` | `shui mian mo shi` |
| `CN_SPEECH_COMMAND_ID13` | `da kai lan ya` |
| `CN_SPEECH_COMMAND_ID14` | `guan bi lan ya` |
| `CN_SPEECH_COMMAND_ID15` | `kai shi bo fang` |
| `CN_SPEECH_COMMAND_ID16` | `zan ting bo fang` |
| `CN_SPEECH_COMMAND_ID17` | `ding shi yi xiao shi` |
| `CN_SPEECH_COMMAND_ID18` | `da kai dian deng` |
| `CN_SPEECH_COMMAND_ID19` | `guan bi dian deng` |

> mn6/mn7 不读这些 Kconfig 字符串，推荐用 `esp_mn_commands_add()` 运行时添加。

## 8. ESP32-S3 sdkconfig.defaults 推荐模板

来自 `examples/cn_speech_commands_recognition/sdkconfig.defaults.esp32s3`：

```ini
CONFIG_IDF_TARGET="esp32s3"
CONFIG_ESPTOOLPY_FLASHMODE_QIO=y
CONFIG_ESPTOOLPY_FLASHSIZE_16MB=y
CONFIG_PARTITION_TABLE_CUSTOM=y
CONFIG_PARTITION_TABLE_CUSTOM_FILENAME="partitions.csv"
CONFIG_PARTITION_TABLE_FILENAME="partitions.csv"
CONFIG_PARTITION_TABLE_OFFSET=0x8000
CONFIG_SR_VADN_VADNET1_MEDIUM=y          # 或 SR_VADN_WEBRTC
CONFIG_SR_WN_WN9_HIESP=y                 # 唤醒词，按需
CONFIG_SR_MN_CN_MULTINET7_QUANT=y        # 中文命令，按需
CONFIG_SPIRAM=y
CONFIG_SPIRAM_MODE_OCT=y
CONFIG_SPIRAM_SPEED_80M=y
CONFIG_ESP_DEFAULT_CPU_FREQ_MHZ_240=y
CONFIG_ESP32S3_INSTRUCTION_CACHE_32KB=y
CONFIG_ESP32S3_DATA_CACHE_64KB=y
CONFIG_ESP32S3_DATA_CACHE_LINE_64B=y
```

## 9. partitions.csv 标准模板

```
# Name,  Type, SubType, Offset,  Size
factory, app,  factory, 0x010000, 2048k
model,  data, spiffs,         , 5168K
```

- `factory` —— 应用固件（≥ 2 MB）
- `model` —— SPIFFS 模型分区，label 必须与 `esp_srmodel_init("model")` 一致
- TTS 另需 `voice_data` 分区（见 `examples/chinese_tts/partitions.csv`）

## 10. idf_component.yml（main/ 下）

```yaml
dependencies:
  espressif/esp-sr: "^2.0.0"
  espressif/led_strip: "^2.5.0"   # 仅 cn_speech_commands_recognition 需要（LED 灯效）
```
