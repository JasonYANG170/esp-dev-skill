# Changelog

本技能的版本演进记录。格式参考 [Keep a Changelog](https://keepachangelog.com/)。

## [1.1.0] - 2026-06-18

补齐 5 个被审计确认的高价值场景缺口：高级音视频采集、蓝牙音频、多流混音渲染、视频合成显示、AI 语音前端。所有内容均基于真实仓库 docs/、头文件与示例 ground 而成。

### 新增

- **recipes/（5 个新配方）**：
  - `esp_capture.md`：用 `esp_capture` 按 source/path/sink 模型做音视频采集（AEC 源、多 sink 并行流式+MP4、overlay、单帧抓拍、自定义处理链）
  - `esp_bt_audio.md`：经典蓝牙（A2DP Sink/Source、HFP HF/AG、AVRCP、PBAP）与 LE Audio（TMAP 单播/广播、VCP/CSIP）经 `esp_gmf_io_bt` 接入 pipeline
  - `esp_audio_render.md`：多路 PCM 混音渲染（per-stream 与 post-mix 处理链、solo、fade、运行时切输出格式）
  - `gmf_ai_audio.md`：AI 语音前端六元素（ai_afe 全功能 / ai_aec / ai_wn / ai_ns / ai_vad / ai_doa）+ afe_manager + wakeup/VAD 状态机 + 命令词 + 手动唤醒
  - `esp_video_render.md`：视频合成与显示（H264/MJPEG 上屏、overlay→container→widget UI 叠加、LCD/LVGL 后端、双目 dual_stream）
- **SKILL.md**：新增「音频渲染与混音」「视频采集与显示」两个 Scenario 分组；扩展「When to Use」适用场景；version 1.0.0 → 1.1.0。
- **resources/api_reference.md**：补 esp_capture/esp_bt_audio/esp_audio_render/esp_video_render/gmf_ai_audio 真实 API 签名。
- **resources/example_list.md**：补 5 类组件的真实示例路径。

### Grounding

- 配方代码改编自真实示例：
  - `packages/esp_capture/examples/audio_capture`、`video_capture`
  - `packages/esp_bt_audio/examples/bt_audio`（`main.c`、`stream_proc.c`）
  - `packages/esp_audio_render/examples/audio_render`、`simple_piano`
  - `elements/gmf_ai_audio/examples/wwe`、`aec_rec`
  - `packages/esp_video_render/examples/video_render`、`dual_eyes`、`video_player`
- 所有结构体/枚举/宏（`esp_capture_sink_cfg_t`、`esp_bt_audio_config_t`、`esp_audio_render_cfg_t`、`esp_gmf_afe_cfg_t`、`esp_video_render_stream_info_t` 等）取自真实头文件。

### 已知局限

- 蓝牙 LE Audio 的 TMAP 角色常量（`ESP_BLE_AUDIO_TMAP_ROLE_*`）与 PACS context 掩码（`ESP_BLE_AUDIO_CONTEXT_TYPE_*`）来自 ESP-IDF BLE Audio 协议栈，本技能仅引用、不展开其内部定义。
- `esp_video_render` 的 `esp_video_render_lcd_cfg_t` 字段随板级 `dev_display_lcd_config_t` 变化，本技能只给出 `out_format/width/height` 通用字段。
- `gmf_ai_audio` 的 `afe_config_t`、`AFE_TYPE_*`、`AFE_MODE_*`、`DET_MODE_*` 来自 `esp-sr`，本技能仅说明用法不重复其完整字段表。

## [1.0.0] - 2026-06-18

首个公开发布版本。内容完全基于 ESP-GMF 仓库（`gmf_core` / `elements` / `packages` / `gmf_examples`）的 `docs/`、头文件与示例构建。

### 新增

- **SKILL.md**：12 条核心原则、组件支持矩阵、pipeline 状态机与控制 API 有效状态表、12 条「错误 vs 正确」对照陷阱、执行工作流与失败策略。
- **AGENTS.md**：项目上下文、命名/include 约定、标准 `app_main` 模板（pool + pipeline + task）、事件回调模板、构建流程、代码生成检查清单。
- **recipes/（11 个配方）**：
  - `simple_player.md`：用 `esp_audio_simple_player` 快速 URI 播放
  - `esp_player.md`：用 `esp_player` 做音视频同步播放、seek、多轨道、HLS
  - `pipeline_play_embed.md`：GMF-Core 最小播放流水线（embed_flash → 效果链 → codec_dev）
  - `pipeline_play_http.md`：HTTP/HTTPS 播放（TLS 栈、异步 IO、控制超时）
  - `pipeline_record.md`：codec_dev 录音 → aud_enc → io_file
  - `pipeline_muxer.md`：aud_enc → aud_muxer 容器封装
  - `audio_effects.md`：运行时调 EQ/ALC/Sonic/Fade/DRC/MBC
  - `custom_element.md`：自定义 audio element 模板（open/process/close + 端口属性）
  - `runtime_methods.md`：AMETHOD + exe_method 解耦接口与实现
  - `loop_play.md`：无缝循环播放（strategy_func + RESET）
  - `state_error_recovery.md`：状态机、ERROR 恢复、stop 超时、pause/seek
- **resources/**：
  - `api_reference.md`：按模块分组的真实函数签名（pool/pipeline/task/element/port/payload/cache/io/elements/loader/各 package）
  - `config_reference.md`：task/IO/data bus/FourCC/URL 评分/menuconfig 真实配置
  - `pitfalls.md`：20 条高频陷阱汇总
  - `example_list.md`：仓库真实示例与测试路径表

### Grounding

- 所有 API、结构体、宏、配置项、文件路径、代码片段均来自 ESP-GMF 仓库的真实 `docs/` 与头文件。
- 配方代码改编自 `gmf_examples/basic_examples/*` 的真实示例（`play_embed_music.c`、`play_http_music.c`、`play_record_sdcard.c`、`play_music_without_gap.c`、`pipeline_audio_effects.c` 等）。
- 仓库未覆盖的 API 一律不写。

### 已知局限

- `aud_howl` 的 GMF 封装初始化 API（`esp_gmf_howl.h`）官方文档标注仍在开发中，本技能未给出具体初始化签名。
- 板级外设初始化经外部仓库 `esp_board_manager`，本技能不展开其内部 API，仅展示示例中的标准用法。
- `esp_capture` / `esp_video_render` / `gmf_ai_audio` 等组件的硬件 sink/后端配置（具体 LCD 型号、摄像头接线、麦克风阵列通道映射）依赖目标板，本技能给出 API 与示例入口，实际接线参对应示例 README。
