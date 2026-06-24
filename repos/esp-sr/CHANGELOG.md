# Changelog

本 skill 的版本演进记录。格式参考 [Keep a Changelog](https://keepachangelog.com/)，版本号遵循语义化版本。

## [1.1.0] - 2026-06-18

补充审计确认的两个 recipe 缺口：DOA 声源定位与 AFE 语音通信/全双工主线。

### Added
- **recipes/doa_sound_localization.md** — SRP-PHAT 双麦声源定位（0~180°）。覆盖独立 `esp_doa_create/process/destroy`（左右声道分开）与 AFE-aware `afe_doa_create/process/destroy`（按 `input_format` 自动从交错多通道数据解抽 left/right）；SRP-PHAT 四参数（`fs`/`resolution`/`d_mics`/`input_timedate_samples`）含义与推荐值、各 target 头文件分布、独立 vs AFE-aware 选型对照。
  - 来源示例：`test_apps/esp-sr/main/test_afe.cpp` 的 `TEST_CASE("test doa interface", "[afe]")`
  - 来源头文件：`include/esp32s3/esp_doa.h`、`include/esp32s3/esp_afe_doa.h`、`include/esp32/esp_doa.h`、`include/esp32p4/esp_afe_doa.h`
- **recipes/afe_vc_pipeline.md** — AFE 语音通信（VC）/ 全双工（FD）主线，与既有 SR 主线 recipe 对仗。覆盖 `AFE_TYPE_VC`/`AFE_TYPE_VC_8K`/`AFE_TYPE_FD` 选型、与 SR 不同的 pipeline 组合（VC: AEC(VOIP)→NS(nsnet2)→VAD；FD: AEC(FD)→[SE(BSS)]→VAD→WakeNet）、8kHz 输入约束、ESP32-S3/P4 资源占用表、fetch 出干净音频送网络/对话通路。
  - 来源示例：`test_apps/esp-sr/main/test_afe.cpp` 的 `AFE default setting` 与 `afe performance test (1ch/2ch)`
  - 来源文档：`docs/en/audio_front_end/README.rst`、`docs/en/benchmark/README.rst`

### Changed
- **SKILL.md**：算法模块 Scenario Quick Reference 表新增 `doa_sound_localization.md` 与 `afe_vc_pipeline.md` 两行；metadata.version `1.0.0` → `1.1.0`
- **resources/example_list.md**：扩充 `test_afe.cpp` 说明，显式列出 VC/FD 遍历（`AFE default setting`、`AFE_TYPE_VC` 性能测试）与 DOA 测试用例（参数与泄漏检查）
- **resources/api_reference.md**：第 8 节 DOA 扩充 `afe_doa_handle_t` 公开结构体与 `afe_doa_create/process/destroy` 签名，并标注仅 S3/S31/P4 提供 `esp_afe_doa.h`

### Grounding
- 所有函数签名、结构体、枚举值、推荐参数（16000/20.0/0.06/1024）、pipeline 字符串、资源占用数值均取自仓库 `include/<target>/esp_doa.h`、`include/<target>/esp_afe_doa.h`、`include/<target>/esp_afe_config.h`、`docs/en/audio_front_end/README.rst`、`docs/en/benchmark/README.rst`、`test_apps/esp-sr/main/test_afe.cpp`

## [1.0.0] - 2026-06-18

首个发布版本。基于 `esp-sr` 仓库（组件版本 `2.4.6`，ESP-SR V2.0 API）整理。

### Added
- **SKILL.md**：核心原则、何时使用、recipe 索引、芯片/模型支持矩阵、AFE/AEC/VAD 模式速查、12 条关键踩坑（含 WRONG/CORRECT 对照）、执行工作流、失败策略
- **AGENTS.md**：项目上下文、文件命名、include 模式（按 `include/<target>/` 自动选）、标准工程结构、AFE 启动样板、构建工作流、代码生成 checklist、工具与脚本表
- **recipes/**（9 个场景）：
  - `afe_sr_pipeline.md` — AFE feed/fetch 双任务 + MultiNet 唤醒后命令词主线
  - `model_partition.md` — menuconfig 选模型、partitions.csv、srmodels.bin 生成烧录
  - `migration_v1_v2.md` — `AFE_CONFIG_DEFAULT`/`ESP_AFE_SR_HANDLE` → `afe_config_init`/`esp_afe_handle_from_config`
  - `wakenet_standalone.md` — 独立 WakeNet create/detect/阈值/多唤醒
  - `multinet_commands.md` — MultiNet 加载、API/sdkconfig 增删改、单/连续模式
  - `custom_commands.md` — 中英文命令词定制、`commands_*.txt`、`multinet_g2p.py`
  - `vadnet.md` — VADNet 与 WebRTC VAD、vad_cache 防“吃字”
  - `aec_usage.md` — AEC 三种集成方式（aec_create / afe_aec_create / AFE）、SR/FD/VOIP mode 选型与资源占用
  - `chinese_tts.md` — 中文 TTS voice_data 分区、流式合成
- **resources/**（5 个速查）：
  - `api_reference.md` — AFE/WakeNet/MultiNet/VADNet/AEC/DOA/TTS 全部真实签名与结构体
  - `config_reference.md` — Kconfig 符号、分区配置、afe_config_t 字段、阈值范围、构建命令
  - `pitfalls.md` — 20 条踩坑汇总（WRONG/CORRECT 对照）
  - `example_list.md` — `test_apps/` 可编译工程索引 + 命令词/模型/工具/文档路径
  - `state_machine.md` — AFE pipeline 数据流、唤醒/命令词/VAD 状态机、index 约定
- **README.md** / **CHANGELOG.md**

### Grounding
- 所有 API、结构体、枚举、宏、模型名、配置项均取自仓库 `include/`、`src/include/`、`docs/`、`test_apps/`、`Kconfig.projbuild`、`idf_component.yml`、`CMakeLists.txt`、`README.md`
- 许可证按仓库 `LICENSE` 标注为 ESPRESSIF MIT
- 芯片支持矩阵按 README `Supported Targets` 标注
