# Changelog

本文件记录 esp-skainet-skill 的发布历史。格式参考 [Keep a Changelog](https://keepachangelog.com/)。

## [1.1.0] - 2026-06-18

补充产品发布前测试验收与硬件移植两个确认缺失的场景。

### 新增

- **recipes/perf_benchmarking.md** — 性能基准测试与测试报告生成。覆盖三种入口：`perf_tester` 控制台（`config <fast|norm> <all|none|pink|pub> <all|none|0|5|10>` / `start`）、串口命令 `rar`/`far`（离线跑 SD 卡 CSV 测试集）、pytest CI 自动化。含 RAR/FAR 测试集 CSV 格式、`offline_wn_tester_start` API、串口报告格式（`print_wn_report`）、FAR→RAR 阈值闭环、`report.json` 生成与内存断言。
  - 来源：`test/wakenet/main/wakenet_main.c`、`components/perf_tester/wn_perf_tester.{c,h}`、`components/perf_tester/perf_tester_cmd.{c,h}`、`test/wakenet/pytest_wakenet.py`、`test/wakenet/README.md`、`test/README.md`、`test/record_test_set/README.md`、`test/multinet/main/multinet_main.c`。
- **recipes/custom_board_porting.md** — 自定义音频板移植。讲解 `bsp_board.h` 的 `CONFIG_ESP_CUSTOM_BOARD` → `esp_custom_board.h` 分支、5 个必须实现的 `bsp_*` 函数、引脚表头文件结构（I2C/I2S/SDMMC/电源）、ES7210+ES8311 codec 初始化模板、`bsp_get_feed_data` 通道重排与 `input_format` 不变量（`"RMNM"`/`"MMR"`/`"MR"`/`"MN"`）、Kconfig 注册两种方式。
  - 来源：`components/hardware_driver/boards/esp32s3-korvo-1/bsp_board.c`、`boards/esp32s3-box|esp32p4-function-ev|esp32s3-eye/bsp_board.c`（input_format 对照）、`boards/include/bsp_board.h`、`boards/include/esp32_s3_korvo_1_v4_board.h`、`include/esp_board_init.h`、`esp_board_init.c`、`Kconfig.projbuild`。

### 变更

- **SKILL.md**：`metadata.version` 1.0.0 → 1.1.0；Scenario Quick Reference 新增「测试与移植」分组，索引 2 个新 recipe。
- **resources/example_list.md**：扩充 `test/` 节（新增 `test/multinet/`、`test/vad/`，细化 `test/wakenet/` 与 `test/record_test_set/` 说明）；`components/` 节新增 4 个板级 `bsp_board.c` 路径、`bsp_board.h`、`esp32_s3_korvo_1_v4_board.h`，细化 `perf_tester/` 说明。
- **resources/api_reference.md**：第 9 节扩充 perf_tester C API（`perf_tester_config_t`、`offline_wn_tester_start/_stop`、`tester_audio_t` 枚举、`check_noise`/`check_snr`、`register_perf_tester_*`）、报告格式说明；新增 9.2 自定义板移植 API（`bsp_*` 函数签名、选板 Kconfig、移植不变量）。

## [1.0.0] - 2026-06-18

首个正式发布。面向 ESP-Skainet（乐鑫智能语音助手）的 AI 编码技能包。

### 新增

- **SKILL.md**：12 条核心原则、When to Use、10 个 recipe 索引、板子支持表、`afe_config_t` 关键字段表、唤醒/命令状态机、12 条 Critical Pitfalls（每条含 WRONG/CORRECT 代码块）、Execution Workflow、Failure Strategies。
- **AGENTS.md**：项目上下文、文件命名、include 模式、标准工程结构、partitions.csv 模板、`app_main` 与 feed/detect 任务模板、构建流程、codegen 清单、Do Not Modify 说明。
- **recipes/**（10 个）：
  - `wake_word_afe.md` — 基于 AFE 的实时唤醒词检测（双模型 + 阈值调节）
  - `wake_word_raw.md` — 离线 PCM/WAV 唤醒推理（不走 AFE）
  - `cn_speech_commands.md` — 中文命令词识别（MultiNet7 + sdkconfig 导入）
  - `en_speech_commands.md` — 英文命令词识别（仅 ESP32-S3）
  - `customize_commands.md` — 运行时自定义命令词（add/update/modify/remove）
  - `afe_config_tuning.md` — AFE 配置项调优（AEC/SE/NS/VAD/AGC/type/mode/内存）
  - `deep_noise_suppression.md` — 深度降噪（AFE_TYPE_VC + NSNET2 + PCM 落盘）
  - `voice_communication.md` — 语音通信数据增强
  - `voice_activity_detection.md` — VAD 检测（vad_cache 防截断）
  - `direction_of_arrival.md` — 双麦方向角估计（关 AEC + 拆声道）
  - `chinese_tts.md` — 中文 TTS 合成（voice_data 分区 mmap）
- **resources/**：
  - `api_reference.md` — 取自 `esp-sr/include/esp32s3/` 的真实函数/结构/枚举/宏速查（模型加载、WakeNet、MultiNet、AFE、板级、DOA、TTS、player、perf_tester）
  - `config_reference.md` — 板子/模型/降噪/VAD 全部 Kconfig 符号 + sdkconfig.defaults 模板 + partitions.csv 模板
  - `pitfalls.md` — 26 条构建期/运行期陷阱汇总
  - `example_list.md` — `examples/` 真实工程与 `components/`/`tools/`/`test/` 索引
- **README.md**：中文介绍、特性、安装、使用、支持范围、目录结构。
- **CHANGELOG.md**：本文件。

### 依据来源

- API/结构/枚举：`espressif-repos/esp-sr/include/esp32s3/*.h`、`esp-sr/src/include/model_path.h`、`esp-sr/src/include/esp_mn_speech_commands.h`、`esp-sr/esp-tts/esp_tts_chinese/include/*.h`
- Kconfig：`espressif-repos/esp-sr/Kconfig.projbuild`、`esp-skainet/components/hardware_driver/Kconfig.projbuild`
- 代码示例：`esp-skainet/examples/*/main/main.c` 及 README
- 板级 API：`esp-skainet/components/hardware_driver/include/esp_board_init.h`
