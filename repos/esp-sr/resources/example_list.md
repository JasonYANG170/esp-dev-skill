# ESP-SR 示例索引（仓库自带 test_apps）

> 仓库本身是 **component**，产品级完整示例在 esp-skainet 仓库 `examples/`。下表是 **esp-sr 仓库内** 可编译的 `test_apps/`，每条都是 ESP-IDF 工程，包含真实可运行的 API 用法，是本 skill recipe 的代码来源。

## test_apps/esp-sr（主测试工程，ESP32-S3 等）

| 路径 | 说明 |
|---|---|
| `test_apps/esp-sr/main/app_main.cpp` | 测试入口，注册各 unity test case |
| `test_apps/esp-sr/main/test_afe.cpp` | AFE create/destroy 内存泄漏测试、`AFE default setting`（遍历 SR/FD/VC × LOW/HIGH × MR/MMNR）、单/双麦性能测试（`test_feed_Task`/`test_fetch_Task`/`afe_task_into_t`、`AFE_TYPE_VC`）、`test afe aec interface`（对比 `aec_process` 与 `afe_aec_process`）、`test doa interface`（`esp_doa_create(16000,20,0.06,1024)` → `esp_doa_process` → 打印角度 + 5 次 create/destroy 泄漏检查） |
| `test_apps/esp-sr/main/test_wakenet.cpp` | WakeNet create/destroy/detect/cpu loading，含 `DET_MODE_90/95/2CH/3CH`、`set_det_threshold`、`reset_det_threshold` |
| `test_apps/esp-sr/main/test_multinet.cpp` | MultiNet create/detect/get_results；命令词 add/remove/modify/clear/duplicated/incorrect；`esp_mn_commands_update_from_sdkconfig` 与 API 两种加载方式 |
| `test_apps/esp-sr/main/test_vadnet.cpp` | VADNet create/detect/cpu loading（`vadnet->create(name, VAD_MODE_0, 1, 32, 64)`） |
| `test_apps/esp-sr/main/test_mfcc.cpp` | MFCC 特征提取测试 |
| `test_apps/esp-sr/main/CMakeLists.txt` | `REQUIRES unity esp-sr esp_timer`，`WHOLE_ARCHIVE` |

## test_apps/esp-tts（中文 TTS 测试）

| 路径 | 说明 |
|---|---|
| `test_apps/esp-tts/main/test_chinese_tts.cpp` | TTS create/destroy 内存泄漏；`esp_partition_mmap` voice_data、`esp_tts_voice_set_init`、`esp_tts_create`（IDF v4/v5 mmap 兼容） |
| `test_apps/esp-tts/main/test_chinese_tts.c` | TTS 的 C 版本 |
| `test_apps/esp-tts/main/app_main.cpp` | 入口 |

## test_apps/esp32c5（C5 专用，无 PSRAM 场景）

| 路径 | 说明 |
|---|---|
| `test_apps/esp32c5/main/test_wakenet.cpp` | C5 上 WakeNet（wn9s）create/detect |
| `test_apps/esp32c5/main/test_aec.cpp` | C5 上 AEC（filter_length 推荐 2） |

## 关键源文件（component 自身，非示例但常需查阅）

| 路径 | 说明 |
|---|---|
| `src/model_path.c` | `esp_srmodel_init/filter/exists/get_wake_words` 实现 |
| `src/esp_mn_speech_commands.c` | 命令词链表 add/remove/modify/clear/update 实现 |
| `src/esp_process_sdkconfig.c` | `esp_mn_commands_update_from_sdkconfig`、`check_chip_config` |
| `src/esp_sr_debug.c` | 调试辅助 |
| `CMakeLists.txt` | 组件构建脚本：按 target 选 `lib/<target>/`、链接预编译库、自动跑 `movemodel.py` |
| `Kconfig.projbuild` | 全部 menuconfig 选项（模型选择、数据路径、命令词） |

## 命令词 / 模型数据文件

| 路径 | 说明 |
|---|---|
| `model/multinet_model/fst/commands_cn.txt` | 中文默认命令词表（拼音，313 条） |
| `model/multinet_model/fst/commands_en.txt` | 英文默认命令词表（grapheme + phoneme，44 条） |
| `model/wakenet_model/` | WakeNet 模型权重 |
| `model/multinet_model/` | MultiNet 模型权重 |
| `model/vadnet_model/` | VADNet 模型权重 |
| `model/nsnet_model/` | NSNet 降噪模型权重 |
| `model/movemodel.py` | 打包 `srmodels.bin`（CMake 自动调用） |
| `model/pack_model.py` | 模型打包辅助 |

## 工具

| 路径 | 说明 |
|---|---|
| `tool/multinet_g2p.py` | 英文命令词 Grapheme→Phoneme 转换（mn5/mn7） |
| `tool/multinet_pinyin.py` | 中文命令词拼音辅助 |
| `tool/fst/` | FST 相关数据 |

## 文档（仓库内）

| 路径 | 说明 |
|---|---|
| `docs/en/index.rst` | 文档总索引 |
| `docs/en/getting_started/readme.rst` | 入门 |
| `docs/en/audio_front_end/README.rst` | AFE 使用 |
| `docs/en/audio_front_end/migration_guide.rst` | V1→V2 迁移 |
| `docs/en/audio_front_end/Espressif_Microphone_Design_Guidelines.rst` | 麦克风设计指南 |
| `docs/en/wake_word_engine/README.rst` | WakeNet |
| `docs/en/wake_word_engine/ESP_Wake_Words_Customization.rst` | 唤醒词定制流程 |
| `docs/en/vadnet/README.rst` | VADNet（含 vad_cache） |
| `docs/en/acoustic_echo_cancellation/README.rst` | AEC（含资源占用表） |
| `docs/en/speech_command_recognition/README.rst` | MultiNet 命令词 |
| `docs/en/speech_synthesis/readme.rst` | 中文 TTS |
| `docs/en/flash_model/README.rst` | 模型选择与烧录 |
| `docs/en/benchmark/README.rst` | 资源占用基准 |
| `docs/en/test_report/README.rst` | 测试方法 |
| `docs/en/glossary/glossary.rst` | 术语表 |

中文版在 `docs/zh_CN/` 对应路径。

## 仓库外的产品级示例（esp-skainet，仅供参考不在本仓库）

esp-skainet `examples/` 提供含 I2S 驱动、LED、OTA 的完整产品示例，常见包括：
- `wake_word_detection` — 唤醒词检测
- `cn_speech_commands_recognition` / `en_speech_commands_recognition` — 中/英文命令词
- `chinese_tts` — 中文 TTS
- 各类 Korvo 开发板配套示例

> 本 skill 不引用其代码（不在 esp-sr 仓库内），仅作指引。
