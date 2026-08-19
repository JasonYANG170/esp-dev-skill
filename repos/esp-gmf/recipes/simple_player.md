# 用 esp_audio_simple_player 快速播放

> **适用摘要**: 用高级包 `esp_audio_simple_player` 以 URI 驱动播放音频（file/http/embed/raw），自动按 scheme 选 IO、按扩展名选解码器，支持同步/异步、事件回调与运行时 pause/stop。

> Version: selected repo/component version; align with the user project and dependency manifest.
> Evidence: `repos/esp-gmf/resources/`, source/examples in `repos/esp-gmf/`, and this recipe path `repos/esp-gmf/recipes/simple_player.md`.
> Validation: draft metadata added from repository routing; verify APIs, Kconfig symbols, and component dependencies against the selected version.

## 触发意图

- "快速做个播放器"
- "esp_audio_simple_player"
- "URI 播放"
- "不想手写 pipeline"
- "simple player"

## 前置条件

| 条件 | 要求 |
|---|---|
| 组件 | `espressif/esp_audio_simple_player`（依赖 `gmf_audio`、`gmf_io`） |
| 参考文档 | `docs/en/gmf-framework/gmf-package/esp-audio-simple-player.rst` |
| 参考头文件 | `packages/esp_audio_simple_player/include/esp_audio_simple_player.h` |

## 分步说明

### 1. 配置：task、输入/输出回调

```c
#include "esp_audio_simple_player.h"

// 输出回调（必须提供）：拿到解码 PCM 写硬件
static int my_out_cb(uint8_t *data, int size, void *ctx) {
    esp_codec_dev_handle_t dev = (esp_codec_dev_handle_t)ctx;
    return esp_codec_dev_write(dev, data, size);
}

// 输入回调：仅 raw:// 数据需要（如外部喂 PCM/裸帧）
static int my_in_cb(uint8_t *data, int size, void *ctx) {
    // 从自定义源填 data，返回实际填充字节数
    return fill_from_source(data, size);
}

esp_asp_cfg_t cfg = {
    .out          = { .cb = my_out_cb, .user_ctx = playback_handle },
    .in           = { .cb = my_in_cb,  .user_ctx = NULL },   // raw:// 才需要
    .task_prio    = 5,
    .task_stack   = 6 * 1024,
    .task_core    = 0,
    .task_stack_in_ext = false,
};

esp_asp_handle_t player = NULL;
esp_audio_simple_player_new(&cfg, &player);
```

### 2. 注册事件回调

事件类型：`ESP_ASP_EVENT_TYPE_STATE`（payload 为 `esp_asp_state_t`）与 `ESP_ASP_EVENT_TYPE_MUSIC_INFO`（payload 为 `esp_asp_music_info_t`）。

```c
static int my_event_cb(esp_asp_event_pkt_t *pkt, void *ctx) {
    if (pkt->type == ESP_ASP_EVENT_TYPE_MUSIC_INFO) {
        esp_asp_music_info_t *m = (esp_asp_music_info_t *)pkt->payload;
        ESP_LOGI(TAG, "info sr=%d ch=%d bits=%d", m->sample_rate, m->channels, m->bits);
        // 据此 open/重配 codec_dev 采样率
    } else if (pkt->type == ESP_ASP_EVENT_TYPE_STATE) {
        esp_asp_state_t st = *(esp_asp_state_t *)pkt->payload;
        if (st == ESP_ASP_STATE_FINISHED || st == ESP_ASP_STATE_ERROR) {
            /* 通知主流程 */
        }
    }
    return ESP_GMF_ERR_OK;
}
esp_audio_simple_player_set_event(player, my_event_cb, NULL);
```

### 3. run（自动按 URI 建 pipeline）

URI 的 scheme 决定 IO（http/file/embed/raw），扩展名决定解码器；对无可靠后缀的流，结合后缀提示与内容探测选解码器。

```c
// 异步播放（非阻塞，靠事件回调拿状态）
esp_audio_simple_player_run(player, "file://sdcard/test.mp3", NULL);

// 或同步阻塞到播完/出错
esp_audio_simple_player_run_to_end(player, "https://dl.espressif.com/dl/audio/gs-16b-2c-44100hz.mp3", NULL);
```

支持的 URI 形式：
- `https://.../track.mp3`、`http://.../stream.aac`
- `embed://tone/0_test.mp3`
- `file://sdcard/test.mp3`
- `raw://...`（需配 `cfg.in` 输入回调，`music_info` 适用于 PCM/无 OGG 头的 Opus）

### 4. 运行时控制

```c
esp_audio_simple_player_pause(player);
esp_audio_simple_player_resume(player);
esp_audio_simple_player_stop(player);

esp_asp_state_t st;
esp_audio_simple_player_get_state(player, &st);
const char *s = esp_audio_simple_player_state_to_str(st);
```

### 5. 销毁

```c
esp_audio_simple_player_destroy(player);
```

`esp_asp_state_t` 取值：`NONE(0) / RUNNING(1) / PAUSED(2) / STOPPED(3) / FINISHED(4) / ERROR(5)`。

> 支持格式：AAC, MP3, AMR, M4A, PCM, WAV, ADPCM, G711, OGG, VORBIS, OPUS, ALAC, FLAC, SBC, LC3, TS。可在 menuconfig 按需只启用实际格式以减小固件。

## 常见错误

| 错误 | 原因 | 解决方法 |
|---|---|---|
| run 返回 `ESP_GMF_ERR_INVALID_URI` | URI 缺 scheme/host/path | 完整 URI，如 `file://sdcard/test.mp3` |
| raw:// 无声 | 没配 `cfg.in` 输入回调 | 提供 `esp_asp_func_t in`，并在 PCM 时传 `music_info` |
| 无法打开网络资源 | Wi-Fi 未连/HTTPS 栈不足 | 先连网；HTTPS 需足够 task 栈与证书 |
| 想改解码格式不动 | menuconfig 未启用对应格式 | 启用 `gmf_loader` 对应解码器 |
| 输出卡顿 | out 回调写硬件阻塞 | 确保 codec_dev 已按 MUSIC_INFO 的采样率 open |

## 参考

- `packages/esp_audio_simple_player/include/esp_audio_simple_player.h`
- `docs/en/gmf-framework/gmf-package/esp-audio-simple-player.rst`
- `packages/esp_audio_simple_player/test_apps/`（组件测试参考）
