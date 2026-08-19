# 用 esp_capture 做高级音视频采集

> **适用摘要**: 用高级包 `esp_capture` 按「source → path → sink」模型从麦克风/摄像头采集音视频，自动按 source/target 格式协商编码与转码链，支持 AEC 采集、多 sink 并行（一路流式拉帧 + 一路 MP4 本地存储）、overlay 文字叠加、单帧抓拍与自定义处理流水线。与 `pipeline_record.md`（GMF-Core 手写的 `codec_dev→aud_enc→io_file` 单链）不同，`esp_capture` 封装了整套协商与多路分发逻辑。

> Version: selected repo/component version; align with the user project and dependency manifest.
> Evidence: `repos/esp-gmf/resources/`, source/examples in `repos/esp-gmf/`, and this recipe path `repos/esp-gmf/recipes/esp_capture.md`.
> Validation: draft metadata added from repository routing; verify APIs, Kconfig symbols, and component dependencies against the selected version.

## 触发意图

- "音视频采集录制"
- "esp_capture 怎么用"
- "RTMP/WebRTC 推流采集"
- "麦克风 + 摄像头同步录制"
- "MP4 切片本地存储"
- "采集时叠加文字水印"
- "AEC 回声消除采集"

## 前置条件

| 条件 | 要求 |
|---|---|
| 硬件 | 含 ADC 麦克风的音频板（AEC 需 ESP32-S3/S3-1/P4）；视频采集需 V4L2/DVP 摄像头板（P4-Function-EV / S3-Korvo2） |
| 组件 | `espressif/esp_capture`（依赖 `gmf-audio`/`gmf-video`、`esp_muxer`、`esp_audio_codec`、`esp_video_codec`、`esp_board_manager`） |
| menuconfig | 按目标格式启用编码器（AAC/MJPEG/H264...）；MP4 存储需 `mp4_muxer_register()` |
| 参考头文件 | `packages/esp_capture/include/esp_capture.h`、`esp_capture_sink.h`、`esp_capture_advance.h`、`esp_capture_types.h`、`esp_capture_defaults.h` |
| 参考示例 | `packages/esp_capture/examples/audio_capture`、`video_capture` |

## 分步说明

`esp_capture` 把采集抽象为三个角色：**source**（输入设备，audio/video src_if）、**path**（内部自动协商出的编码/格式转换流水线）、**sink**（每路输出，可各自挂 muxer/overlay、各自独立 enable）。多个 sink 共享同一 source，适合「一路流式拉帧给网络 + 一路 MP4 落盘」。

### 1. 注册默认编码器与 muxer，设线程调度

```c
#include "esp_capture.h"
#include "esp_capture_defaults.h"
#include "esp_audio_codec.h"   // esp_audio_enc_register_default
#include "mp4_muxer.h"          // mp4_muxer_register（MP4 存储才需要）

// 在 app_main 早期注册默认 codec/muxer（capture 内部按 FourCC 自动选用）
esp_audio_enc_register_default();
mp4_muxer_register();

// 可选：统一调度所有内部线程的栈/优先级/核/PSRAM（必须在 esp_capture_start 之前）
static void sched_cb(const char *name, esp_capture_thread_schedule_cfg_t *cfg) {
    cfg->stack_size = 8 * 1024;
    cfg->priority    = 5;
    cfg->core_id     = 0;
    cfg->stack_in_ext = true;   // 放 PSRAM
}
esp_capture_set_thread_scheduler(sched_cb);
```

> `esp_capture_set_thread_scheduler` 当前只支持静态调度（启动前一次性应用）；要分析耗时可用 `esp_gmf_oal_sys_get_real_time_stats()`。

### 2. 创建 source（audio dev / audio AEC / video V4L2）

source 由 `esp_capture_defaults.h` 提供的工厂函数创建，`record_handle`/`dev_path` 来自板级 `esp_board_manager`。

```c
#include "esp_board_device.h"
#include "esp_board_manager_defs.h"
#include "dev_audio_codec.h"
#include "dev_camera.h"   // video 才需要

// --- 音频 source（普通）---
dev_audio_codec_handles_t *codec_handle = NULL;
esp_board_device_get_handle(ESP_BOARD_DEVICE_NAME_AUDIO_ADC, (void **)&codec_handle);

esp_capture_audio_dev_src_cfg_t codec_cfg = {
    .record_handle = codec_handle->codec_dev,
};
esp_capture_audio_src_if_t *aud_src = esp_capture_new_audio_dev_src(&codec_cfg);

// --- 音频 source（带 AEC，S3/S3-1/P4 专用）---
#if CONFIG_IDF_TARGET_ESP32S3 || CONFIG_IDF_TARGET_ESP32S31 || CONFIG_IDF_TARGET_ESP32P4
esp_capture_audio_aec_src_cfg_t aec_cfg = {
    .record_handle = codec_handle->codec_dev,
#if CONFIG_IDF_TARGET_ESP32S3 || CONFIG_IDF_TARGET_ESP32S31
    .channel      = 4,            // 4-channel ADC
    .channel_mask = 1 | 2,        // 取参考/回采通道
#endif
};
esp_capture_audio_src_if_t *aud_aec = esp_capture_new_audio_aec_src(&aec_cfg);
#endif

// --- 视频 source（V4L2，P4-Function-EV 等）---
dev_camera_handle_t *camera_handle = NULL;
esp_board_device_get_handle(ESP_BOARD_DEVICE_NAME_CAMERA, (void **)&camera_handle);

esp_capture_video_v4l2_src_cfg_t v4l2_cfg = {
    .buf_count = 2,
};
strncpy(v4l2_cfg.dev_name, camera_handle->dev_path, sizeof(v4l2_cfg.dev_name) - 1);
esp_capture_video_src_if_t *vid_src = esp_capture_new_video_v4l2_src(&v4l2_cfg);
```

> 视频源也支持 DVP：`esp_capture_new_video_dvp_src`（同样来自 `esp_capture_defaults.h`）。

### 3. open capture（含音视频 src 与 sync_mode）

```c
esp_capture_cfg_t capture_cfg = {
    .sync_mode   = ESP_CAPTURE_SYNC_MODE_AUDIO,   // 视频跟随音频（AV 同步采集必选）
    .audio_src   = aud_src,
    .video_src   = vid_src,                        // 纯音频采集设 NULL
    .share_overlay = false,                        // true：overlay 先合成再分给所有 sink
};
esp_capture_handle_t capture = NULL;
esp_capture_open(&capture_cfg, &capture);
```

`sync_mode` 取值：`NONE`（不同步）/ `SYSTEM`（按系统时间）/ `AUDIO`（视频跟随音频，最常用）。

### 4. setup sink 并设定目标格式（触发自动协商）

每个 sink 通过 `sink_idx` 区分；`esp_capture_sink_setup` 内部按 source 真实格式与 sink 目标 FourCC 自动构造「格式转换 → 编码」path。

```c
// sink 0：AAC 音频 + H264 视频
esp_capture_sink_cfg_t sink0_cfg = {
    .audio_info = {
        .format_id       = ESP_CAPTURE_FMT_ID_AAC,
        .sample_rate     = 16000,
        .channel         = 2,
        .bits_per_sample = 16,
    },
    .video_info = {
        .format_id = ESP_CAPTURE_FMT_ID_H264,
        .width     = 720,
        .height    = 1280,
        .fps       = 30,
    },
};
esp_capture_sink_handle_t sink0 = NULL;
esp_capture_sink_setup(capture, 0, &sink0_cfg, &sink0);

// sink 1：第二个目标格式（如低分辨率 MJPEG 给网络推流）
esp_capture_sink_cfg_t sink1_cfg = {
    .video_info = {
        .format_id = ESP_CAPTURE_FMT_ID_MJPEG,
        .width     = 360, .height = 640, .fps = 15,
    },
};
esp_capture_sink_handle_t sink1 = NULL;
esp_capture_sink_setup(capture, 1, &sink1_cfg, &sink1);
```

> 支持的 FourCC（`esp_capture_format_id_t`）：音频 `PCM/G711A/G711u/OPUS/AAC`；视频 `H264/MJPEG/RGB565/RGB565_BE/RGB888/BGR888/YUV420/YUV422P/YUV422/O_UYY_E_VYY`，以及 `ANY`(0xFFFF) 作为协商失败兜底。

### 5. （可选）给 sink 挂 muxer 写 MP4 切片

`url_pattern_ex` 回调在每次切片到期时被调，用于生成下一个文件路径。

```c
static int storage_slice_hdlr(esp_muxer_slice_info_t *info, void *ctx) {
    snprintf(info->file_path, info->len, "/sdcard/vid_%d.mp4", info->slice_index);
    return 0;
}

mp4_muxer_config_t mp4_cfg = {
    .base_config = {
        .muxer_type     = ESP_MUXER_TYPE_MP4,
        .url_pattern_ex = storage_slice_hdlr,
        .slice_duration = 60000,           // 60 s 切一片
        .ctx            = NULL,
    },
};
esp_capture_muxer_cfg_t muxer_cfg = {
    .base_config = &mp4_cfg.base_config,
    .cfg_size    = sizeof(mp4_cfg),
    .muxer_mask  = ESP_CAPTURE_MUXER_MASK_ALL,   // 音视频都封装
};
esp_capture_sink_add_muxer(sink0, &muxer_cfg);
esp_capture_sink_enable_muxer(sink0, true);
```

> 音频 muxer 示例用的是旧字段 `url_pattern`（仅文件名+idx），视频示例用 `url_pattern_ex`（带 `esp_muxer_slice_info_t`）。两者都来自 `mp4_muxer.h`。`muxer_mask`：`ALL`(0)/`AUDIO`(1)/`VIDEO`(2)，用于只封装某一流。

### 6. （可选）给 sink 加文字 overlay 叠加

overlay 仅在视频源 RGB565 时可用；需先 `set_fixed_caps` 把源锁成 RGB565。

```c
#include "esp_gmf_video_overlay.h"   // 颜色宏来自这里

// 锁定源输出为 RGB565（overlay 依赖）
esp_capture_video_info_t fixed = {
    .format_id = ESP_CAPTURE_FMT_ID_RGB565,
    .width = 720, .height = 1280, .fps = 30,
};
vid_src->set_fixed_caps(vid_src, &fixed);

esp_capture_rgn_t rgn = { .x = 100, .y = 100, .width = 100, .height = 40 };
esp_capture_overlay_if_t *text_ov = esp_capture_new_text_overlay(&rgn);
text_ov->open(text_ov);
text_ov->set_alpha(text_ov, 0xFF);   // 不透明

// 初始底色
esp_capture_text_overlay_draw_start(text_ov);
esp_capture_rgn_t full = { .x = 0, .y = 0, .width = 100, .height = 30 };
esp_capture_text_overlay_clear(text_ov, &full, COLOR_RGB565_CYAN);
esp_capture_text_overlay_draw_finished(text_ov);

esp_capture_sink_add_overlay(sink0, text_ov);
esp_capture_sink_enable_overlay(sink0, true);   // 可在 start 后动态开关

// 运行中刷新文字（定时调）
esp_capture_text_overlay_draw_info_t font = { .color = COLOR_RGB565_WHITE, .font_size = 12 };
esp_capture_text_overlay_draw_start(text_ov);
esp_capture_text_overlay_clear(text_ov, &full, COLOR_RGB565_CYAN);
esp_capture_text_overlay_draw_text_fmt(text_ov, &font, "PTS: %d\nText Overlay", elapsed_ms);
esp_capture_text_overlay_draw_finished(text_ov);
```

### 7. enable sink + start，然后拉帧/释放

```c
// ALWAYS 持续采集；ONESHOT 每次只抓一帧（适合抓拍）
esp_capture_sink_enable(sink0, ESP_CAPTURE_RUN_MODE_ALWAYS);

esp_capture_set_event_cb(capture, my_event_cb, ctx);
esp_capture_start(capture);

// 流式拉帧（muxer 内部会自动写文件，这里只读不读都行）
esp_capture_stream_frame_t frame = { .stream_type = ESP_CAPTURE_STREAM_TYPE_AUDIO };
while (running) {
    if (esp_capture_sink_acquire_frame(sink0, &frame, false) == ESP_CAPTURE_ERR_OK) {
        // 用 frame.data / frame.size / frame.pts
        esp_capture_sink_release_frame(sink0, &frame);   // 必须成对释放
    }
}
// ONESHOT 抓拍：循环 enable(ONESHOT) → acquire(wait) → release，间隔 sleep
```

事件类型：`STARTED/STOPPED/ERROR/AUDIO_PIPELINE_BUILT/VIDEO_PIPELINE_BUILT`。后两者允许你在 pipeline 已构建但尚未 run 时配置 element（见第 8 步）。

### 8. （可选）自定义处理链：注册 element + 手工 build pipeline

需要插入标准 `gmf_loader` 之外的 element（如 ALC、自定义特效）时：

```c
#include "esp_gmf_alc.h"

// 在 start 之前：建一个 ALC element 注册进 capture 内部 pool
esp_ae_alc_cfg_t alc_cfg = DEFAULT_ESP_GMF_ALC_CONFIG();
esp_gmf_element_handle_t alc_hd = NULL;
esp_gmf_alc_init(&alc_cfg, &alc_hd);
esp_capture_register_element(capture, ESP_CAPTURE_STREAM_TYPE_AUDIO, alc_hd);
// 注册成功后 element 归 capture 所有，close 时自动销毁；失败需自己 esp_gmf_obj_delete

// 用事件回调在 pipeline 建好后调 element 的运行时 setter
static esp_capture_err_t on_built(esp_capture_event_t event, void *ctx) {
    if (event == ESP_CAPTURE_EVENT_AUDIO_PIPELINE_BUILT) {
        esp_gmf_element_handle_t alc = NULL;
        esp_capture_sink_get_element_by_tag(ctx, ESP_CAPTURE_STREAM_TYPE_AUDIO, "aud_alc", &alc);
        if (alc) esp_gmf_alc_set_gain(alc, 0, -3);
    }
    return ESP_CAPTURE_ERR_OK;
}
esp_capture_set_event_cb(capture, on_built, sink0);

// 手工指定 element 顺序（名字必须在 pool 中存在；vid_color_cvt/vid_fps_cvt/vid_enc/vid_ppa 均由 capture 内部注册）
const char *aud_chain[] = {"aud_ch_cvt", "aud_rate_cvt", "aud_alc", "aud_enc"};
esp_capture_sink_build_pipeline(sink0, ESP_CAPTURE_STREAM_TYPE_AUDIO,
                                aud_chain, sizeof(aud_chain) / sizeof(aud_chain[0]));
```

### 9. 停止与销毁

```c
esp_capture_stop(capture);
esp_capture_close(capture);   // 销毁所有 path 与 pool 内 element
// source 由创建者自己释放：free(aud_src); free(vid_src);
```

## 常见错误

| 错误 | 原因 | 解决方法 |
|---|---|---|
| `sink_setup` 返回 `INVALID_STATE` | 已经 start 后再 setup | 所有 `setup/add_muxer/add_overlay/build_pipeline` 都必须在 `esp_capture_start` 之前 |
| 视频采集启动失败 | 摄像头未初始化或分辨率/fps 组合不支持 | 用 `esp_board_manager` 初始化 camera；核对 `VIDEO_SINKx_*` 配置 |
| AEC 不生效或无效 | AEC 仅 S3/S3-1/P4 支持，且需 4 通道 ADC | 在 `CONFIG_IDF_TARGET_*` 下设 `channel=4, channel_mask=1\|2`，并用 `esp_capture_new_audio_aec_src` |
| MP4 文件未生成 | SD 卡未挂载或 `mp4_muxer_register()` 未调用 | `esp_board_manager` 初始�� `FS_SDCARD`；启动时调 `mp4_muxer_register()` |
| overlay 无效 | overlay 仅支持 RGB565 源 | 先 `vid_src->set_fixed_caps(...RGB565...)` 再 setup sink |
| acquire_frame 一直失败 | sink 未 enable 或 run_mode 不对 | 先 `esp_capture_sink_enable(sink, ESP_CAPTURE_RUN_MODE_ALWAYS)`；ONESHOT 模式每次抓拍前都要再 enable 一次 |
| 多 sink 第二路协商不出 | 第二 sink 的 FourCC/分辨率与源无法转换 | 选 capture 内置转换链支持的组合（如 H264+MJPEG、不同分辨率），或用 `ESP_CAPTURE_FMT_ID_ANY` 兜底 |
| `register_element` 返回 `INVALID_STATE` | 在 start 之后才注册 | 元素注册必须在 start 前；失败分支记得 `esp_gmf_obj_delete(alc_hd)` |
| 性能不稳/掉帧 | 内部线程调度不合理 | 用 `esp_capture_set_thread_scheduler` 调栈/核/PSRAM；降分辨率或换更轻编码格式 |

## 参考

- `packages/esp_capture/examples/audio_capture/main/audio_capture.c`（basic / AEC / muxer / 自定义链 四个 case）
- `packages/esp_capture/examples/video_capture/main/video_capture.c`（basic / one-shot / overlay / muxer / 自定义 / dual-path 六个 case）
- `packages/esp_capture/include/esp_capture.h`、`esp_capture_sink.h`、`esp_capture_advance.h`、`esp_capture_types.h`、`esp_capture_defaults.h`
- `docs/en/gmf-framework/gmf-package/esp-capture.rst`、`docs/zh_CN/gmf-framework/gmf-package/esp-capture.rst`
