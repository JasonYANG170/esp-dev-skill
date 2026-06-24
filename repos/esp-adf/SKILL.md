---
name: esp-adf-skill
description: >-
  AI Skill for Espressif Audio Development Framework (ESP-ADF) firmware development. Used when users
  need to create, modify, or debug audio/multimedia applications on ESP32 / ESP32-S2 / ESP32-S3 /
  ESP32-C3 / ESP32-C6 / ESP32-C5 / ESP32-P4 SoCs, including audio playback, recording, streaming
  (HTTP / HLS / FatFs / SPIFFS), codec decoding/encoding (MP3/AAC/FLAC/OGG/OPUS/AMR/WAV), audio
  pipelines, peripherals, services (Wi-Fi/BT/OTA), and speech recognition integration.
  Trigger words: "ESP-ADF", "esp-adf", "audio_pipeline", "i2s_stream", "mp3_decoder", "http_stream",
  "音频", "音频流水线", "录音", "播放", "Espressif Audio", "ESP32 音频", "LyraT", "Korvo", "audio_board"
tags:
  - embedded
  - audio
  - espressif
  - esp32
  - esp-adf
  - codec
  - i2s
  - pipeline
  - streaming
  - firmware
license: ESPRESSIF MIT
compatibility: Requires ESP-IDF (release/v5.1 - v5.5) + ESP-ADF master; CMake build via idf.py; targets ESP32 / ESP32-S2 / ESP32-S3 / ESP32-C3 / ESP32-C5 / ESP32-C6 / ESP32-P4
metadata:
  author: Community
  version: "1.1.0"
---

# esp-adf-skill

面向 Espressif Audio Development Framework（ESP-ADF）的 AI 技能。提供基于真实仓库文档与源码的场景化配方、API 速查、配置参考与常见陷阱。ESP-ADF 是 ESP32 系列芯片的官方高级音频开发框架，核心思想是用「Audio Element → Audio Pipeline」组合出播放 / 录音 / 流媒体应用，并辅以 Services（Wi-Fi、BT、OTA 等）与 Peripherals 抽象层。

## Core Principles

1. **Never guess APIs** — 任何函数名、结构体、宏、配置项必须能在 `resources/` 或仓库 `docs/`、`components/*/include` 中找到；找不到即视为不存在，不要编造。
2. **Pipeline 是 ESP-ADF 的核心抽象** — 一个应用通常由若干 `audio_element_handle_t`（stream / codec / filter）注册进一条 `audio_pipeline_handle_t`，再用 `audio_pipeline_link()` 按 `link_tag` 顺序串联，元素之间通过 ringbuffer 传递数据。
3. **Element 即任务** — 每个 element 启动后是独立的 FreeRTOS 任务，回调链为 `open → [loop: read → process → write] → close`；不要在主线程直接读写 element 的数据，应通过事件接口控制。
4. **初始化顺序固定** — `audio_board_init()`（codec）→ `audio_pipeline_init()` → 创建各 element → `audio_pipeline_register()` → `audio_pipeline_link()` → `audio_pipeline_run()`。播放结束前停止必须 `stop → wait_for_stop → terminate`。
5. **板级支持由 Kconfig 选择** — `CONFIG_*_BOARD`（如 `CONFIG_ESP_LYRAT_V4_3_BOARD`、`CONFIG_ESP32_S3_KORVO2_V3_BOARD`）决定链接哪一套 `board.h` / codec 驱动；自定义板用 `CONFIG_AUDIO_BOARD_CUSTOM`。`board.h` 提供 `audio_board_init / audio_board_key_init / audio_board_sdcard_init` 等统一入口。
6. **事件统一走 audio_event_iface** — pipeline 与 peripherals 都把事件汇聚到一个 `audio_event_iface_handle_t`，主循环用 `audio_event_iface_listen()` 读取 `audio_event_iface_msg_t`，按 `msg.source_type` + `msg.cmd` 分发。
7. **采样率/位深动态对齐** — 解码器解析出真实格式后，会发 `AEL_MSG_CMD_REPORT_MUSIC_INFO` 事件；此时必须调用 `i2s_stream_set_clk(writer, sr, bits, ch)` 重配 I2S，否则播放变速 / 变调。
8. **stream 分 reader / writer** — `audio_stream_type_t` 取 `AUDIO_STREAM_READER`（取数据）或 `AUDIO_STREAM_WRITER`（送数据），通过 `i2s_cfg.type` / `fatfs_cfg.type` 等设置；raw stream 不创建线程，仅做数据中转。
9. **read 回调接入自定义数据源** — 对内存/Flash 中的数据，用 `audio_element_set_read_cb(el, fn, ctx)` 给第一个 element 注入数据，回调返回字节数，结束时返回 `AEL_IO_DONE`。
10. **ESP-ADF 与 ESP-IDF 版本绑定** — `master` 分支仅与 IDF release/v5.1–v5.5 兼容，IDF master 不保证兼容；版本不匹配会导致编译失败。
11. **资源释放顺序敏感** — 销毁前必须先 `audio_pipeline_remove_listener()`，再 `audio_event_iface_destroy(evt)`，最后 `audio_pipeline_deinit()` 与各 `audio_element_deinit()`；顺序错误会内存泄漏或崩溃。
12. **esp-adf-libs / esp-sr 为预编译子模块** — 编解码器（mp3/aac/flac 等）与语音算法以预编译库形式提供，需 `git clone --recursive`；缺失子模块会导致 `mp3_decoder.h` 等头文件找不到。

## When to Use

**Applicable:**
- 创建 ESP-ADF 工程（播放器 / 录音机 / 流媒体 / 网络收音机）
- 构建 audio pipeline，串联 stream + codec + filter
- 接入数据源：HTTP/HLS、FatFs（SD 卡）、SPIFFS、内存 Flash、TCP、A2DP
- 使用编解码器：MP3 / AAC / FLAC / OGG / OPUS / AMR(NB/WB) / WAV / TS
- 音频处理：EQ / Downmix / Sonic / Resample / ALC / AEC+NS+AGC 算法流
- 外设与服务：Wi-Fi、蓝牙（A2DP Sink/Source、HFP）、OTA、SD 卡、按键、触摸、显示
- 语音识别（WakeNet / MultiNet）与 TTS、VoIP、RTMP、DLNA

**Not applicable:**
- 不使用 ESP-ADF 的纯 ESP-IDF 音频项目（直接用 `driver/i2s.h`，本技能不覆盖）
- 非 ESP32 系列芯片（如 STM32、RK）的音频开发
- PCB 硬件设计与原理图、声学腔体设计
- esp-sr 模型训练与自定义 WakeNet 命令词离线生成（仅覆盖框架集成）

---

## Scenario Quick Reference (Recipes)

当用户意图匹配下列场景时，**先阅读对应 recipe** — 其中包含完整调用链、分步说明、常见错误与可复制代码。

### Get Started / 播放

| recipe | scenario |
|---|---|
| `recipes/play_mp3_flash.md` | 从 Flash 内存播放 MP3（自定义 read 回调，最经典入门） |
| `recipes/play_http_mp3.md` | 通过 HTTP/HTTPS 流式播放 MP3（含 Wi-Fi 连接） |
| `recipes/play_sdcard_fatfs.md` | 从 SD 卡（FatFs）播放音乐文件 |

### 录音 / 编码

| recipe | scenario |
|---|---|
| `recipes/record_to_sdcard.md` | I2S 录音并编码 WAV/AMR 写入 SD 卡 |
| `recipes/speech_recognition.md` | 语音唤醒（WakeNet）+ 命令词（MultiNet）+ VAD + `audio_recorder` 高层录音器 |

### 流与数据源

| recipe | scenario |
|---|---|
| `recipes/pipeline_streams.md` | 各类 stream（i2s / fatfs / spiffs / http / raw / tcp / tone / pwm）速查与组合 |

### 蓝牙与外设

| recipe | scenario |
|---|---|
| `recipes/bt_a2dp_hfp.md` | 蓝牙服务（A2DP Sink/Source、HFP）接入 pipeline |
| `recipes/peripherals_services.md` | 外设集（Wi-Fi / SD 卡 / 触摸 / 按键）与事件集成 |
| `recipes/display_service.md` | 显示服务（LED 灯效模式 + PWM/AW2013/IS31x/WS2812 驱动） |

### 音频处理 / 高级

| recipe | scenario |
|---|---|
| `recipes/audio_processing.md` | EQ / Downmix / Sonic / Resample / ALC 处理元素接入 |
| `recipes/ota_service.md` | OTA 固件升级服务集成 |
| `recipes/cli_console.md` | CLI 命令行交互（esp-console + ADF） |
| `recipes/playlist.md` | 播放列表管理（sdcard_scan + SD/DRAM/NVS/Partition 多后端） |
| `recipes/voip_sip.md` | VoIP / SIP 网络电话（av_stream + AEC + PBX） |
| `recipes/cloud_tts.md` | 云端 TTS（百度 Speech / AWS Polly / Google Translate） |

---

## Audio Board Support（Kconfig `CONFIG_*_BOARD`）

下表来自 `components/audio_board/Kconfig.projbuild`，决定链接的 `board.h` 与 codec 驱动。

| Kconfig 符号 | 开发板 | 主芯片 |
|---|---|---|
| `CONFIG_ESP_LYRAT_V4_3_BOARD` | ESP32-LyraT V4.3 | ESP32 |
| `CONFIG_ESP_LYRAT_V4_2_BOARD` | ESP32-LyraT V4.2 | ESP32 |
| `CONFIG_ESP_LYRATD_MSC_V2_2_BOARD` | ESP32-LyraTD-MSC V2.2 | ESP32 |
| `CONFIG_ESP_LYRAT_MINI_V1_1_BOARD` | ESP32-LyraT-Mini V1.1 | ESP32 |
| `CONFIG_ESP32_KORVO_DU1906_BOARD` | ESP32-Korvo-DU1906 | ESP32 |
| `CONFIG_ESP32_S2_KALUGA_1_V1_2_BOARD` | ESP32-S2-Kaluga-1 v1.2 | ESP32-S2 |
| `CONFIG_ESP32_S3_KORVO2_V3_BOARD` | ESP32-S3-Korvo-2 v3 | ESP32-S3 |
| `CONFIG_ESP32_S3_KORVO2L_V1_BOARD` | ESP32-S3-Korvo-2L v1 | ESP32-S3 |
| `CONFIG_ESP32_S3_BOX_BOARD` | ESP32-S3-BOX | ESP32-S3 |
| `CONFIG_ESP32_S3_BOX_3_BOARD` | ESP32-S3-BOX-3 | ESP32-S3 |
| `CONFIG_ESP32_C3_LYRA_V2_BOARD` | ESP32-C3-Lyra v2.0 | ESP32-C3 |
| `CONFIG_ESP32_C6_DEVKIT_BOARD` | ESP32-C6-DEVKIT | ESP32-C6 |
| `CONFIG_ESP32_P4_FUNCTION_EV_BOARD` | ESP32-P4-FUNCTION-EV-BOARD | ESP32-P4 |
| `CONFIG_AUDIO_BOARD_CUSTOM` | 自定义板（需自带 board 实现） | 任意 |

> 选板方式：`idf.py menuconfig` → Audio board，或直接写 `sdkconfig.defaults`。Korvo-DU1906 还需选择 DAC/ADC 子芯片（`CONFIG_ESP32_KORVO_DU1906_DAC_*` / `CONFIG_ESP32_KORVO_DU1906_ADC_*`）。

---

## Audio Element 状态机

`audio_element_state_t`（来自 `audio_element.h`）：

```
AEL_STATE_NONE → AEL_STATE_INIT → AEL_STATE_INITIALIZING → AEL_STATE_RUNNING
                                                       ↕                ↕
                                                AEL_STATE_PAUSED   AEL_STATE_STOPPED
                                                       ↓                ↓
                                                 (resume)        AEL_STATE_FINISHED / AEL_STATE_ERROR
```

关键消息命令（`audio_element_msg_cmd_t`）：`AEL_MSG_CMD_FINISH`(2)、`AEL_MSG_CMD_STOP`(3)、`AEL_MSG_CMD_PAUSE`(4)、`AEL_MSG_CMD_RESUME`(5)、`AEL_MSG_CMD_DESTROY`(6)、`AEL_MSG_CMD_REPORT_STATUS`(8)、`AEL_MSG_CMD_REPORT_MUSIC_INFO`(9)。

IO 返回值（`audio_element_err_t`）：`AEL_IO_OK`、`AEL_IO_FAIL`(-1)、`AEL_IO_DONE`(-2)、`AEL_IO_ABORT`(-3)、`AEL_IO_TIMEOUT`(-4)、`AEL_PROCESS_FAIL`(-5)。**自定义 read/write 回调返回 `AEL_IO_DONE` 表示数据结束**。

---

## Critical Pitfalls (Must Read)

这些是最高频的错误，违反任何一条都会导致固件不工作。

### 1. Pipeline 元素必须先 register 再 link，顺序即数据流

```c
// ❌ WRONG — 没 register 就 link，或 link_tag 顺序与数据流不符
audio_pipeline_link(pipeline, (const char *[]){"i2s","mp3"}, 2);

// ✅ CORRECT — 按数据流方向 register，link_tag 顺序 = 数据流
audio_pipeline_register(pipeline, http_stream_reader, "http");
audio_pipeline_register(pipeline, mp3_decoder,        "mp3");
audio_pipeline_register(pipeline, i2s_stream_writer,  "i2s");
const char *link_tag[3] = {"http", "mp3", "i2s"};
audio_pipeline_link(pipeline, link_tag, 3);
```

### 2. 解码器报告采样率后必须重配 I2S

```c
// ❌ WRONG — 用固定 44100 播放 22050 的源，会变速变调
i2s_stream_writer = i2s_stream_init(&i2s_cfg);  // 默认 44100
audio_pipeline_run(pipeline);

// ✅ CORRECT — 监听 MUSIC_INFO 事件后动态 set_clk
if (msg.cmd == AEL_MSG_CMD_REPORT_MUSIC_INFO && msg.source == mp3_decoder) {
    audio_element_info_t info = {0};
    audio_element_getinfo(mp3_decoder, &info);
    i2s_stream_set_clk(i2s_stream_writer, info.sample_rates, info.bits, info.channels);
}
```

### 3. 停止 pipeline 三连：stop → wait_for_stop → terminate

```c
// ❌ WRONG — 直接 terminate 或只 stop，任务残留 / ringbuffer 泄漏
audio_pipeline_terminate(pipeline);

// ✅ CORRECT
audio_pipeline_stop(pipeline);
audio_pipeline_wait_for_stop(pipeline);
audio_pipeline_terminate(pipeline);
// 之后才能 unregister / deinit
```

### 4. 销毁顺序：先摘 listener 再销毁 event_iface

```c
// ❌ WRONG — 先 destroy event 会导致 pipeline 事件悬空
audio_event_iface_destroy(evt);
audio_pipeline_remove_listener(pipeline);

// ✅ CORRECT
audio_pipeline_remove_listener(pipeline);   // 先解绑
audio_event_iface_destroy(evt);             // 再销毁事件接口
audio_pipeline_deinit(pipeline);
audio_element_deinit(i2s_stream_writer);
audio_element_deinit(mp3_decoder);
```

### 5. stream 的 type（reader/writer）必须按数据方向设置

```c
// ❌ WRONG — 录音却用了默认的 WRITER
i2s_stream_cfg_t i2s_cfg = I2S_STREAM_CFG_DEFAULT();  // 默认 WRITER
i2s_stream_reader = i2s_stream_init(&i2s_cfg);         // 名为 reader 实为 writer

// ✅ CORRECT — 录音用 READER
i2s_stream_cfg_t i2s_cfg = I2S_STREAM_CFG_DEFAULT();
i2s_cfg.type = AUDIO_STREAM_READER;
i2s_stream_reader = i2s_stream_init(&i2s_cfg);
```

### 6. 自定义数据源用 read 回调，返回 AEL_IO_DONE 表示��束

```c
// ❌ WRONG — 文件读完仍返回 0，pipeline 永远不结束
int read_cb(audio_element_handle_t el, char *buf, int len, TickType_t w, void *ctx) {
    return 0;
}

// ✅ CORRECT — 无数据时返回 AEL_IO_DONE
int read_cb(audio_element_handle_t el, char *buf, int len, TickType_t w, void *ctx) {
    int remaining = end - start - pos;
    if (remaining == 0) return AEL_IO_DONE;          // 结束
    int n = len < remaining ? len : remaining;
    memcpy(buf, start + pos, n);
    pos += n;
    return n;                                          // 返回实际字节数
}
audio_element_set_read_cb(mp3_decoder, read_cb, NULL);
```

### 7. ESP-IDF 与 ESP-ADF 版本必须匹配

```text
# ❌ WRONG — ADF master 配 IDF master 或过旧的 IDF
ESP-ADF master  +  ESP-IDF master      → 编译失败
ESP-ADF v2.7    +  ESP-IDF v5.4        → 不支持

# ✅ CORRECT — ADF master 支持 IDF release/v5.1 ~ v5.5
ESP-ADF master  +  ESP-IDF release/v5.3 → OK
```

### 8. Wi-Fi/网络初始化要在 HTTP stream 之前

```c
// ❌ WRONG — 先 run pipeline，Wi-Fi 还没连上
audio_pipeline_run(pipeline);
periph_wifi_wait_for_connected(wifi_handle, portMAX_DELAY);

// ✅ CORRECT — 先联网再 run
esp_periph_start(set, wifi_handle);
periph_wifi_wait_for_connected(wifi_handle, portMAX_DELAY);
audio_pipeline_run(pipeline);
```

### 9. esp-adf-libs / esp-sr 子模块必须递归克隆

```bash
# ❌ WRONG — 浅克隆，缺编解码预编译库，找不到 mp3_decoder.h
git clone https://github.com/espressif/esp-adf.git

# ✅ CORRECT — 递归克隆
git clone --recursive https://github.com/espressif/esp-adf.git
# 或更新已有仓库：
git submodule update --init --recursive
```

### 10. I2S 24-bit 时 buffer_len 必须是 3 的倍数

```c
// ❌ WRONG — 24 位采样下 buffer_len 不是 3 的倍数，DMA 对齐错误
i2s_cfg.buffer_len = 4000;   // 不是 3 的倍数

// ✅ CORRECT — 注释明确：24 位时 buffer_len 必须为 3 的倍数，推荐 3600
i2s_cfg.buffer_len = I2S_STREAM_BUF_SIZE;  // = 3600
```

### 11. Peripherals 事件需单独 set_listener

```c
// ❌ WRONG — 只监听 pipeline，收不到按键 / Wi-Fi 事件
audio_pipeline_set_listener(pipeline, evt);

// ✅ CORRECT — peripherals 的事件接口也要挂到同一个 evt
audio_pipeline_set_listener(pipeline, evt);
audio_event_iface_set_listener(esp_periph_set_get_event_iface(set), evt);
```

### 12. 选择目标芯片要用 idf.py set-target

```bash
# ❌ WRONG — 不设 target，sdkconfig 用默认 esp32，烧到 S3 板会异常
idf.py build

# ✅ CORRECT
idf.py set-target esp32s3   # 或 esp32 / esp32c3 / esp32p4 ...
idf.py build flash monitor
```

---

## Execution Workflow

| Step | Name | Description |
|------|------|-------------|
| 1 | Plan | 明确需求：播放/录音/流？数据源（Flash/SD/HTTP/BT）？格式（MP3/WAV/...）？目标芯片与开发板 |
| 2 | Recipe | 在 `recipes/` 找最接近的场景，遵循其调用链 |
| 3 | Query | recipe 未覆盖的 API，查 `resources/api_reference.md` 与 `resources/config_reference.md` |
| 4 | Validate | 核对头文件包含、element 顺序、stream type、板级 Kconfig |
| 5 | Confirm | 向用户呈现方案：includes、pipeline 拓扑、事件处理、board 选择 |
| 6 | Execute | **新工程：** 拷贝 `examples/` 下最接近的示例到工作目录再改。**已有工程：** 原地编辑 |
| 7 | Check | 检查：register→link 顺序、MUSIC_INFO 后 set_clk、停止三连、销毁顺序 |
| 8 | Build | `idf.py set-target <chip>` → `idf.py build` |
| 9 | Flash | `idf.py -p PORT flash monitor`（默认波特 460800） |

### Step 6 Detail — 工程创建策略

**目标目录无工程（首次创建）：**

1. 从 `examples/` 挑最接近的示例（见 `resources/example_list.md`）：
   - Flash 播 MP3 → `examples/get-started/play_mp3_control`
   - HTTP 播 MP3 → `examples/player/pipeline_http_mp3`
   - SD 卡播放 → `examples/player/pipeline_play_sdcard_music`
   - SPIFFS 播 MP3 → `examples/player/pipeline_spiffs_mp3`
   - 录音到 SD（WAV/AMR）→ `examples/recorder/pipeline_wav_amr_sdcard`
   - 录音到 SD（通用）→ `examples/recorder/pipeline_recording_to_sdcard`
   - 蓝牙 A2DP Sink → `examples/player/pipeline_a2dp_sink_stream`
   - 蓝牙 A2DP Source → `examples/player/pipeline_a2dp_source_stream`
   - HFP → `examples/player/pipeline_hfp_stream`
   - 音效处理 EQ → `examples/audio_processing/pipeline_equalizer`
   - Downmix → `examples/advanced_examples/downmix_pipeline`
   - OTA → `examples/ota/main`
   - CLI → `examples/cli/main`
   - VoIP → `examples/protocols/voip`

2. 整目录拷贝（保留 `main/`、`CMakeLists.txt`、`sdkconfig.defaults`）。

3. 在拷贝基础上修改：改 `link_tag`、换数据源、加 element、调采样率。

4. 说明拷贝来源与修改点。

**已有工程：** 原地编辑，不要覆盖既有文件，除非用户明确要求。

---

## Failure Strategies

| Situation | Action |
|---|---|
| API 在 resources/ 找不到 | 立即停止，告知用户该 API 可能不存在或属未拉取的子模块 |
| `mp3_decoder.h` / `aac_decoder.h` 找不到 | 检查 `components/esp-adf-libs` 是否递归克隆；运行 `git submodule update --init --recursive` |
| 编译报 ADF 与 IDF 不兼容 | 核对版本矩阵：ADF master ↔ IDF v5.1–v5.5 |
| 播放变速 / 杂音 | 未监听 `AEL_MSG_CMD_REPORT_MUSIC_INFO` 调 `i2s_stream_set_clk` |
| pipeline 卡住不结束 | read 回调未返回 `AEL_IO_DONE`，或未调用 stop 三连 |
| 事件收不到 | peripherals 未 `audio_event_iface_set_listener` 到统一 evt |
| 内存不足 | 减小 `out_rb_size` / ringbuffer；关掉未用的 codec；检查 PSRAM 是否启用 |
| 板子选错无声音 | `idf.py menuconfig` → Audio board 选对；自定义板需实现 `board.c` |

## References

- 场景配方 → `recipes/` 目录
- API 速查 → `resources/api_reference.md`
- 配置参考 → `resources/config_reference.md`
- 陷阱合集 → `resources/pitfalls.md`
- 示例工程索引 → `resources/example_list.md`
- 仓库原始文档 → 仓库 `docs/en/`（framework / streams / codecs / services / peripherals / speech-recognition / abstraction）
- 上手指南 → 仓库 `docs/en/get-started/index.rst`
