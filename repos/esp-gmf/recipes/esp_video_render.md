# 视频合成与显示叠加（esp_video_render）

> **适用摘要**: 用 `esp_video_render` 把一路或多路视频（H.264/MJPEG 解码输入）与 UI 叠加（overlay→container→widget 层级）合成到统一显示后端（直接 LCD / LVGL 集成 / framebuffer）。每路 stream 独立控制位置/裁剪/旋转/显隐/z-order/alpha；dirty-region 部分刷新减少重绘；支持同步或异步渲染、手动 compose 模式。覆盖视频播放器、智能屏、机器人双眼、视频门铃、摄像头预览等场景。

## 触发意图

- "视频显示/渲染"
- "esp_video_render"
- "H264/MJPEG 解码上屏"
- "视频 UI 叠加（进度条/字幕/控件）"
- "双目/双摄合成（dual_eyes）"
- "LVGL 视频背景"
- "dirty-region 部分刷新"

## 前置条件

| 条件 | 要求 |
|---|---|
| 硬件 | ESP32-P4（带 LCD/DSI）、ESP32-S3（RGB LCD）等显示板；UI 叠加需 RGB565 显示格式 |
| 组件 | `espressif/esp_video_render`（含 `vui` overlay/container/widget 子模块）；依赖 `gmf_core`、`esp_video_codec`（解码）、`esp_lvgl_port`（LVGL 后端可选） |
| 解码器 | MJPEG/H264 输入需 `esp_video_dec_register_default()`；纯 raw 帧不需要 |
| 参考头文件 | `packages/esp_video_render/include/esp_video_render.h`、`esp_video_render_types.h`、`esp_video_render_backend.h`、`esp_video_render_dual_stream.h`、`vui/esp_vui_container.h`、`vui/esp_vui_widget_default.h` |
| 参考示例 | `packages/esp_video_render/examples/video_render`（单/双 MJPEG、同步 vs cached、进度条 overlay、LCD 与 LVGL 后端）、`dual_eyes`（双目合成）、`video_player` |

## 分步说明

`esp_video_render` 系统层次：**render（一个后端）→ 多个 stream → 每 stream 一个视频输入(可选) + 一个 overlay(可选) → overlay 多个 container → container 多个 widget**。后端可在没 stream open 时切换；compose 默认 auto（每帧自动合成），manual 模式需手动 `esp_video_render_compose`。

### 1. 注册解码器与 element pool，create render

```c
#include "esp_video_render.h"
#include "esp_video_codec.h"   // esp_video_dec_register_default
#include "esp_gmf_pool.h"

// 解码 MJPEG/H264 输入帧才需要；传 raw RGB 帧到 stream 可跳过
esp_video_dec_register_default();

// pool 用来容纳解码等 element（示例 create_default_pool 注册必要 element）
esp_gmf_pool_handle_t pool = NULL;
esp_gmf_pool_init(&pool);   // 或参考示例 create_default_pool 注册视频解码 element

esp_video_render_cfg_t render_cfg = {
    .pool = pool,
    .fps  = 30,           // 渲染目标帧率；stream_info.fps 非零时按该值限速
};
esp_video_render_handle_t render = NULL;
esp_video_render_create(&render_cfg, &render);
```

### 2. 设置显示后端（LCD 直驱 或 LVGL）

后端二选一；没 stream open 时才能 `set_display`。

```c
#include "esp_video_render_backend.h"   // esp_video_render_get_lcd_backend / _get_lvgl_backend

esp_video_render_backend_cfg_t backend_cfg = {};

// 方式 A：直接 LCD 后端（esp_video_render_lcd_cfg_t 来自板级 dev_display_lcd_config_t）
esp_video_render_lcd_cfg_t lcd_cfg = {
    .out_format = ESP_VIDEO_RENDER_FORMAT_RGB565,   // SPI/MIPI 多为 RGB565；某些板需 RGB565_BE
    .width      = 480,
    .height     = 272,
    /* 其他 LCD 句柄字段由板级填 */
};
backend_cfg.ops      = esp_video_render_get_lcd_backend();
backend_cfg.cfg      = &lcd_cfg;
backend_cfg.cfg_size = sizeof(lcd_cfg);
esp_video_render_set_display(render, &backend_cfg);

// 方式 B：LVGL 后端（需要先 esp_lvgl_port 初始化 lv_disp）
esp_video_render_lvgl_cfg_t lv_cfg = {
    .out_format = ESP_VIDEO_RENDER_FORMAT_RGB565,
    .width      = lcd_cfg.width,
    .height     = lcd_cfg.height,
    .lv_disp    = disp_handle,   // 来自 lvgl_port_add_disp*
};
backend_cfg.ops      = esp_video_render_get_lvgl_backend();
backend_cfg.cfg      = &lv_cfg;
backend_cfg.cfg_size = sizeof(lv_cfg);
esp_video_render_set_display(render, &backend_cfg);

// 查询显示真实分辨率/格式（决定 stream disp_rect 与 overlay 格式）
esp_video_render_disp_info_t disp = {};
esp_video_render_get_display_info(render, &disp);   // disp.format / width / height
```

### 3. 设背景、compose 模式（stream open 之前）

```c
esp_video_render_clr_t bg = { .r = 0x40, .g = 0x40, .b = 0x40 };
esp_video_render_set_bg_color(render, &bg);

// 或设背景图（编码图会同步解码；raw 图需保证整个 render 生命周期内有效）
esp_video_render_img_t bg_img = { .info = { .format = ESP_VIDEO_RENDER_FORMAT_RGB565, .width = 480, .height = 272 },
                                  .data = bg_buf, .size = sizeof(bg_buf) };
esp_video_render_set_bg_image(render, &bg_img);

// 默认 AUTO（每帧自动合成）；MANUAL 模式省空闲 CPU，需手动 compose
esp_video_render_set_compose_mode(render, ESP_VIDEO_RENDER_COMPOSE_MODE_MANUAL);

// 重新配置内部渲染任务（仅在没 stream open 时；同步 blend 模式无需调）
esp_video_render_task_cfg_t task_cfg = { .stack_size = 6 * 1024, .priority = 5, .core_id = 0 };
esp_video_render_task_reconfigure(render, &task_cfg);
```

### 4. open stream（指定输入格式与 cached）

`cached=true` 用额外缓冲平衡解码与渲染速率（FPS 不匹配时避免互相等待），多用于异步渲染；`cached=false` 同步渲染（单流默认就在调用任务内渲染）。

```c
esp_video_render_stream_info_t stream_info = {
    .info   = {
        .format = ESP_VIDEO_RENDER_FORMAT_MJPEG,   // 或 H264 / RGB565 / YUV420P / UYVY ...
        .width  = 320,
        .height = 240,
        .fps    = 30,                               // 非零：renderer 限速到该 FPS
    },
    .cached = true,
};
esp_video_render_stream_handle_t stream = NULL;
esp_video_render_stream_open(render, &stream_info, &stream);

// 设置显示矩形（缩放/定位到屏幕区域）
esp_video_render_rect_t disp_rect = { .x = 80, .y = 40, .width = 320, .height = 240 };
esp_video_render_stream_set_disp_rect(stream, &disp_rect);

// 可选：源裁剪、旋转（0/90/180/270）、z-order（大者在上）、显隐、透明度
esp_video_render_rect_t src_rect = { .x = 0, .y = 0, .width = 320, .height = 240 };
esp_video_render_stream_set_src_rect(stream, &src_rect);
esp_video_render_stream_set_rotate(stream, 90);
esp_video_render_stream_set_zorder(stream, 10);
esp_video_render_stream_set_visible(stream, true);
esp_video_render_stream_set_alpha(stream, 255);   // 0=透明, 255=不透明

// cached 模式建议切异步渲染
esp_video_render_stream_render_async(stream);
```

输入格式枚举 `esp_video_render_format_t`：`H264/MJPEG/RGB565/RGB565_BE/RGB888/BGR888/YUV420P/YUV422P/UYVY/YUV422/O_UYY_E_VYY`。

### 5. 写帧（两种方式）

```c
// 方式 A：直接写 frame（推荐；data 由应用持有）
esp_video_render_frame_t frame = {
    .format = ESP_VIDEO_RENDER_FORMAT_MJPEG,
    .width  = 320, .height = 240,
    .data   = mjpeg_buf, .size = mjpeg_size,
    .pts    = pts_ms,
};
esp_video_render_stream_write(stream, &frame);

// 方式 B：acquire 显存 fb → 写 → write_fb → release（共享显存，省一次拷贝；不建议多 stream 共用）
esp_video_render_fb_t fb = {};
if (esp_video_render_stream_acquire_fb(stream, &fb) == ESP_VIDEO_RENDER_ERR_OK) {
    memcpy(fb.data, my_raw_data, fb.size);   // 或解码器直接解码进 fb.data
    esp_video_render_stream_write_fb(stream, &fb);
    esp_video_render_stream_release_fb(stream, &fb);
}

// 多线程修改 frame 数据时用 lock/unlock 保护一致性
esp_video_render_stream_lock(stream);
/* 修改 frame 数据 */
esp_video_render_stream_unlock(stream);
```

### 6. UI 叠加：overlay → container → widget

每个 stream 可挂一个 overlay；overlay 上可叠多个 container，每个 container 可放多个 widget（如进度条、字幕、按钮）。dirty-region 跟踪只刷新变化区域。

```c
#include "esp_vui_container.h"
#include "esp_vui_widget_default.h"

esp_vui_overlay_handle_t overlay = NULL;
esp_video_render_stream_get_overlay(stream, &overlay);

// container 位置 + 尺寸 + 显示格式（UI 叠加目前仅支持 RGB565/RGB565_BE）
esp_video_render_frame_info_t c_info = { .width = 200, .height = 20, .format = ESP_VIDEO_RENDER_FORMAT_RGB565 };
esp_video_render_pos_t       c_pos  = { .x = 60, .y = 250 };
esp_vui_container_handle_t container = NULL;
esp_vui_container_create(overlay, &c_info, &c_pos, true /*opaque*/, &container);

// 自定义 widget（实现 redraw/destroy 回调；rect 用 container 局部坐标）
static const esp_vui_widget_ops_t my_ops = { .redraw = my_redraw, .destroy = my_destroy };
my_widget.ops    = &my_ops;
my_widget.rect   = (esp_video_render_rect_t){ .x = 0, .y = 0, .width = 200, .height = 20 };
my_widget.dirty  = my_widget.rect;
my_widget.visible = true;
esp_vui_container_add_widget(container, &my_widget);

// 更新 UI 时：compose_lock → 改 widget 字段（如 progress 百分比）→ mark dirty → compose_unlock
esp_video_render_stream_compose_lock(stream);
bar->current_percent = new_pct;
bar->widget.dirty    = bar->widget.rect;   // 标记需要重绘
esp_video_render_stream_compose_unlock(stream);
```

> widget 的 `redraw` 由框架在合成时调用，写入 container 的 framebuffer；dirty region 只覆盖标记过的 rect。

### 7. 手动 compose（仅 MANUAL 模式）

```c
// AUTO 模式每帧自动合成；MANUAL 模式由应用按需触发
esp_video_render_compose(render);
```

事件回调用于节能（所有 stream 关闭时挂起设备）：

```c
static int on_event(esp_video_render_event_type_t ev, void *ctx) {
    if (ev == ESP_VIDEO_RENDER_EVENT_TYPE_CLOSED) { /* 全部 close，可挂起 */ }
    else if (ev == ESP_VIDEO_RENDER_EVENT_TYPE_OPENED) { /* 至少一路 open */ }
    else if (ev == ESP_VIDEO_RENDER_EVENT_TYPE_VSYNC) { /* 一次绘制完成 */ }
    return 0;
}
esp_video_render_set_event_cb(render, on_event, NULL);
```

### 8. 双目/双流合成（dual_eyes 专用便捷 API）

封装两路解码 + 渲染（单屏或双屏），适合机器人双眼、双摄预览。每只眼一个 stream，按 `render[2]` 数组决定是同一 render（单屏分屏）还是两个 render（双屏）。

```c
#include "esp_video_render_dual_stream.h"

esp_video_render_dual_stream_cfg_t cfg = {
    .render          = { render, render },     // 双屏时填两个不同 render
    .task_cfg        = { { .stack_size = 6*1024, .priority = 5, .core_id = 0 },
                         { .stack_size = 6*1024, .priority = 5, .core_id = 1 } },
    .frame_count     = 3,                       // 每路帧缓冲数
    .max_frame_size  = 20 * 1024,               // 单帧最大字节（默认 20 KB）
    .fps             = 30,                      // 0 = 不限速
    .render_async    = true,                    // 解码与渲染并行（高 FPS 建议）
};
esp_video_render_dual_stream_handle_t eye = NULL;
esp_video_render_dual_stream_open(&cfg, &eye);

// 设每只眼的显示矩形（屏幕分左右或上下）
esp_video_render_rect_t left  = { .x = 0,   .y = 0, .width = 240, .height = 240 };
esp_video_render_rect_t right = { .x = 240, .y = 0, .width = 240, .height = 240 };
esp_video_render_dual_stream_set_display_rect(eye, 0, &left);
esp_video_render_dual_stream_set_display_rect(eye, 1, &right);

// 取缓冲 → 填两路数据 → 同时送显 → 释放
esp_video_render_frame_t fa = {}, fb = {};
esp_video_render_dual_stream_get_buffer(eye, 0, &fa);
esp_video_render_dual_stream_get_buffer(eye, 1, &fb);
/* 解码/填充 fa.data 与 fb.data */
esp_video_render_dual_stream_send_buffer(eye, &fa, &fb);
esp_video_render_dual_stream_release_buffer(eye, 0, &fa);
esp_video_render_dual_stream_release_buffer(eye, 1, &fb);

esp_video_render_dual_stream_close(eye);
```

### 9. 销毁

```c
esp_video_render_stream_close(stream);
esp_video_render_destroy(render);
esp_gmf_pool_deinit(pool);
```

> 性能分析：`esp_video_render_measure_enable(true); vTaskDelay(...); esp_video_render_measure_enable(false);` 在禁用时打印耗时统计。

## 常见错误

| 错误 | 原因 | 解决方法 |
|---|---|---|
| `stream_open` 返回 `NOT_SUPPORTED` | 后端未设置 | 先 `esp_video_render_set_display` 再 open |
| `set_display` 失败 | 有 stream 仍 open | 先 close 所有 stream 再换后端 |
| `task_reconfigure`/`set_compose_mode` 返回 `INVALID_STATE` | 已有 stream open | 这些 API 必须在 open 前调 |
| 视频不上屏 | 输入格式与 `stream_info.info.format` 不符 / 解码器未注册 | MJPEG/H264 输入前调 `esp_video_dec_register_default()`；raw 帧确保 format 一致 |
| UI 叠加不显示 | overlay/container 用了非 RGB565 格式 | UI 叠加目前仅支持 `RGB565`/`RGB565_BE`；按 `get_display_info` 的格式设 |
| 单流渲染卡顿 | cached=false 时同步在调用任务渲染 | `cached=true` + `stream_render_async` 切异步；或调大 task 栈 |
| 多 stream 抖动 | 共用 acquire_fb 的帧缓冲争用 | 多 stream 优先用 `stream_write(frame)` 而非 acquire_fb；fb 是全局共享 |
| 旋转返回 `NOT_SUPPORTED` | 角度非 0/90/180/270 | 用合法角度 |
| dirty region 不准 | 改了 widget 字段未标 dirty | `compose_lock` 内同时更新 `widget.dirty` rect 再 `compose_unlock` |
| dual_eyes 黑屏 | `render[2]` 没都创建或矩形超屏 | 单屏分屏填同一 render 两次；双屏填两个；矩形不超 `disp.width/height` |
| LVGL 后端创建失败 | `lv_disp` 未初始化 | 先用 `lvgl_port_add_disp*` 初始化再创建 LVGL 后端 |
| 内存吃紧 | cached + 多 stream 缓冲翻倍 | 降 `frame_count`/`max_frame_size`；非高 FPS 关 `render_async` |

## 参考

- `packages/esp_video_render/examples/video_render/main/video_render.c`（单/双 MJPEG、cached/sync、stream write、disp_rect）
- `packages/esp_video_render/examples/video_render/main/video_render_sys.c`（LCD 与 LVGL 两种后端初始化）
- `packages/esp_video_render/examples/video_render/main/progress.c`（自定义 widget + container 实现 UI 叠加）
- `packages/esp_video_render/examples/dual_eyes/main/dual_eyes.c`（双目 dual_stream 用法）
- `packages/esp_video_render/examples/video_player/main/video_player_app.c`（基于 render 的视频播放器）
- `packages/esp_video_render/include/esp_video_render.h`、`esp_video_render_types.h`、`esp_video_render_backend.h`、`esp_video_render_dual_stream.h`、`vui/esp_vui_container.h`、`vui/esp_vui_widget_default.h`
- `docs/en/gmf-framework/gmf-package/esp-video-render.rst`、`docs/zh_CN/gmf-framework/gmf-package/esp-video-render.rst`
