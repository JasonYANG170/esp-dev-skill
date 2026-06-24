---
name: esp-gmf-skill
description: >-
  AI Skill for ESP-GMF (Espressif General Multimedia Framework) development on ESP32 chips.
  Used when users need to build, modify, or debug ESP-GMF audio/video/image streaming pipelines,
  including element/pipeline/pool construction, codec decode/encode, audio effects, IO sources
  (file/http/flash/codec_dev), recording, muxing, and the high-level packages esp_audio_simple_player,
  esp_player, esp_capture.
  Trigger words: "ESP-GMF", "GMF", "gmf_core", "gmf_audio", "gmf_io", "esp_player", "esp_capture",
  "esp_audio_simple_player", "esp_gmf_pipeline", "esp_gmf_element", "esp_gmf_pool", "多媒体框架", "音频流水线", "播放器", "录音"
tags:
  - embedded
  - ESP32
  - ESP-IDF
  - multimedia
  - audio
  - video
  - streaming
  - pipeline
  - espressif
  - firmware
license: LicenseRef-Espressif-Modified-MIT
compatibility: Built on ESP-IDF (>= v5.4.3 release/v5.4, >= v5.5.2 release/v5.5, or >= v6.0); targets ESP32 family SoCs (ESP32, ESP32-S3, ESP32-P4, ESP32-C3, etc.)
metadata:
  author: Community
  version: "1.1.0"
---

# esp-gmf-skill

面向 ESP-GMF（Espressif General Multimedia Framework）的 AI 开发技能。提供基于真实仓库文档与示例的场景化配方、API 速查、配置参考与陷阱清单，帮助 AI 代理在 ESP32 上正确构建音频/视频/图像流式处理流水线。ESP-GMF 由 GMF-Core、Elements、Packages、GMF-Examples 四个模块组成，RAM 占用最低可达 7 KB，适用于资源受限的 IoT 多媒体产品。

## Core Principles

1. **优先使用高级包，而非手写流水线** — 快速落地用 `esp_audio_simple_player` / `esp_player` / `esp_capture`；需要细粒度控制才下沉到 GMF-Core 的 pipeline/element。
2. **三对象分工不可混淆** — element 负责 `open/process/close` 算法逻辑；pipeline 负责编排连接与事件分发；task 负责按 job 列表调度 element 回调。应用代码只与 pipeline 的控制 API 交互。
3. **Pool 模式：先注册模板，再按名实例化** — 应用启动时用 `gmf_loader_setup_*` 或 `esp_gmf_pool_register_element/io` 把所有支持的 element/IO 注册进 pool；构建流水线时仅提供名字数组，pool 用 `esp_gmf_obj_dupl` 复制实例并链式连接。
4. **流水线构建六步** — `pool_init → new_pipeline(name[]) → task_init → bind_task → loading_jobs → set_event/run`，顺序不能乱。`loading_jobs` 把 element 的 open/process 注册进 task job 列表。
5. **依赖型 element 必须等上游 REPORT_INFO** — 设置 `dependency = true` 的 element（如 rate_cvt、resampler）需上游解析出采样率并通过 `esp_gmf_element_notify_snd_info` 上报 `esp_gmf_info_sound_t` 后，pipeline 才为其注册 open/process job。
6. **acquire/release 必须严格成对** — process 内 `acquire_in/release_in` 与 `acquire_out/release_out` 成对调用；任何错误分支都要先 release 已 acquire 的 payload，否则 port 泄漏耗尽缓冲区。
7. **错误/结束都会触发 close** — 无论自然结束、用户 stop 还是 element FAIL，框架都会对每个已 open 的 element 依次 close。close 实现应只记错并继续，避免资源泄漏。
8. **ERROR 后必须 reset 才能 run** — 进入 `ESP_GMF_EVENT_STATE_ERROR` 后，需 `esp_gmf_pipeline_reset` → `esp_gmf_pipeline_loading_jobs` → `run`，否则 run 返回 `ESP_GMF_ERR_NOT_SUPPORT`。
9. **同步/异步 IO 由 thread.stack 决定** — `thread.stack > 0` 且 `buffer_size > 0` 为异步（内部起 `io_process` 任务做预取）；`thread.stack == 0` 为同步。HTTP 默认异步，file/codec_dev/embed_flash 默认同步。
10. **格式协商用 FourCC** — 所有媒体格式（`ESP_FOURCC_MP3`/`AAC`/`H264`/`PCM`/`NV12`...）统一为 `esp_fourcc_t`，跨模块（element、codec 库、capture、render）作为通用格式语言。
11. **menuconfig 裁剪体积** — 通��� `gmf_loader` 的 Kconfig 仅启用实际需要的解码器/编码器/效果/IO，按需减少固件大小与内存占用。
12. **控制 API 默认阻塞，2 s 超时** — run/stop/pause/resume 默认阻塞，最大等待 `DEFAULT_TASK_OPT_MAX_TIME_MS`（2000 ms），可用 `esp_gmf_task_set_timeout` 调整；stop 超时后仍在后台继续，不等于失败。

## When to Use

**适用：**
- 在 ESP32 上构建音频播放（本地/HTTP/Flash）、录音、编码、混音、容器封装
- 基于 GMF-Core 自定义 element/IO 或手工编排多级 pipeline
- 使用 `esp_player` 做音视频同步播放（含 HLS、容器解封装）
- 使用 `esp_capture` 做音视频采集录制（含多 sink 并行流式 + 本地存储、overlay、AEC）
- 使用 `esp_bt_audio` 做蓝牙音箱/耳机/免提通话，或 LE Audio TMAP 单播/广播
- 使用 `esp_audio_render` 做多路 PCM 混音（音乐 + TTS + 提示音、多轨合成）
- 使用 `esp_video_render` 做视频显示与 UI 叠加（H264/MJPEG 上屏、双目合成）
- 使用 `gmf_ai_audio` 做唤醒词、命令词、AEC、NS、VAD、DOA 语音前端
- 调试流水线状态机、错误恢复、依赖型 element 启动问题

**不适用：**
- 非 Espressif 芯片的音视频开发
- ESP-ADF 旧 `audio_pipeline` 模块（ESP-GMF 是其后继，但 API 不同，不可混用）
- 纯 PCB / 硬件原理图设计
- 与多媒体无关的通用 ESP-IDF 编程

---

## Scenario Quick Reference (Recipes)

当用户意图匹配下列场景时，**先读对应配方**——内含完整调用链、分步说明、真实代码与常见错误表。

### 高级播放器包

| 配方 | 场景 |
|---|---|
| `recipes/simple_player.md` | 用 `esp_audio_simple_player` 快速实现 URI 驱动的音频播放（file/http/embed/raw） |
| `recipes/esp_player.md` | 用 `esp_player` 做音视频同步播放、seek、多轨道、HLS |
| `recipes/esp_bt_audio.md` | 蓝牙音频（A2DP Sink/Source、HFP 免提、AVRCP/PBAP；LE Audio TMAP 单播/广播）经 `esp_gmf_io_bt` 接入 pipeline |
| `recipes/esp_capture.md` | 用 `esp_capture` 做高级音视频采集（source/path/sink 模型、AEC、多 sink 并行流式+MP4、overlay、自定义链） |

### 音频渲染与混音

| 配方 | 场景 |
|---|---|
| `recipes/esp_audio_render.md` | 用 `esp_audio_render` 多路混音渲染（per-stream 与 post-mix 处理链、solo、fade、运行时切格式） |

### 视频采集与显示

| 配方 | 场景 |
|---|---|
| `recipes/esp_video_render.md` | 用 `esp_video_render` 视频合成与显示（H264/MJPEG 上屏、overlay→container→widget UI 叠加、LCD/LVGL 后端、双目 dual_stream） |

### 基于 GMF-Core 的流水线

| 配方 | 场景 |
|---|---|
| `recipes/pipeline_play_embed.md` | 用 pool 构建「embed_flash → aud_dec → 效果链 → codec_dev」最小播放流水线 |
| `recipes/pipeline_play_http.md` | HTTP/HTTPS 音频播放（异步 IO、TLS 栈调大、URL 评分选 IO） |
| `recipes/pipeline_record.md` | codec_dev 录音 → aud_enc → io_file 写 SD 卡 |
| `recipes/pipeline_muxer.md` | aud_enc → aud_muxer 封装 TS/MP4/FLV 等容器到文件或流 |

### Element 与效果

| 配方 | 场景 |
|---|---|
| `recipes/audio_effects.md` | 运行时调用 EQ/ALC/Sonic/Fade/DRC/MBC 的命名 setter 调整音效 |
| `recipes/custom_element.md` | 自定义 audio element 模板：open/process/close + 端口属性 + acquire/release |
| `recipes/runtime_methods.md` | 用 `AMETHOD(MODULE, METHOD)` + `esp_gmf_element_exe_method` 解耦接口与实现 |
| `recipes/gmf_ai_audio.md` | AI 语音前端 element（ai_afe 全功能、ai_aec/ai_wn/ai_ns/ai_vad/ai_doa 单能力；wakeup+VAD 状态机、命令词、手动唤醒） |

### 控制与高级

| 配方 | 场景 |
|---|---|
| `recipes/loop_play.md` | 无缝循环播放：`esp_gmf_task_set_strategy_func` 配合 RESET 动作与 `esp_gmf_io_reload` |
| `recipes/state_error_recovery.md` | 状态机、ERROR 恢复、pause/seek、stop 超时处理 |

---

## Pipeline 状态机

pipeline 与 task 共享 `esp_gmf_event_state_t`。应用通过事件回调感知每次状态迁移。

```
NONE → INITIALIZED → OPENING → RUNNING → FINISHED   (is_done)
                         │        │ ↕ PAUSED
                         │        └→ STOPPED   (用户 stop)
                         └─────────└→ ERROR    (job FAIL)
   STOPPED / FINISHED / ERROR  ──reset──→  INITIALIZED
```

| 状态 | 值 | 含义 |
|---|---|---|
| `ESP_GMF_EVENT_STATE_NONE` | 0 | 对象刚创建，未初始化 |
| `ESP_GMF_EVENT_STATE_INITIALIZED` | 1 | 初始化完成，可启动 |
| `ESP_GMF_EVENT_STATE_OPENING` | 2 | element open 阶段进行中 |
| `ESP_GMF_EVENT_STATE_RUNNING` | 3 | process 循环调度中 |
| `ESP_GMF_EVENT_STATE_PAUSED` | 4 | 暂停，上下文保留 |
| `ESP_GMF_EVENT_STATE_STOPPED` | 5 | 用户发起 stop，close 进行中 |
| `ESP_GMF_EVENT_STATE_FINISHED` | 6 | 数据自然结束（payload `is_done`） |
| `ESP_GMF_EVENT_STATE_ERROR` | 7 | job 返回失败，清理中 |

控制 API 有效状态速查：

| API | 有效状态 |
|---|---|
| `esp_gmf_pipeline_run` | INITIALIZED / STOPPED / FINISHED |
| `esp_gmf_pipeline_stop` | RUNNING / PAUSED |
| `esp_gmf_pipeline_pause` | RUNNING |
| `esp_gmf_pipeline_resume` | PAUSED |
| `esp_gmf_pipeline_reset` | 非 RUNNING |
| `esp_gmf_pipeline_seek` | PAUSED / STOPPED / FINISHED |

---

## 组件支持矩阵

| 组件 | 作用 | 依赖 | 典型入口 |
|---|---|---|---|
| `gmf_core` | 框架核心（pipeline/task/element/pool/payload/port/data_bus） | 仅 ESP-IDF 系统能力 | `esp_gmf_pool_init` |
| `gmf_io` | file/http/embed_flash/i2s_pdm/codec_dev 五类 IO | `gmf_core`、`esp_codec_dev` | `esp_gmf_io_file_init` |
| `gmf_audio` | 17 个音频 element（codec/format/effects/channel/muxer） | `gmf_core`、`esp_audio_codec`、`esp_audio_effects` | `esp_gmf_audio_dec_init` |
| `gmf_video` | 视频编解码/特效 element | `gmf_core`、`esp_video_codec` | — |
| `gmf_ai_audio` | AI 音频（AEC/NS/AGC/VAD/WakeNet） | `esp-sr`、`gmf_core` | — |
| `gmf_loader` | 按 menuconfig 批量注册 element/IO 进 pool | 各 element/IO | `gmf_loader_setup_io_default` 等 |
| `gmf_app_utils` | 板级外设、内存检测工具 | `esp_board_manager` | `esp_gmf_app_*` |
| `esp_audio_simple_player` | 简易音频播放器 | `gmf_audio`、`gmf_io` | `esp_audio_simple_player_new` |
| `esp_player` | 音视频同步播放器 | render 句柄 | `esp_player_init` |
| `esp_capture` | 高级音视频采集 | `gmf-audio`、`gmf-video`、`esp_muxer` | `esp_capture_open` |
| `esp_audio_render` | 带混音的音频渲染 | `gmf_core`、`gmf-audio` | — |
| `esp_asrc` | 硬件/软件采样率转换 | `esp_audio_effects` | `esp_gmf_asrc_init` |

---

## Critical Pitfalls (Must Read)

下列是最常见的错误。违反任一条都会导致流水线不工作。

### 1. 构建顺序：必须先 bind_task 再 loading_jobs 再 run

```c
// ❌ 错误 — 未 loading_jobs 直接 run，job 列表为空
esp_gmf_pipeline_bind_task(pipe, task);
esp_gmf_pipeline_run(pipe);   // 没有任何 element 被调度

// ✅ 正确
esp_gmf_pipeline_bind_task(pipe, task);
esp_gmf_pipeline_loading_jobs(pipe);   // 注册 open/process job
esp_gmf_pipeline_set_event(pipe, cb, ctx);
esp_gmf_pipeline_run(pipe);
```

### 2. reset 后必须重新 loading_jobs

```c
// ❌ 错误 — reset 清空了 job 列表，直接 run 不会恢复
esp_gmf_pipeline_reset(pipe);
esp_gmf_pipeline_run(pipe);   // job 列表仍为空

// ✅ 正确
esp_gmf_pipeline_reset(pipe);
esp_gmf_pipeline_loading_jobs(pipe);   // 重新注册 job
esp_gmf_pipeline_run(pipe);
```

### 3. ERROR 状态不能直接 run

```c
// ❌ 错误 — ERROR 后直接 run 返回 ESP_GMF_ERR_NOT_SUPPORT
esp_gmf_pipeline_run(pipe);   // 状态仍为 ERROR

// ✅ 正确 — reset 回到 INITIALIZED 再 run
esp_gmf_pipeline_reset(pipe);
esp_gmf_pipeline_loading_jobs(pipe);
esp_gmf_pipeline_run(pipe);
```

### 4. acquire/release 必须成对（含错误分支）

```c
// ❌ 错误 — acquire_out 失败后没有 release_in，port 泄漏
ret = esp_gmf_port_acquire_in(in, &in_load, 1024, ESP_GMF_MAX_DELAY);
ret = esp_gmf_port_acquire_out(out, &out_load, 1024, ESP_GMF_MAX_DELAY);
if (ret < 0) return ESP_GMF_JOB_ERR_FAIL;   // in_load 泄漏！

// ✅ 正确 — 错误分支先释放已 acquire 的 payload
ret = esp_gmf_port_acquire_in(in, &in_load, 1024, ESP_GMF_MAX_DELAY);
if (ret < 0) return (ret == ESP_GMF_IO_ABORT) ? ESP_GMF_JOB_ERR_ABORT : ESP_GMF_JOB_ERR_FAIL;
ret = esp_gmf_port_acquire_out(out, &out_load, in_load->valid_size, ESP_GMF_MAX_DELAY);
if (ret < 0) { esp_gmf_port_release_in(in, in_load, 0); return ESP_GMF_JOB_ERR_FAIL; }
/* ... 处理 ... */
esp_gmf_port_release_out(out, out_load, 0);
esp_gmf_port_release_in(in, in_load, 0);
```

### 5. 解码器格式要显式配置或依赖自动检测

```c
// ❌ 错误 — use_frame_dec=true 却不设 dec_type，无法解析
cfg.use_frame_dec = true;   // 跳过解析层，但又没指定格式

// ✅ 正确 — 显式指定格式（推荐，启动快且避免误判）
esp_audio_simple_dec_cfg_t cfg = DEFAULT_ESP_GMF_AUDIO_DEC_CONFIG();
cfg.dec_type = ESP_AUDIO_SIMPLE_DEC_TYPE_MP3;
esp_gmf_audio_dec_init(&cfg, &dec);

// ✅ 也正确 — 用 helper 从 URI 推断 FourCC 后 reconfig
esp_gmf_info_sound_t info = {0};
esp_gmf_audio_helper_get_audio_type_by_uri(uri, &info.format_id);
esp_gmf_audio_dec_reconfig_by_sound_info(dec_el, &info);
```

### 6. HTTPS 播放：task 栈必须调大（TLS 握手）

```c
// ❌ 错误 — 用默认 4 KB 栈跑 HTTPS，mbedtls 握手栈溢出
esp_gmf_task_cfg_t cfg = DEFAULT_ESP_GMF_TASK_CONFIG();

// ✅ 正确 — HTTPS 至少 8 KB 栈，并放宽控制超时
esp_gmf_task_cfg_t cfg = DEFAULT_ESP_GMF_TASK_CONFIG();
cfg.thread.stack = 8 * 1024;
esp_gmf_task_init(&cfg, &task);
esp_gmf_task_set_timeout(task, 20000);
```

### 7. element 名字数组必须与 pool 注册的 tag 一致

```c
// ❌ 错误 — 名字拼写与 pool 注册 tag 不符，pool_new_pipeline 查不到
const char *name[] = {"audio_decoder", "resampler"};   // 不存在

// ✅ 正确 — 用 gmf_loader 默认注册的 tag
const char *name[] = {"aud_dec", "aud_rate_cvt", "aud_ch_cvt", "aud_bit_cvt"};
esp_gmf_pool_new_pipeline(pool, "io_file", name, 4, "io_codec_dev", &pipe);
```

### 8. stop 超时不等于失败

```c
// ❌ 错误 — 把 stop 超时当致命错误，重复 stop 导致状态混乱
if (esp_gmf_pipeline_stop(pipe) == ESP_GMF_ERR_TIMEOUT) { abort(); }

// ✅ 正确 — 超时仅表示本次同步等待未完成，stop 仍在后台进行
esp_gmf_pipeline_stop(pipe);   // 可继续后续清理，或等待事件回调确认 STOPPED
```

### 9. 切换音轨要正确处理 IO done/clear_done

```c
// ❌ 错误 — 标记 done 后直接换 URI，下游 IO 不前进
esp_gmf_io_done(io);
esp_gmf_io_set_uri(io, next_uri);

// ✅ 正确 — done 后必须 clear_done 才能让后续 IO 继续推进
esp_gmf_io_done(io);
/* 等待 pipeline 处理完残留 payload */
esp_gmf_io_set_uri(io, next_uri);
esp_gmf_io_clear_done(io);
```

### 10. embed_flash URL 格式必须带下划线索引

```c
// ❌ 错误 — 缺少 index_name 下划线分隔，报 "No _ in file name"
esp_gmf_io_set_uri(io, "embed://tone/startup.mp3");

// ✅ 正确 — embed://<group>/<index>_<name>.<ext>
esp_gmf_io_set_uri(io, "embed://tone/0_startup.mp3");
```

### 11. 录音编码：AMR 需要更大栈与合法 bitrate

```c
// ❌ 错误 — 用默认 4 KB 栈编码 AMR，运行时栈溢出
esp_gmf_task_cfg_t cfg = DEFAULT_ESP_GMF_TASK_CONFIG();

// ✅ 正确 — AMR 编码栈调到 40 KB（可放 PSRAM），并按格式设置 bitrate
esp_gmf_task_cfg_t cfg = DEFAULT_ESP_GMF_TASK_CONFIG();
cfg.thread.stack = 40 * 1024;
cfg.thread.stack_in_ext = true;
// AMRWB 用 ESP_AMRWB_ENC_BITRATE_MD885，AMRNB 用 ESP_AMRNB_ENC_BITRATE_MR122
```

### 12. 策略函数内禁止调用 pipeline/task 控制 API

```c
// ❌ 错误 — 策略函数里调 stop/run，触发超时错误
static int strategy(int type, void *ctx) {
    esp_gmf_pipeline_stop(pipe);   // 死锁/超时
    return 0;
}

// ✅ 正确 — 只返回动作枚举；要等外部事件用信号量阻塞
static int strategy(int type, void *ctx) {
    return GMF_TASK_STRATEGY_ACTION_RESET;   // 自动续播下一曲，不重建 pipeline
}
```

---

## Execution Workflow

| 步骤 | 名称 | 说明 |
|------|------|------|
| 1 | 需求分析 | 区分「快速落地」(用高级包) 与「细粒度控制」(用 GMF-Core) |
| 2 | 配方匹配 | 查 `recipes/` 找最接近的场景，沿用其调用链 |
| 3 | API 查证 | 未覆盖的 API 查 `resources/api_reference.md`；配置项查 `config_reference.md` |
| 4 | 校验 | 核对 element tag、构建顺序、栈大小、IO 同步/异步模式、状态机约束 |
| 5 | 方案确认 | 向用户说明：依赖、构建步骤、board 选择、menuconfig 关键项 |
| 6 | 执行 | 新项目：用 `idf.py create-project-from-example "espressif/gmf_examples=1.0.0:<name>"` 拉取最接近示例再改；已有项目：就地编辑 |
| 7 | 构建 | `idf.py bmgr -b <board>` 选板 → `idf.py menuconfig` → `idf.py build` |
| 8 | 烧录 | `idf.py -p PORT flash monitor` |
| 9 | 调试 | 串口看 `CHANGE_STATE` 日志确认状态机走向；用 `ESP_GMF_POOL_SHOW_ITEMS` 确认注册项 |

### 步骤 6 详情 — 项目创建策略

**目标目录无项目时（首次创建）：**

1. 根据 `resources/example_list.md` 选最接近的真实示例：
   - 播放 embed flash 音乐 → `gmf_examples/basic_examples/pipeline_play_embed_music`
   - 播放 SD 卡音乐 → `pipeline_play_sdcard_music`
   - 播放 HTTP 音乐 → `pipeline_play_http_music`
   - 录音到 SD 卡 → `pipeline_record_sdcard`
   - 容器封装录制 → `pipeline_record_audio_muxer`
   - 效果/混音 → `pipeline_audio_effects`
   - 无缝循环 → `pipeline_loop_play_no_gap`
   - 多源切换 → `pipeline_play_multi_source_music`
   - HTTP 下载到 SD 卡 → `pipeline_http_download_to_sdcard`
2. 用组件管理器拉取：`idf.py create-project-from-example "espressif/gmf_examples=1.0.0:<example>"`
3. 在拉取的代码上改名/调整，比从零写更快更可靠。

**目标目录已有项目时：** 就地编辑，除非用户明确要求否则不要覆盖。

---

## Failure Strategies

| 情况 | 处置 |
|---|---|
| API 不在 `resources/api_reference.md` | 立即停止，告知用户该 API 不存在，不要臆造 |
| 依赖型 element 迟迟不 open | 检查上游是否调用 `esp_gmf_element_notify_snd_info`，或应用层 `esp_gmf_pipeline_report_info` 主动上报 |
| HTTPS 播放卡死/复位 | 把 task 栈调到 ≥ 8 KB，`esp_gmf_task_set_timeout(task, 20000)` |
| IO 选错 | 用 `esp_gmf_pool_get_io_tag_by_url` 按评分选 IO；自定义 IO 可返回 `ESP_GMF_IO_SCORE_PERFECT` 抢占 |
| stop 一直不返回 | element process 内有长阻塞（网络/互斥），增大 `esp_gmf_task_set_timeout` 或在 element 内查 abort 标志主动退出 |
| 录音 AMR/AAC 栈溢出 | task 栈调到 40 KB 并 `stack_in_ext=true` 放 PSRAM |
| 切歌后无声音 | `esp_gmf_io_done` 后未 `esp_gmf_io_clear_done`，或未 reset/loading_jobs |

## References

- 场景配方 → `recipes/` 目录
- API 速查 → `resources/api_reference.md`
- 配置项参考 → `resources/config_reference.md`
- 陷阱汇总 → `resources/pitfalls.md`
- 示例清单 → `resources/example_list.md`
- 状态机 → 本文档「Pipeline 状态机」一节
- 仓库官方文档 → https://docs.espressif.com/projects/esp-gmf/en/latest/ 与 https://docs.espressif.com/projects/esp-gmf/zh_CN/latest/
