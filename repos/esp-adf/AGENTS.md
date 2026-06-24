# AGENTS.md — Supplementary Agent Guide

> 核心原则、配方索引、状态机、陷阱与执行工作流均在 `SKILL.md`。
> 本文件**只**补充 `SKILL.md` 未覆盖的工程约定与工具链指引，不重复内容。

## Project Context

**Language**: C · **Target**: ESP32 / ESP32-S2 / ESP32-S3 / ESP32-C3 / ESP32-C5 / ESP32-C6 / ESP32-P4
**Framework**: ESP-ADF（master）+ ESP-IDF（release/v5.1 ~ v5.5）· **Build**: CMake + `idf.py`（Ninja）

## Environment Setup（每次新会话）

```bash
# Linux / macOS
. $ADF_PATH/export.sh          # ADF 仓库根目录，同时导出 ADF_PATH / IDF_PATH
# Windows (cmd)
%ADF_PATH%\export.bat
# Windows (PowerShell)
. $ADF_PATH\export.ps1
```

`ADF_PATH` 必须指向 ESP-ADF 仓库根。`export.*` 会同时 source ESP-IDF 的环境。

## Code Generation Conventions

### File Naming
- 应用入口：`<app_name>.c`，含 `app_main(void)`（ESP-IDF 入口约定）
- 示例工程统一：`main/<example_name>.c`（旧示例带 `_example` 后缀，如 `play_mp3_control_example.c`）
- 自定义板：`components/<my_board>/board.c` + `board.h`，并实现 `audio_board_init / audio_board_key_init / audio_board_sdcard_init`

### Include Pattern（播放器典型）
```c
#include "esp_log.h"
#include "audio_element.h"
#include "audio_pipeline.h"
#include "audio_event_iface.h"
#include "audio_mem.h"
#include "audio_common.h"
#include "i2s_stream.h"
#include "mp3_decoder.h"      // 或 aac_decoder.h / flac_decoder.h ...
#include "http_stream.h"       // 数据源按需：fatfs_stream.h / spiffs_stream.h / raw_stream.h
#include "esp_peripherals.h"
#include "periph_wifi.h"       // 按需：periph_touch.h / periph_button.h / periph_adc_button.h / periph_sdcard.h
#include "board.h"             // 由 CONFIG_*_BOARD 决定的板级头
```

### Standard Project Structure
```
my_audio_app/
├── CMakeLists.txt                # 顶层：include($ENV{ADF_PATH}/CMakeLists.txt) + include($ENV{IDF_PATH}/tools/cmake/project.cmake)
├── sdkconfig.defaults            # 默认 CONFIG_*_BOARD / WIFI 等
├── partitions.csv                # 分区表（音频/录音/OTA 必备，通常需较大 app 分区）
└── main/
    ├── CMakeLists.txt            # idf_component_register(SRCS ... INCLUDE_DIRS .)
    └── my_audio_app.c            # app_main(void)
```

顶层 `CMakeLists.txt` 必须含（来自 get-started 示例）：
```cmake
include($ENV{ADF_PATH}/CMakeLists.txt)
include($ENV{IDF_PATH}/tools/cmake/project.cmake)
project(my_audio_app)
```

### Canonical Playback Pattern（app_main）
```c
void app_main(void)
{
    // 1. codec
    audio_board_handle_t board = audio_board_init();
    audio_hal_ctrl_codec(board->audio_hal, AUDIO_HAL_CODEC_MODE_DECODE, AUDIO_HAL_CTRL_START);

    // 2. pipeline + elements
    audio_pipeline_cfg_t pipeline_cfg = DEFAULT_AUDIO_PIPELINE_CONFIG();
    pipeline = audio_pipeline_init(&pipeline_cfg);

    http_stream_cfg_t http_cfg = HTTP_STREAM_CFG_DEFAULT();
    http_stream_reader = http_stream_init(&http_cfg);

    i2s_stream_cfg_t i2s_cfg = I2S_STREAM_CFG_DEFAULT();
    i2s_cfg.type = AUDIO_STREAM_WRITER;
    i2s_stream_writer = i2s_stream_init(&i2s_cfg);

    mp3_decoder_cfg_t mp3_cfg = DEFAULT_MP3_DECODER_CONFIG();
    mp3_decoder = mp3_decoder_init(&mp3_cfg);

    // 3. register + link
    audio_pipeline_register(pipeline, http_stream_reader, "http");
    audio_pipeline_register(pipeline, mp3_decoder,        "mp3");
    audio_pipeline_register(pipeline, i2s_stream_writer,  "i2s");
    const char *link_tag[3] = {"http", "mp3", "i2s"};
    audio_pipeline_link(pipeline, link_tag, 3);

    // 4. set data source URI
    audio_element_set_uri(http_stream_reader, "https://dl.espressif.com/dl/audio/ff-16b-2c-44100hz.mp3");

    // 5. peripherals + events
    esp_periph_config_t periph_cfg = DEFAULT_ESP_PERIPH_SET_CONFIG();
    esp_periph_set_handle_t set = esp_periph_set_init(&periph_cfg);
    // ... periph_wifi_init / esp_periph_start / wait_for_connected ...

    audio_event_iface_cfg_t evt_cfg = AUDIO_EVENT_IFACE_DEFAULT_CFG();
    audio_event_iface_handle_t evt = audio_event_iface_init(&evt_cfg);
    audio_pipeline_set_listener(pipeline, evt);
    audio_event_iface_set_listener(esp_periph_set_get_event_iface(set), evt);

    // 6. run + event loop
    audio_pipeline_run(pipeline);
    while (1) {
        audio_event_iface_msg_t msg;
        if (audio_event_iface_listen(evt, &msg, portMAX_DELAY) != ESP_OK) continue;
        // handle msg.cmd / msg.source_type ...
    }
}
```

### Debug Output Convention
```c
#include "esp_log.h"
static const char *TAG = "MY_APP";
esp_log_level_set("*", ESP_LOG_WARN);
esp_log_level_set(TAG, ESP_LOG_INFO);
ESP_LOGI(TAG, "sample_rates=%d", sr);   // 串口 115200 默认波特
```

## Build Workflow

1. 设目标芯片：`idf.py set-target esp32s3`
2. 配板与参数：`idf.py menuconfig` → “Audio board” 选板；按需设 Wi-Fi SSID/密码
3. 编译：`idf.py build`
4. 烧录 + 监控：`idf.py -p PORT flash monitor`（PORT 如 `/dev/ttyUSB0` 或 `COM3`；默认烧录波特 460800）
5. 退出监控：`Ctrl + ]`

> ESP-IDF 构建系统不支持路径含空格。`ADF_PATH` 与工程目录都不能有空格。

## Codegen Checklist

- [ ] 顶层 `CMakeLists.txt` 含 `include($ENV{ADF_PATH}/CMakeLists.txt)`
- [ ] `main/CMakeLists.txt` 用 `idf_component_register(...)` 注册源文件
- [ ] `idf.py set-target <chip>` 已执行（sdkconfig 不残留旧目标）
- [ ] `CONFIG_*_BOARD` 与实际硬件一致
- [ ] element 顺序：input stream → codec/filter → output stream
- [ ] 每个 element 都 `audio_pipeline_register` 且 tag 唯一
- [ ] `link_tag` 顺序与数据流方向一致
- [ ] 输出 stream 的 `type` = `AUDIO_STREAM_WRITER`；录音输入 = `AUDIO_STREAM_READER`
- [ ] 监听 `AEL_MSG_CMD_REPORT_MUSIC_INFO` 并 `i2s_stream_set_clk`
- [ ] HTTP 场景：Wi-Fi 已连接后再 `audio_pipeline_run`
- [ ] 停止走 `stop → wait_for_stop → terminate`；销毁先 `remove_listener` 再 `event_iface_destroy`
- [ ] 录音 / OTA / 大固件：分区表 `partitions.csv` 足够大
- [ ] `esp-adf-libs`、`esp-sr` 子模块已递归拉取

## Do Not Modify

- ESP-ADF 仓库源码（`components/`、`docs/`、`examples/`）— 仅作参考与拷贝起点，不要原地改仓库文件
- `SKILL.md` 的 frontmatter（技能元数据）
- `resources/` 中的速查文档为只读参考；如需补充请基于真实仓库证据更新
