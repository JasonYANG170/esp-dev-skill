# ESP-GMF 真实示例清单

> 全部为仓库 `gmf_examples/basic_examples/` 与各组件 `examples/` 下的真实路径。创建项目：
> `idf.py create-project-from-example "espressif/gmf_examples=1.0.0:<name>"`
> （`1.0.0` 为本快照的 `gmf_examples` 版本，按实际拉取版本调整）

## 基础示例（gmf_examples/basic_examples）

| 路径 | 说明 |
|---|---|
| `gmf_examples/basic_examples/pipeline_play_embed_music` | 播放 Flash 内嵌 MP3：embed_flash → aud_dec → 效果链 → codec_dev，GMF 入门模板 |
| `gmf_examples/basic_examples/pipeline_play_sdcard_music` | 播放 SD 卡音乐：io_file → aud_dec → 效果链 → codec_dev |
| `gmf_examples/basic_examples/pipeline_play_http_music` | 播放 HTTP/HTTPS 网络音乐，含 TLS 栈调大、URL 评分选 IO |
| `gmf_examples/basic_examples/pipeline_record_sdcard` | codec_dev 录音 → aud_enc → io_file 写 SD 卡（AAC/AMR/OPUS） |
| `gmf_examples/basic_examples/pipeline_record_audio_muxer` | 录音编码 + aud_muxer 封装容器（TS/MP4/FLV...）到文件 |
| `gmf_examples/basic_examples/pipeline_record_http` | 麦克风录音编码后上传 HTTP 服务器 |
| `gmf_examples/basic_examples/pipeline_audio_effects` | 播放并运行时调整 EQ/ALC/Sonic/Fade/DRC/MBC/Mixer 等效果 |
| `gmf_examples/basic_examples/pipeline_howl` | SD 卡伴奏 + 麦克风啸叫抑制（aud_howl）混音播放 |
| `gmf_examples/basic_examples/pipeline_loop_play_no_gap` | 无缝循环播放：strategy_func + RESET + io_reload |
| `gmf_examples/basic_examples/pipeline_play_multi_source_music` | 多源播放器（HTTP/SD/Flash 切换） |
| `gmf_examples/basic_examples/pipeline_http_download_to_sdcard` | HTTP 文件下载写 SD 卡（整体速率优化） |

## 组件级示例

| 路径 | 说明 |
|---|---|
| `packages/esp_audio_simple_player/test_apps` | esp_audio_simple_player 组件测试 |
| `packages/esp_player/examples/audio_player` | esp_player 音频播放 |
| `packages/esp_player/examples/video_player` | esp_player 视频/音视频播放 |
| `packages/esp_audio_render/examples/audio_render` | 音频渲染（单流 + 8 流混音，per-stream + post-mix ALC）—— 见 `recipes/esp_audio_render.md` |
| `packages/esp_audio_render/examples/simple_piano` | 4 轨钢琴实时合成混音 |
| `packages/esp_capture/examples/audio_capture` | 音频采集（basic / AEC / MP4 muxer / 自定义处理链）—— 见 `recipes/esp_capture.md` |
| `packages/esp_capture/examples/video_capture` | 视频采集（basic / one-shot / overlay / muxer / dual-path）—— 见 `recipes/esp_capture.md` |
| `packages/esp_video_render/examples/video_render` | 单/双 MJPEG、cached/sync、进度条 overlay、LCD 与 LVGL 后端 —— 见 `recipes/esp_video_render.md` |
| `packages/esp_video_render/examples/dual_eyes` | 双目 dual_stream 合成 |
| `packages/esp_video_render/examples/video_player` | 基于 esp_video_render 的视频播放器 |
| `packages/esp_bt_audio/examples/bt_audio` | 蓝牙音频（A2DP/HFP/AVRCP/PBAP + LE Audio TMAP；含串口命令、可选 LVGL UI）—— 见 `recipes/esp_bt_audio.md` |
| `packages/esp_asrc/examples/asrc_demo` | 硬件/软件采样率转换 demo |
| `elements/gmf_ai_audio/examples/aec_rec` | AI 音频 ai_aec 双 pipeline 边播边录 —— 见 `recipes/gmf_ai_audio.md` |
| `elements/gmf_ai_audio/examples/wwe` | ai_afe 唤醒词 + 命令词 + VAD-to-file |

## 测试（单元/集成）

| 路径 | 说明 |
|---|---|
| `gmf_core/test_apps` | GMF-Core 框架单元/集成测试（含 fake_dec/fake_io/general_el 测试夹具） |
| `elements/test_apps` | gmf_audio 各 element 单测（gmf_audio_el_test / play_el_test / effects_test / rec_el_test） |
| `gmf_core/test_apps/main/common/gmf_general_el.h` | 自定义 element 测试参考 |

## 文档模板

| 路径 | 说明 |
|---|---|
| `docs/GMF_TEMPLATE_EXAMPLE_README_EN.md` / `_CN.md` | 新示例 README 模板（贡献新示例时参考） |
