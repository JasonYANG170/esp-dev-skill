# 从 SD 卡（FatFs）播放音乐

> **适用摘要**: 用 fatfs_stream 作为 reader 从 SD 卡读音频文件，接解码器与 i2s 输出。SD 卡通过 `audio_board_sdcard_init` 挂载。数据流：SD 卡 → fatfs_stream(reader) → decoder → i2s_stream(writer) → codec。

## 触发意图

- "播放 SD 卡音乐"
- "FatFs 播放"
- "TF 卡 / 内存卡 播放 MP3"
- "pipeline_play_sdcard_music"

## 前置条件

| 条件 | 要求 |
|---|---|
| 参考示例 | `examples/player/pipeline_play_sdcard_music/`、`examples/player/pipeline_sdcard_mp3_control/` |
| 硬件 | 板载 SD 卡槽（如 LyraT），已插入 FAT32 卡 |
| API | `audio_board_sdcard_init(set, SD_MODE_1_LINE)` 挂载 |

## 分步说明

### 1. 挂载 SD 卡（peripherals）

```c
#include "esp_peripherals.h"
#include "periph_sdcard.h"
#include "board.h"

esp_periph_config_t periph_cfg = DEFAULT_ESP_PERIPH_SET_CONFIG();
esp_periph_set_handle_t set = esp_periph_set_init(&periph_cfg);
audio_board_sdcard_init(set, SD_MODE_1_LINE);   // 1 线 SD 模式；部分板用 SD_MODE_4_LINE / SPI
```

> `SD_MODE_*` 宏来自 ESP-IDF SDMMC 接口；`audio_board_sdcard_init` 内部注册 periph_sdcard。

### 2. codec + pipeline

```c
audio_board_handle_t board = audio_board_init();
audio_hal_ctrl_codec(board->audio_hal, AUDIO_HAL_CODEC_MODE_DECODE, AUDIO_HAL_CTRL_START);

audio_pipeline_cfg_t pipeline_cfg = DEFAULT_AUDIO_PIPELINE_CONFIG();
audio_pipeline_handle_t pipeline = audio_pipeline_init(&pipeline_cfg);
```

### 3. 创建 fatfs reader + decoder + i2s writer

```c
#include "fatfs_stream.h"
#include "i2s_stream.h"
#include "mp3_decoder.h"

fatfs_stream_cfg_t fatfs_cfg = FATFS_STREAM_CFG_DEFAULT();
fatfs_cfg.type = AUDIO_STREAM_READER;
audio_element_handle_t fatfs_reader = fatfs_stream_init(&fatfs_cfg);

mp3_decoder_cfg_t mp3_cfg = DEFAULT_MP3_DECODER_CONFIG();
audio_element_handle_t mp3_decoder = mp3_decoder_init(&mp3_cfg);

i2s_stream_cfg_t i2s_cfg = I2S_STREAM_CFG_DEFAULT();
i2s_cfg.type = AUDIO_STREAM_WRITER;
audio_element_handle_t i2s_writer = i2s_stream_init(&i2s_cfg);

audio_pipeline_register(pipeline, fatfs_reader, "file");
audio_pipeline_register(pipeline, mp3_decoder, "mp3");
audio_pipeline_register(pipeline, i2s_writer,  "i2s");
const char *link_tag[3] = {"file", "mp3", "i2s"};
audio_pipeline_link(pipeline, link_tag, 3);
```

### 4. 设置文件 URI 并运行

FatFs 的 URI 形如 `/sdcard/file.mp3`：
```c
audio_element_set_uri(fatfs_reader, "/sdcard/test.mp3");

// 事件监听（同其他播放器）
audio_event_iface_cfg_t evt_cfg = AUDIO_EVENT_IFACE_DEFAULT_CFG();
audio_event_iface_handle_t evt = audio_event_iface_init(&evt_cfg);
audio_pipeline_set_listener(pipeline, evt);
audio_event_iface_set_listener(esp_periph_set_get_event_iface(set), evt);

audio_pipeline_run(pipeline);
// ... event loop: 处理 MUSIC_INFO / FINISHED ...
```

### 5. 切歌（不改 pipeline，换 URI 重 run）

```c
audio_pipeline_stop(pipeline);
audio_pipeline_wait_for_stop(pipeline);
audio_pipeline_reset_ringbuffer(pipeline);
audio_pipeline_reset_elements(pipeline);
audio_element_set_uri(fatfs_reader, "/sdcard/next.mp3");
audio_pipeline_run(pipeline);
```

## 常见错误

| 错误 | 原因 | 解决方法 |
|---|---|---|
| 挂载失败 / 读不到文件 | SD 模式与板子布线不符 | 改 `SD_MODE_4_LINE` 或 SPI 模式；确认卡为 FAT32 |
| 路径错 | URI 不是 `/sdcard/...` 前缀 | 用 `audio_element_set_uri(fatfs_reader, "/sdcard/xxx.mp3")` |
| 切歌卡死 | 直接 set_uri 未先 stop | 必须先 stop + wait + reset 再换 URI 重 run |
| 杂音 | 未按 MUSIC_INFO 重配 I2S | 加 `i2s_stream_set_clk` |
| 分区表报错 | app 分区太小 | 调整 `partitions.csv` |

## 参考项目

- `examples/player/pipeline_play_sdcard_music/` — 单曲播放
- `examples/player/pipeline_sdcard_mp3_control/` — SD 卡多曲 + 按键控制（切歌/暂停/音量）
- `examples/recorder/pipeline_recording_to_sdcard/` — 录音到 SD 的挂载参考
