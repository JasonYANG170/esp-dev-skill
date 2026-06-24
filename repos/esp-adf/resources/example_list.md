# ESP-ADF Example Index

> 全部为仓库 `examples/` 下真实存在的示例路径。每条一行说明（用途）。分类与 README 顺序一致。

## get-started

| 路径 | 说明 |
|---|---|
| `examples/get-started/play_mp3_control` | 入门经典：Flash 嵌入 MP3 + 按键控制（播放/暂停/停止/音量） |
| `examples/get-started/pipeline_tcp_client` | TCP client stream 演示 |
| `examples/get-started/pipeline_a2dp_sink_and_hfp` | A2DP Sink 与 HFP 共存 |

## player（播放）

| 路径 | 说明 |
|---|---|
| `examples/player/pipeline_http_mp3` | HTTP/HTTPS 在线播放 MP3 |
| `examples/player/pipeline_living_stream` | HLS（HTTP Live Streaming）直播流 |
| `examples/player/pipeline_http_select_decoder` | 根据内容自动选择解码器 |
| `examples/player/pipeline_play_sdcard_music` | SD 卡（FatFs）播放 |
| `examples/player/pipeline_sdcard_mp3_control` | SD 卡多曲 + 按键控制 |
| `examples/player/pipeline_spiffs_mp3` | SPIFFS 播放 MP3 |
| `examples/player/pipeline_play_mp3_with_dac_or_pwm` | 内置 DAC / PWM 输出 |
| `examples/player/pipeline_flash_tone` | tone_stream 提示音 |
| `examples/player/pipeline_embed_flash_tone` | embed_flash_stream |
| `examples/player/pipeline_tts_stream` | 文本转语音（依赖 esp-sr） |
| `examples/player/pipeline_a2dp_sink_stream` | 蓝牙 A2DP Sink（蓝牙音箱） |
| `examples/player/pipeline_a2dp_source_stream` | 蓝牙 A2DP Source（蓝牙音源） |
| `examples/player/pipeline_bt_sink` | 经典蓝牙 Sink |
| `examples/player/pipeline_bt_source` | 经典蓝牙 Source |
| `examples/player/pipeline_hfp_stream` | HFP 免提通话 |
| `examples/player/pipeline_loop_playback_without_gap` | 无缝循环播放 |

## recorder（录音）

| 路径 | 说明 |
|---|---|
| `examples/recorder/pipeline_wav_amr_sdcard` | 录 WAV 与 AMR 双 pipeline 写 SD |
| `examples/recorder/pipeline_recording_to_sdcard` | 通用录音到 SD |
| `examples/recorder/element_wav_amr_sdcard` | element 回调直写 |
| `examples/recorder/element_cb_sdcard_amr` | AMR 回调直写 |
| `examples/recorder/pipeline_raw_http` | raw stream 录音上传 HTTP |
| `examples/recorder/av_muxer_sdcard` | 音视频复用录到 SD |

## audio_processing（音频处理）

| 路径 | 说明 |
|---|---|
| `examples/audio_processing/pipeline_equalizer` | 均衡器 EQ |
| `examples/audio_processing/pipeline_resample` | 重采样 |
| `examples/audio_processing/pipeline_sonic` | 变速变调 Sonic |
| `examples/audio_processing/pipeline_alc` | 自动电平控制 ALC |
| `examples/audio_processing/pipeline_audio_forge` | 综合音频处理 |
| `examples/audio_processing/pipeline_passthru` | 透传（直通） |
| `examples/audio_processing/pipeline_spiffs_amr_resample` | SPIFFS AMR + 重采样 |

## advanced_examples（高级）

| 路径 | 说明 |
|---|---|
| `examples/advanced_examples/downmix_pipeline` | Downmix 混音 |
| `examples/advanced_examples/audio_mixer_tone` | 提示音混音 |
| `examples/advanced_examples/flexible_pipeline` | 动态可重构 pipeline |
| `examples/advanced_examples/http_play_and_save_to_file` | 边播边存盘 |
| `examples/advanced_examples/multi-room` | 多房间同步 |
| `examples/advanced_examples/algorithm` | 算法流（AEC/AGC/NS） |
| `examples/advanced_examples/aec` | 回声消除 |
| `examples/advanced_examples/dlna` | DLNA |
| `examples/advanced_examples/nvs_dispatcher` | NVS + dispatcher |
| `examples/advanced_examples/wifi_bt_ble_coex` | Wi-Fi/BT/BLE 共存 |

## protocols（协议）

| 路径 | 说明 |
|---|---|
| `examples/protocols/voip` | VoIP 通话 |
| `examples/protocols/esp-rtc` | ESP-RTC（SIP/RTSP/RTCP） |
| `examples/protocols/esp-rtsp` | RTSP |
| `examples/protocols/rtmp` | RTMP 直播 |

## cli（命令行）

| 路径 | 说明 |
|---|---|
| `examples/cli` | esp_console 命令行交互（含 BLE GATS 触发） |

## ota

| 路径 | 说明 |
|---|---|
| `examples/ota` | OTA 固件升级服务 |

## cloud_services / dueros / ai_agent

| 路径 | 说明 |
|---|---|
| `examples/cloud_services/google_translate_device` | Google 翻译 |
| `examples/cloud_services/pipeline_aws_polly_mp3` | AWS Polly TTS |
| `examples/cloud_services/pipeline_baidu_speech_mp3` | 百度语音合成 |
| `examples/dueros/main`、`examples/dueros/spiffs` | DuerOS 服务 |
| `examples/ai_agent/coze_ws_app` | Coze WebSocket 应用 |
| `examples/ai_agent/volc_rtc` | 火山 RTC |

## speech_recognition（语音识别）

| 路径 | 说明 |
|---|---|
| `examples/speech_recognition/vad` | 语音活动检测 |
| `examples/speech_recognition/wwe` | 唤醒词（WakeWord Engine） |

## system（系统）

| 路径 | 说明 |
|---|---|
| `examples/system/battery` | 电池服务 |
| `examples/system/coredump` | Core dump 上传 |
| `examples/system/power_save` | 省电 |
| `examples/system/wpa2_enterprise` | WPA2 企业级 Wi-Fi |

## display（显示）

| 路径 | 说明 |
|---|---|
| `examples/display/music_player` | 音乐播放器 UI |
| `examples/display/lcd_jpeg` | LCD 显示 JPEG |
| `examples/display/lcd_camera` | LCD + 摄像头 |
| `examples/display/dual_eyes` | 双眼动画 |
| `examples/display/led_pixels` | LED 像素灯 |

## checks（自检）

| 路径 | 说明 |
|---|---|
| `examples/checks/check_board_buttons` | 板载按键自检 |
| `examples/checks/check_display_led` | 显示/LED 自检 |

## 板级专属

| 路径 | 说明 |
|---|---|
| `examples/korvo_du1906/main` | Korvo-DU1906 专属综合示例（含固件/profiles/tone） |

> 完整的「示例 × 开发板」兼容矩阵见仓库 `examples/README.md`。
