# 通过 HTTP/HTTPS 流式播放 MP3

> **适用摘要**: 用 http_stream 从网络读 MP3，经 mp3_decoder 解码，i2s_stream 输出到 codec。包含 Wi-Fi 连接（periph_wifi）与事件监听。数据流：HTTP server → http_stream → mp3_decoder → i2s_stream → codec。

> Version: selected repo/component version; align with the user project and dependency manifest.
> Evidence: `repos/esp-adf/resources/`, source/examples in `repos/esp-adf/`, and this recipe path `repos/esp-adf/recipes/play_http_mp3.md`.
> Validation: draft metadata added from repository routing; verify APIs, Kconfig symbols, and component dependencies against the selected version.

## 触发意图

- "播放网络 MP3"
- "HTTP 流播放"
- "网络收音机 / 在线音乐"
- "HLS / HTTPS 音频"
- "pipeline_http_mp3 那个例子"

## 前置条件

| 条件 | 要求 |
|---|---|
| 参考示例 | `examples/player/pipeline_http_mp3/main/play_http_mp3_example.c` |
| 网络 | Wi-Fi 已配置（`CONFIG_WIFI_SSID` / `CONFIG_WIFI_PASSWORD`） |
| 板子 | 任一受支持的 `CONFIG_*_BOARD` |
| 子模块 | `esp-adf-libs` 已递归克隆 |

## 分步说明

### 1. NVS / Netif 初始化（HTTP + Wi-Fi 必备）

```c
#include "nvs_flash.h"
#include "esp_netif.h"

ESP_ERROR_CHECK(nvs_flash_init());
ESP_ERROR_CHECK(esp_netif_init());
```

### 2. codec + pipeline + 三元素

```c
#include "audio_element.h"
#include "audio_pipeline.h"
#include "audio_event_iface.h"
#include "audio_mem.h"
#include "audio_common.h"
#include "http_stream.h"
#include "i2s_stream.h"
#include "mp3_decoder.h"
#include "esp_peripherals.h"
#include "periph_wifi.h"
#include "board.h"

audio_board_handle_t board = audio_board_init();
audio_hal_ctrl_codec(board->audio_hal, AUDIO_HAL_CODEC_MODE_DECODE, AUDIO_HAL_CTRL_START);

audio_pipeline_cfg_t pipeline_cfg = DEFAULT_AUDIO_PIPELINE_CONFIG();
audio_pipeline_handle_t pipeline = audio_pipeline_init(&pipeline_cfg);

http_stream_cfg_t http_cfg = HTTP_STREAM_CFG_DEFAULT();
audio_element_handle_t http_reader = http_stream_init(&http_cfg);

i2s_stream_cfg_t i2s_cfg = I2S_STREAM_CFG_DEFAULT();
i2s_cfg.type = AUDIO_STREAM_WRITER;
audio_element_handle_t i2s_writer = i2s_stream_init(&i2s_cfg);

mp3_decoder_cfg_t mp3_cfg = DEFAULT_MP3_DECODER_CONFIG();
audio_element_handle_t mp3_decoder = mp3_decoder_init(&mp3_cfg);

audio_pipeline_register(pipeline, http_reader,  "http");
audio_pipeline_register(pipeline, mp3_decoder, "mp3");
audio_pipeline_register(pipeline, i2s_writer,  "i2s");
const char *link_tag[3] = {"http", "mp3", "i2s"};
audio_pipeline_link(pipeline, link_tag, 3);

// 设置数据源 URI（支持 http/https）
audio_element_set_uri(http_reader, "https://dl.espressif.com/dl/audio/ff-16b-2c-44100hz.mp3");
```

### 3. 外设：Wi-Fi 连接（必须先连上再 run）

```c
esp_periph_config_t periph_cfg = DEFAULT_ESP_PERIPH_SET_CONFIG();
esp_periph_set_handle_t set = esp_periph_set_init(&periph_cfg);

periph_wifi_cfg_t wifi_cfg = {
    .wifi_config.sta.ssid     = CONFIG_WIFI_SSID,
    .wifi_config.sta.password = CONFIG_WIFI_PASSWORD,
};
esp_periph_handle_t wifi_handle = periph_wifi_init(&wifi_cfg);
esp_periph_start(set, wifi_handle);
periph_wifi_wait_for_connected(wifi_handle, portMAX_DELAY);
```

### 4. 事件监听 + 启动 + 循环

```c
audio_event_iface_cfg_t evt_cfg = AUDIO_EVENT_IFACE_DEFAULT_CFG();
audio_event_iface_handle_t evt = audio_event_iface_init(&evt_cfg);
audio_pipeline_set_listener(pipeline, evt);
audio_event_iface_set_listener(esp_periph_set_get_event_iface(set), evt);  // peripherals 也要挂

audio_pipeline_run(pipeline);

while (1) {
    audio_event_iface_msg_t msg;
    if (audio_event_iface_listen(evt, &msg, portMAX_DELAY) != ESP_OK) continue;

    if (msg.source == (void *)mp3_decoder
        && msg.cmd == AEL_MSG_CMD_REPORT_MUSIC_INFO) {
        audio_element_info_t info = {0};
        audio_element_getinfo(mp3_decoder, &info);
        i2s_stream_set_clk(i2s_writer, info.sample_rates, info.bits, info.channels);
    }
    if (msg.source == (void *)i2s_writer
        && msg.cmd == AEL_MSG_CMD_REPORT_STATUS
        && (int)msg.data == AEL_STATUS_STATE_FINISHED) {
        break;
    }
}
```

### 5. HTTPS 自定义证书（可选）

```c
esp_err_t http_stream_set_server_cert(http_reader, "-----BEGIN CERTIFICATE-----\n...\n-----END CERTIFICATE-----\n");
```

## 常见错误

| 错误 | 原因 | 解决方法 |
|---|---|---|
| 连不上 / 一直 buffering | Wi-Fi 未连上就 run | `periph_wifi_wait_for_connected` 后再 `audio_pipeline_run` |
| HTTPS 握手失败 | 未设服务器证书或时间未同步 | `http_stream_set_server_cert`，或对受限证书同步 SNTP |
| 收不到事件 | 只 set_listener 了 pipeline | peripherals 事件接口也要 `audio_event_iface_set_listener` |
| 内存吃紧 | ringbuffer / TLS 缓冲大 | 调小 `http_cfg.out_rb_size`，启用 PSRAM |
| HLS 不播放 | `esp-adf-libs/audio_stream/lib/hls` 未启用 | 确认子模块存在，HLS 由 http_stream 内部支持 |

## 参考项目

- `examples/player/pipeline_http_mp3/main/play_http_mp3_example.c` — 标准在线 MP3 播放
- `examples/player/pipeline_living_stream/` — HTTP Live Streaming (HLS) 播放
- `examples/player/pipeline_http_select_decoder/` — 根据内容自动选择解码器
