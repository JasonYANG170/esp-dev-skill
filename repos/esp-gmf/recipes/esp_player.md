# 用 esp_player 做音视频同步播放

> **适用摘要**: 用 `esp_player` 单实例串联解封装、解码、渲染，支持本地文件、HTTP(S)、HLS、外部帧（fill/block）输入，含 A/V 同步、seek、变速、多轨道选择。

## 触发意图

- "音视频播放器"
- "播放 mp4/ts/m3u8"
- "esp_player"
- "A/V 同步"
- "seek/变速"

## 前置条件

| 条件 | 要求 |
|---|---|
| 组件 | `espressif/esp_player`（依赖 render 句柄） |
| 渲染句柄 | 音频：`esp_audio_render_stream_handle_t`（来自 `esp_audio_render`）；视频：`esp_video_render_handle_t`（来自 `esp_video_render`） |
| 参考头文件 | `packages/esp_player/include/esp_player.h`、`esp_player_types.h` |
| 参考示例 | `packages/esp_player/examples/audio_player`、`video_player` |

## 分步说明

### 1. 初始化（提供 render 句柄）

```c
#include "esp_player.h"

esp_player_config_t cfg = ESP_PLAYER_CONFIG_DEFAULT();
cfg.audio_render_hd = audio_render_handle;   // 来自 esp_audio_render_stream_get()
cfg.video_render_hd = video_render_handle;   // 来自 esp_video_render_create()；纯音频可设 NULL

esp_player_handle_t player = NULL;
esp_player_init(&cfg, &player);
```

> 对通过 `esp_player_set_av_mask` / `data_src.av_mask` 启用的每条路径，必须提供对应 render 句柄；`ESP_PLAYER_MASK_AUDIO` 需 audio_render_hd，`ESP_PLAYER_MASK_VIDEO` 需 video_render_hd，`AV` 两者都需。

### 2. 注册事件回调（或事件队列）

```c
// 回调方式（在内部线程触发，回调内勿调带 player 锁的 API）
esp_player_set_event_cb(player, my_event_cb, ctx);

// 或队列方式（拷贝 esp_player_event_msg_t 到 FreeRTOS 队列）
QueueHandle_t q = xQueueCreate(8, sizeof(esp_player_event_msg_t));
esp_player_set_event_queue(player, q);
```

典型事件：`ESP_PLAYER_EVENT_PLAYED`、`ESP_PLAYER_EVENT_PAUSED`、`ESP_PLAYER_EVENT_FINISHED`、`ESP_PLAYER_EVENT_ERROR`、`ESP_PLAYER_EVENT_SEEK_DONE`、`ESP_PLAYER_EVENT_STOPPED`。

### 3. 设置数据源（mask + URL + 同步模式可一次设好）

```c
// 音视频同步播放 mp4
esp_player_data_src_t src = {
    .av_mask    = ESP_PLAYER_MASK_AV,
    .url        = "file:///sdcard/movie.mp4",
    .sync_mode  = ESP_PLAYER_SYNC_MODE_AUDIO,   // 视频跟随音频（默认）
};
esp_player_set_data_src(player, &src);

// 纯音频
esp_player_set_av_mask(player, ESP_PLAYER_MASK_AUDIO);
esp_player_set_url(player, "https://example.com/track.mp3");

// HLS：路径以 .m3u8 结尾自动识别
esp_player_set_url(player, "http://example.com/live/playlist.m3u8");
```

URL 支持的 scheme：`file`、`http/https`、`fill`/`block`（外部帧，`fill:///name.codec`，run 后用 `esp_player_submit_frame` 推帧）。本地 PCM 需在 query 带 `sr/ch/bits`。

### 4. 播放控制

```c
esp_player_run(player);                 // 非阻塞，靠事件拿状态
// esp_player_run_to_end(player);       // 阻塞到结束/出错（勿在回调内调）

esp_player_pause(player);
esp_player_resume(player);
esp_player_stop(player);                // 已停止也安全，返回 OK

esp_player_seek(player, 30000);         // 跳到 30 s（ms）；PLAYING/PAUSED 真正 seek
esp_player_set_speed(player, 1.5f);     // 变速，>0
```

### 5. 多轨道选择（容器播放）

```c
uint16_t num = 0;
esp_player_get_track_num(player, ESP_PLAYER_TRACK_TYPE_AUDIO, &num);
esp_player_track_info_t info;
for (uint16_t i = 0; i < num; i++) {
    esp_player_get_track_info(player, ESP_PLAYER_TRACK_TYPE_AUDIO, i, &info);
}
esp_player_enable_track(player, ESP_PLAYER_TRACK_TYPE_AUDIO, 0, true);   // 选音轨 0
```

### 6. 查询时长与进度

```c
uint64_t dur = 0, pos = 0;
esp_player_get_duration(player, &dur);       // 总时长 ms（不支持 fill/block）
esp_player_get_play_time(player, &pos);      // 当前 PTS ms
```

### 7. 销毁

```c
esp_player_deinit(player);   // 先停播再释放
```

> 支持容器：WAV, MP4, M4A, TS, OGG, AVI, FLV, CAF；裸 ES 流：`.mp3/.aac/.flac/.amr`。音频解码：AAC/MP3/Vorbis/Opus/FLAC/AMR-NB·WB/G.711 A·μ-law/ALAC/ADPCM/SBC/LC3；视频解码：H.264, MJPEG。同步模式：AUDIO/VIDEO/SYSTEM/NONE。

## 常见错误

| 错误 | 原因 | 解决方法 |
|---|---|---|
| run 返回 INVALID_STATE | 不在 IDLE/STOPPED/FINISHED | 先 stop 或等结束再 run |
| 视频/音频无输出 | av_mask 启用了但未给对应 render_hd | 为每条启用路径提供 render 句柄 |
| fill/block 无法 seek | 虚拟 URL 不支持 seek | 外部帧模式只能顺序推送 |
| 回调里调 player API 卡住 | 回调在内部线程、带锁 | 回调内只置标志，控制 API 在其他任务调 |
| 多音轨选不到 | 非 AV extractor 模式 | enable_track 仅 AV + extractor 模式可用 |

## 参考

- `packages/esp_player/include/esp_player.h`、`esp_player_advance.h`、`esp_player_types.h`
- `docs/en/gmf-framework/gmf-package/esp-player.rst`
- `packages/esp_player/examples/audio_player/`、`video_player/`
