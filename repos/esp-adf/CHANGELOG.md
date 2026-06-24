# Changelog

本文件记录 esp-adf-skill 的发布历史。格式参考 [Keep a Changelog](https://keepachangelog.com/)，版本号遵循 [Semantic Versioning](https://semver.org/)。

## [1.1.0] - 2026-06-18

填充 5 个经审计确认的高价值配方空白（均有真实文档页 + 示例）。所有 API/结构体/宏均核对自仓库 `components/*/include` 与 `docs/en/`。

### Added
- `recipes/speech_recognition.md` — 语音唤醒（WakeNet）+ 命令词（MultiNet）+ VAD + `audio_recorder` 高层录音器。覆盖 `rec_engine_cb` 事件流（`AUDIO_REC_WAKEUP_START`/`VAD_START`/`COMMAND_DECT`）、`recorder_sr_cfg_t` 配置、模型分区要求、底层 `esp_vad` 独立用法、AMR 编码器接入。来源：`components/audio_recorder/include/audio_recorder.h`、`recorder_sr.h`；`docs/en/api-reference/speech-recognition/{audio_recorder,esp_vad,esp_wn_iface}.rst`；`examples/speech_recognition/wwe`（默认 ESP32-S3-Korvo-2 v3）、`examples/speech_recognition/vad`（默认 ESP32-LyraT V4.3）。
- `recipes/playlist.md` — 播放列表管理：`sdcard_scan()` 扫描 + 四种存储后端（`sdcard_list` / `dram_list` / `flash_list` / `partition_list`）+ `playlist_handle_t` 多列表管理 + 热切歌模式（`set_uri → reset_ringbuffer → reset_elements → change_state(INIT) → run`）。来源：`components/playlist/include/{playlist,sdcard_scan,sdcard_list,dram_list,flash_list,partition_list}.h`；`docs/en/api-reference/playlist/index.rst`；`examples/player/pipeline_sdcard_mp3_control`、`examples/cli`。
- `recipes/display_service.md` — 显示服务：`display_service_set_pattern(handle, pattern, value)` 按功能语义点灯，`display_pattern_t` 全量枚举（29 项，WiFi/BT/录音/唤醒/播放/音量/电源/语音），四种 LED 驱动后端（PWM `led_indicator` / AW2013 / IS31x / WS2812）。来源：`components/display_service/include/display_service.h`；`components/display_service/led_bar/include/{led_bar_ws2812,led_bar_aw2013,led_bar_is31x}.h`；`docs/en/api-reference/services/display_service.rst`；`examples/checks/check_display_led`、`examples/display/led_pixels`、`examples/display/music_player`。
- `recipes/voip_sip.md` — VoIP / SIP 网络电话：`av_stream` 双向音频管线（`algorithm_stream` AEC + `raw_stream` RTP 桥接 + G.711 编解码）+ SIP 服务注册 PBX + 按键映射通话。覆盖 FreeSWITCH/Asterisk 配置、`DEBUG_AEC_INPUT` 延迟调优、audio_flash_tone 烧录。来源：`examples/protocols/voip/{main/voip_app.c,main/sip_service.c,main/sip_service.h,README.md}`；`examples/get-started/pipeline_a2dp_sink_and_hfp`；`docs/en/api-reference/streams/index.rst`（algorithm_stream/raw_stream 把 voip 列为参考应用）。
- `recipes/cloud_tts.md` — 云端 TTS：百度 Speech（API Key/Secret Key → access_token + POST 表单）与 AWS Polly（SNTP 校时 + AWS4-HMAC-SHA256 签名），均通过 `http_stream` 的 `event_handle` 回调注入鉴权头/请求体。三家云 TTS 对照。来源：`examples/cloud_services/{pipeline_baidu_speech_mp3,pipeline_aws_polly_mp3,google_translate_device}`；`docs/en/api-reference/streams/index.rst`。

### Changed
- `SKILL.md`：Scenario Quick Reference 表新增上述 5 个配方（分散到「录音/编码」「蓝牙与外设」「音频处理/高级」三组）；`metadata.version` 1.0.0 → 1.1.0。
- `resources/api_reference.md`：Services / Speech 段补充 `audio_recorder`、`display_service`、`playlist`（+ 四后端 + `sdcard_scan`）、led_bar 驱动的真实函数签名。
- `resources/example_list.md`：`speech_recognition/`、`display/`、`checks/`、`cloud_services/`、`protocols/` 分组已含本批配方引用的全部示例路径。

## [1.0.0] - 2026-06-18

首个发布版本。基于 ESP-ADF（master）仓库真实文档与源码构建。

### Added
- `SKILL.md`：技能入口，含 12 条核心原则、When to Use、10 个配方索引、板级支持表、Element 状态机、12 条关键陷阱（每条含 WRONG/CORRECT 代码）、执行工作流与失败策略表。
- `AGENTS.md`：工程约定补充——环境搭建、文件命名、include 模式、标准工程结构、典型 app_main 模式、构建流程、代码生成 checklist、Do-Not-Modify 说明。
- `recipes/`（10 个场景配方）：
  - `play_mp3_flash.md` — Flash 嵌入 MP3 + read 回调播放
  - `play_http_mp3.md` — HTTP/HTTPS 流式播放（含 Wi-Fi）
  - `play_sdcard_fatfs.md` — SD 卡 FatFs 播放
  - `record_to_sdcard.md` — 录音编码 WAV/AMR 写 SD
  - `pipeline_streams.md` — Stream 速查与组合
  - `bt_a2dp_hfp.md` — 蓝牙 A2DP Sink/Source 与 HFP
  - `peripherals_services.md` — 外设集与事件集成
  - `audio_processing.md` — EQ/Downmix/Sonic/Resample/ALC
  - `ota_service.md` — OTA 固件升级
  - `cli_console.md` — CLI 命令行交互
- `resources/api_reference.md`：按模块分组的真实函数签名（framework / streams / codecs / peripherals / board / audio_hal / services / speech）。
- `resources/config_reference.md`：`CONFIG_*_BOARD` 板选符号、Korvo 子芯片、示例级 Kconfig、sdkconfig.defaults 片段、IDF 版本矩阵、关键常量。
- `resources/pitfalls.md`：分类汇总的高频陷阱（生命周期/方向/采样率/事件/codec/蓝牙/构建/内存）。
- `resources/example_list.md`：仓库 `examples/` 下全部真实路径与一句话说明，按分类组织。
- `README.md`：中文技能介绍、特性、安装方式、覆盖范围。
