# 从 Flash 内存播放 MP3

> **适用摘要**: 将 MP3 数据以二进制嵌入 Flash，通过自定义 read 回调喂给 mp3_decoder，再经 i2s_stream 写到 codec 芯片播放。是 ESP-ADF 最经典的入门 pipeline（Flash → mp3_decoder → i2s）。

> Version: selected repo/component version; align with the user project and dependency manifest.
> Evidence: `repos/esp-adf/resources/`, source/examples in `repos/esp-adf/`, and this recipe path `repos/esp-adf/recipes/play_mp3_flash.md`.
> Validation: draft metadata added from repository routing; verify APIs, Kconfig symbols, and component dependencies against the selected version.

## 触发意图

- "播放 Flash 里的 MP3"
- "内存播放 MP3"
- "嵌入音频"
- "play_mp3_control 那个例子"
- "怎么用 read 回调喂数据"

## 前置条件

| 条件 | 要求 |
|---|---|
| 参考示例 | `examples/get-started/play_mp3_control/main/play_mp3_control_example.c` |
| 板子 | 任一受支持的 `CONFIG_*_BOARD`（如 LyraT V4.3） |
| 子模块 | `components/esp-adf-libs` 已递归克隆（提供 `mp3_decoder.h`） |

## 分步说明

### 1. 把 MP3 嵌入固件（EMBED_FILES）

`main/CMakeLists.txt`：
```cmake
idf_component_register(SRCS "play_mp3_control_example.c"
                       INCLUDE_DIRS "."
                       EMBED_FILES  music-16b-2c-44100hz.mp3)
```

C 源码中用 `asm` 符号引用嵌入数据的起止：
```c
extern const uint8_t mp3_start[] asm("_binary_music_16b_2c_44100hz_mp3_start");
extern const uint8_t mp3_end[]   asm("_binary_music_16b_2c_44100hz_mp3_end");
```

### 2. 实现 read 回调（返回 AEL_IO_DONE 表示结束）

```c
static struct {
    const uint8_t *start;
    const uint8_t *end;
    int pos;
} file;

int mp3_read_cb(audio_element_handle_t el, char *buf, int len, TickType_t wait, void *ctx)
{
    int remaining = file.end - file.start - file.pos;
    if (remaining == 0) {
        return AEL_IO_DONE;            // 数据读完，pipeline 收到后会进入 FINISHED
    }
    int n = (len < remaining) ? len : remaining;
    memcpy(buf, file.start + file.pos, n);
    file.pos += n;
    return n;                            // 必须返回实际读取字节数
}
```

### 3. 初始化 codec 与 pipeline

```c
#include "audio_element.h"
#include "audio_pipeline.h"
#include "audio_event_iface.h"
#include "audio_mem.h"
#include "audio_common.h"
#include "i2s_stream.h"
#include "mp3_decoder.h"
#include "board.h"

audio_pipeline_handle_t pipeline;
audio_element_handle_t mp3_decoder, i2s_writer;

// codec
audio_board_handle_t board = audio_board_init();
audio_hal_ctrl_codec(board->audio_hal, AUDIO_HAL_CODEC_MODE_DECODE, AUDIO_HAL_CTRL_START);

// pipeline
audio_pipeline_cfg_t pipeline_cfg = DEFAULT_AUDIO_PIPELINE_CONFIG();
pipeline = audio_pipeline_init(&pipeline_cfg);
mem_assert(pipeline);
```

### 4. 创建 element、register、link

```c
mp3_decoder_cfg_t mp3_cfg = DEFAULT_MP3_DECODER_CONFIG();
mp3_decoder = mp3_decoder_init(&mp3_cfg);
audio_element_set_read_cb(mp3_decoder, mp3_read_cb, NULL);   // 注入自定义数据源

i2s_stream_cfg_t i2s_cfg = I2S_STREAM_CFG_DEFAULT();
i2s_cfg.type = AUDIO_STREAM_WRITER;
i2s_writer = i2s_stream_init(&i2s_cfg);

audio_pipeline_register(pipeline, mp3_decoder, "mp3");
audio_pipeline_register(pipeline, i2s_writer,  "i2s");
const char *link_tag[2] = {"mp3", "i2s"};
audio_pipeline_link(pipeline, link_tag, 2);
```

��据流：`[mp3_read_cb] → mp3_decoder → i2s_stream → [codec_chip]`

### 5. 事件监听 + 启动 + 主循环

```c
file.start = mp3_start; file.end = mp3_end; file.pos = 0;

audio_event_iface_cfg_t evt_cfg = AUDIO_EVENT_IFACE_DEFAULT_CFG();
audio_event_iface_handle_t evt = audio_event_iface_init(&evt_cfg);
audio_pipeline_set_listener(pipeline, evt);

audio_pipeline_run(pipeline);

while (1) {
    audio_event_iface_msg_t msg;
    if (audio_event_iface_listen(evt, &msg, portMAX_DELAY) != ESP_OK) continue;

    // 解码器上报真实采样率 → 重配 I2S
    if (msg.source_type == AUDIO_ELEMENT_TYPE_ELEMENT
        && msg.source == (void *)mp3_decoder
        && msg.cmd == AEL_MSG_CMD_REPORT_MUSIC_INFO) {
        audio_element_info_t info = {0};
        audio_element_getinfo(mp3_decoder, &info);
        i2s_stream_set_clk(i2s_writer, info.sample_rates, info.bits, info.channels);
    }
    // i2s 上报 FINISHED → 退出
    if (msg.source_type == AUDIO_ELEMENT_TYPE_ELEMENT
        && msg.source == (void *)i2s_writer
        && msg.cmd == AEL_MSG_CMD_REPORT_STATUS
        && (int)msg.data == AEL_STATUS_STATE_FINISHED) {
        break;
    }
}
```

### 6. 停止与清理（顺序不能错）

```c
audio_pipeline_stop(pipeline);
audio_pipeline_wait_for_stop(pipeline);
audio_pipeline_terminate(pipeline);

audio_pipeline_remove_listener(pipeline);
audio_event_iface_destroy(evt);

audio_pipeline_deinit(pipeline);
audio_element_deinit(i2s_writer);
audio_element_deinit(mp3_decoder);
```

## 常见错误

| 错误 | 原因 | 解决方法 |
|---|---|---|
| 找不到 `mp3_decoder.h` | `esp-adf-libs` 子模块未拉取 | `git submodule update --init --recursive` |
| 播放变速/变调 | 未在 MUSIC_INFO 事件后 `i2s_stream_set_clk` | 加事件处理，按 info 重配 I2S |
| 只播一次就卡住 | read 回调读完返回 0 而非 `AEL_IO_DONE` | 无数据时 `return AEL_IO_DONE;` |
| pipeline 永不停止 | 没监听 FINISHED / 没调 stop 三连 | 加退出条件 + stop/wait/terminate |
| EMBED 后链接报错 | `asm` 符号名与文件名不匹配 | 符号名 = `_binary_` + 文件名（`.`/`-` 转 `_`）+ `_start/_end` |

## 参考项目

- `examples/get-started/play_mp3_control/main/play_mp3_control_example.c` — 完整的 Flash MP3 + 按键控制（播放/暂停/停止/音量）
- `examples/player/pipeline_play_mp3_with_dac_or_pwm/main/` — 用内置 DAC / PWM 输出
