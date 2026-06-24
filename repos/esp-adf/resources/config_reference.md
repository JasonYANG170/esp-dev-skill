# ESP-ADF Configuration Reference

> 配置项均来自仓库真实 Kconfig（`components/audio_board/Kconfig.projbuild` 等）与示例 `sdkconfig.defaults*`。通过 `idf.py menuconfig` 修改，或写入 `sdkconfig.defaults`。

## 板级选择（Audio Board）

来源：`components/audio_board/Kconfig.projbuild`，prompt 为 “Audio board”。

| Kconfig 符号 | 说明 |
|---|---|
| `CONFIG_AUDIO_BOARD_CUSTOM` | 自定义板（需自带 board 实现） |
| `CONFIG_ESP_LYRAT_V4_3_BOARD` | ESP32-LyraT V4.3 |
| `CONFIG_ESP_LYRAT_V4_2_BOARD` | ESP32-LyraT V4.2 |
| `CONFIG_ESP_LYRATD_MSC_V2_1_BOARD` | ESP32-LyraTD-MSC V2.1 |
| `CONFIG_ESP_LYRATD_MSC_V2_2_BOARD` | ESP32-LyraTD-MSC V2.2 |
| `CONFIG_ESP_LYRAT_MINI_V1_1_BOARD` | ESP32-LyraT-Mini V1.1 |
| `CONFIG_ESP32_KORVO_DU1906_BOARD` | ESP32-Korvo-DU1906 |
| `CONFIG_ESP32_S2_KALUGA_1_V1_2_BOARD` | ESP32-S2-Kaluga-1 v1.2 |
| `CONFIG_ESP32_S3_KORVO2_V3_BOARD` | ESP32-S3-Korvo-2 v3 |
| `CONFIG_ESP32_S3_KORVO2L_V1_BOARD` | ESP32-S3-Korvo-2L v1 |
| `CONFIG_ESP32_S3_BOX_LITE_BOARD` | ESP32-S3-BOX-Lite |
| `CONFIG_ESP32_S3_BOX_BOARD` | ESP32-S3-BOX |
| `CONFIG_ESP32_S3_BOX_3_BOARD` | ESP32-S3-BOX-3 |
| `CONFIG_M5STACK_ATOMS3R_BOARD` | M5STACK-ATOMS3R |
| `CONFIG_ESP32_C3_LYRA_V2_BOARD` | ESP32-C3-Lyra v2.0 |
| `CONFIG_ESP32_C6_DEVKIT_BOARD` | ESP32-C6-DEVKIT |
| `CONFIG_ESP32_P4_FUNCTION_EV_BOARD` | ESP32-P4-FUNCTION-EV-BOARD |
| `CONFIG_ESP32_P4_FUNCTION_EV_SUB_BOARD` | ESP32-P4-FUNCTION-EV-SUB-BOARD |

### Korvo-DU1906 子芯片选择（仅该板）

| 符号 | 说明 |
|---|---|
| `CONFIG_ESP32_KORVO_DU1906_DAC_TAS5805M` | DAC 用 TAS5805M |
| `CONFIG_ESP32_KORVO_DU1906_DAC_ES7148` | DAC 用 ES7148 |
| `CONFIG_ESP32_KORVO_DU1906_ADC_ES7243` | ADC 用 ES7243 |

## 示例级 Kconfig（Kconfig.projbuild，按示例）

许多示例在自己的 `main/Kconfig.projbuild` 暴露选项，例如：

| 示例 | 典型符号 |
|---|---|
| `recorder/pipeline_wav_amr_sdcard` | `CONFIG_CHOICE_AMR_WB` / `CONFIG_CHOICE_AMR_NB`（选 AMR 宽/窄带，决定采样率 16000/8000） |
| `player/pipeline_http_mp3` 等 | `CONFIG_WIFI_SSID` / `CONFIG_WIFI_PASSWORD`（Wi-Fi 凭据） |
| `cli` | CLI 相关命令开关（见 `examples/cli/main/Kconfig.projbuild`） |

> 这些是**示例自带**的 menuconfig 项，不是 ADF 框架级符号；移植到新工程需自行复制对应 `Kconfig.projbuild`。

## sdkconfig.defaults 片段（按目标芯片）

ESP-ADF 示例按芯片提供 `sdkconfig.defaults.<target>`（如 `examples/cli/sdkconfig.defaults.esp32s3`、`sdkconfig.defaults.esp32p4.idf-v5-4`），常见内容：

```ini
# 选板
CONFIG_ESP32_S3_KORVO2_V3_BOARD=y

# PSRAM（多数音频应用需要）
CONFIG_SPIRAM=y
CONFIG_SPIRAM_BOOT_INIT=y

# 分区表
CONFIG_PARTITION_TABLE_CUSTOM=y
CONFIG_PARTITION_TABLE_CUSTOM_FILENAME="partitions.csv"

# Wi-Fi（网络示例）
CONFIG_ESP_MAIN_TASK_STACK_SIZE=4096
```

> 具体默认值以各示例的 `sdkconfig.defaults*` 为准；不要照抄到不兼容的芯片。

## IDF 版本兼容矩阵（来自 README.md）

| | IDF v5.1 | v5.2 | v5.3 | v5.4 | v5.5 | master |
|---|---|---|---|---|---|---|
| ESP-ADF release/v2.8 | 支持 | 支持 | 支持 | 支持 | 支持 | 不支持 |
| ESP-ADF release/v2.7 | 支持 | 支持 | 支持 | 不支持 | 不支持 | 不支持 |

> ESP-ADF `master` 仅与 IDF release/v5.1–v5.5 兼容；IDF `master` 不保证兼容。

## Python / 工具链

- Python 版本需 3.7 ~ 3.11（来自 get-started 文档）
- 路径不支持空格（ADF_PATH / IDF_PATH / 工程目录）

## 关键常量（来自头文件，非 Kconfig）

| 宏 | 值 | 来源 |
|---|---|---|
| `DEFAULT_PIPELINE_RINGBUF_SIZE` | 8*1024 | audio_pipeline.h |
| `DEFAULT_ELEMENT_RINGBUF_SIZE` | 8*1024 | audio_element.h |
| `DEFAULT_ELEMENT_BUFFER_LENGTH` | 4*1024 | audio_element.h |
| `DEFAULT_ELEMENT_STACK_SIZE` | 2*1024 | audio_element.h |
| `DEFAULT_ELEMENT_TASK_PRIO` | 5 | audio_element.h |
| `DEFAULT_ELEMENT_TASK_CORE` | 0 | audio_element.h |
| `DEFAULT_AUDIO_EVENT_IFACE_SIZE` | 5 | audio_event_iface.h |
| `I2S_STREAM_TASK_STACK` | 3584 | i2s_stream.h |
| `I2S_STREAM_BUF_SIZE` | 3600 | i2s_stream.h（24 位时 buffer_len 须为 3 的倍数） |
| `I2S_STREAM_TASK_PRIO` | 23 | i2s_stream.h |
| `I2S_STREAM_RINGBUFFER_SIZE` | 8*1024 | i2s_stream.h |
