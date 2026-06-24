# 示例工程索引（example_list.md）

> 全部路径为 `esp-skainet` 仓库 `examples/` 下的真实工程。描述依据各 example 的 README 与 `main/main.c`。

## 主示例

| 路径 | 说明 | 最新模型 | 支持板 |
|---|---|---|---|
| `examples/cn_speech_commands_recognition/` | 中文命令词识别：唤醒 → MultiNet 命令模式，命中播放提示音、控 LED | MultiNet7 | ESP32-Korvo, ESP32-S3-Korvo-1/2, ESP-BOX, ESP32-S3-EYE, ESP32-P4-Function-EV |
| `examples/en_speech_commands_recognition/` | 英文命令词识别（仅 ESP32-S3 系列），结构与中文版一致 | MultiNet7 | ESP32-S3-Korvo-1/2, ESP-BOX, ESP32-S3-EYE, ESP32-P4-Function-EV |
| `examples/wake_word_detection/` | 唤醒词检测（含两个子示例） | WakeNet9 | ESP32-Korvo, ESP32-S3-Korvo-1/2, ESP-BOX, ESP32-S3-EYE, ESP32-P4-Function-EV |
| `examples/chinese_tts/` | 中文 TTS 语音合成，从 UART 输入文本实时合成播放 | esp-tts-v1.7 | ESP32-Korvo, ESP32-S3-Korvo-1/2, ESP-BOX, ESP32-P4-Function-EV |
| `examples/usb_mic_recorder/` | USB 麦克风录制（README 提及） | — | ESP-BOX, ESP32-S3-Korvo-2 |

## wake_word_detection 子示例

| 路径 | 说明 |
|---|---|
| `examples/wake_word_detection/afe/` | **基于 AFE 的实时唤醒**：feed/detect 双任务，支持双模型加载与 `set_wakenet_threshold` |
| `examples/wake_word_detection/wakenet/` | **离线 PCM 推理**：不走 AFE，对内嵌 `hilexin`/`hiesp` PCM 数组直接 `wakenet->detect` |

## AFE / 通信 / 降噪示例

| 路径 | 说明 |
|---|---|
| `examples/deep_noise_suppression/` | 深度降噪：`AFE_TYPE_VC` + NSNET2，feed/fetch PCM 通过 ringbuf 落盘 SD 卡对比 |
| `examples/voice_communication/` | 语音通信：`AFE_TYPE_VC`，从 fetch 取增强后单声道 PCM 上行 |
| `examples/voice_activity_detection/` | VAD 语音活动检测：调 `vad_mode`/`vad_min_speech_ms`，用 `vad_cache` 避免首字截断，语音帧落盘 |
| `examples/direction_of_arrival/` | 双麦方向角估计：`esp_doa_create/process`，关 AEC、按 input_format `M` 位拆左右声道 |

## 各示例的关键文件

每个示例标准结构：
```
examples/<name>/
├── CMakeLists.txt
├── partitions.csv                  # factory(app) + model(spiffs)
├── partitions_esp32.csv            # （部分）ESP32 专用分区
├── sdkconfig.defaults              # 公共（分区表）
├── sdkconfig.defaults.esp32        # ESP32 目标
├── sdkconfig.defaults.esp32s3      # ESP32-S3 目标（PSRAM + 模型选择）
├── sdkconfig.defaults.esp32p4      # ESP32-P4 目标
├── README.md / README_cn.md
└── main/
    ├── CMakeLists.txt
    ├── idf_component.yml           # dependencies: espressif/esp-sr ^2.0.0
    ├── main.c                      # app_main
    ├── speech_commands_action.c    # （命令词示例）命中回调
    └── include/
        └── speech_commands_action.h
```

## 组件（components/）

| 路径 | 说明 |
|---|---|
| `components/hardware_driver/` | 板级 I2S/SD/播放抽象，含 `boards/<board>/bsp_board.c` 与 Kconfig 选板 |
| `components/hardware_driver/boards/esp32s3-korvo-1/bsp_board.c` | ES7210+ES8311 双 I2S 完整实现，自定义板移植参考模板（含 `bsp_get_feed_data` 通道重排） |
| `components/hardware_driver/boards/esp32s3-box/bsp_board.c` | ESP32-S3-BOX 双麦实现 |
| `components/hardware_driver/boards/esp32p4-function-ev/bsp_board.c` | ESP32-P4 板（`input_format="MR"`） |
| `components/hardware_driver/boards/esp32s3-eye/bsp_board.c` | 单麦无回采（`input_format="MN"`） |
| `components/hardware_driver/boards/include/bsp_board.h` | 板级 API 声明 + `CONFIG_ESP_CUSTOM_BOARD` → `esp_custom_board.h` 分支 |
| `components/hardware_driver/boards/include/esp32_s3_korvo_1_v4_board.h` | 引脚表 + `I2S_CONFIG_DEFAULT` 宏模板（移植复制起点） |
| `components/perf_tester/` | 性能测试控制台（wakenet/multinet），串口命令 `config`/`start`；`wn_perf_tester.c` 提供 `offline_wn_tester_start`/`print_wn_report`/CSV 解析 |
| `components/player/` | `esp_skainet_player_*` 播放器（ringbuf + 核心） |
| `components/sr_ringbuf/` | 环形缓冲（PCM 落盘调试用） |

## 工具（tools/）

| 路径 | 说明 |
|---|---|
| `tools/default_firmware/` | 预编译默认固件 |
| `tools/generate_audio_file/` | 生成测试音频文件工具 |

## 测试（test/）

| 路径 | 说明 |
|---|---|
| `test/wakenet/` | WakeNet 自动化测试：`main/wakenet_main.c` 注册 `rar`/`far` 串口命令，跑 RAR/FAR 测试集生成报告；`pytest_wakenet.py` 解析串口日志、断言唤醒率与内存上限、生成 `report.json`；`sdkconfig.ci.hilexin`/`sdkconfig.ci.hiesp` 用于 CI 多模型构建 |
| `test/multinet/` | MultiNet 自动化测试：`main/multinet_main.c` 用 `offline_mn_tester` + `register_perf_tester_start_cmd`；`pytest_multinet6.py`/`pytest_multinet7.py` 对应 mn6/mn7 |
| `test/vad/` | VAD 测试：`main/main.c` 验证 `vad_mode`/`vad_min_speech_ms` |
| `test/record_test_set/` | 录制测试集：`create_test_set.py`（WakeNet）/ `create_mn_test_set.py`（MultiNet）合成带 SNR/噪声的音频并通过喇叭播放 + DUT 录制到 SD 卡；`sdcard_recorder/` 录音固件；`config.yml`/`config_mn.yml` 配置源音频与 SNR 组合 |
