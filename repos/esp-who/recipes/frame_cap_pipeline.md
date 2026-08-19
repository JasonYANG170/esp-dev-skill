# 自定义帧采集流水线

> **适用摘要**: 理解并定制 ESP-WHO 的摄像头→处理节点链。讲解 `WhoFrameCap` + `WhoFetchNode`/`WhoDecodeNode`/`WhoPPAResizeNode` 的组装方式，以及 `fb_count`/`ringbuf_len` 的取值规则。

> Version: selected repo/component version; align with the user project and dependency manifest.
> Evidence: `repos/esp-who/resources/`, source/examples in `repos/esp-who/`, and this recipe path `repos/esp-who/recipes/frame_cap_pipeline.md`.
> Validation: draft metadata added from repository routing; verify APIs, Kconfig symbols, and component dependencies against the selected version.

## 触发意图

- "自定义 frame_cap pipeline"
- "加 PPA 缩放节点"
- "UVC 摄像头流水线"
- "fb_count 怎么算"
- "ringbuf_len 设多少"

## 前置条件

| 条件 | 要求 |
|---|---|
| 参考示例 | 任一示例的 `main/frame_cap_pipeline.cpp` |
| 头文件 | `who_frame_cap.hpp`、`who_cam.hpp` |

## 分步说明

### 1. 流水线模型（来自 `who_frame_cap.hpp`）

`WhoFrameCap` 是 `WhoTaskGroup`，通过模板 `add_node<T>(args...)` 顺序串联节点。相邻节点之间由 `add_node` 自动创建长度为 1 的 `QueueHandle_t`。每个节点内部有 `RingBuf<cam_fb_t*>`。

```cpp
auto frame_cap = new who::frame_cap::WhoFrameCap();
frame_cap->add_node<who::frame_cap::WhoFetchNode>("FrameCapFetch", cam);
// 可继续 add_node<WhoDecodeNode>(...) / WhoPPAResizeNode(...)
```

获取节点：
```cpp
frame_cap->get_last_node();            // 链尾（喂给下游 App）
frame_cap->get_node("FrameCapFetch");  // 按名取
frame_cap->get_all_nodes();
```

### 2. `fb_count` 与 `ringbuf_len` 取值规则（来自示例注释）

- `WhoFetchNode` 的 ringbuf_len = `cam->get_fb_count() - 2`（构造时自动算：`cam->get_fb_count() - 2`）。
- 想要“显示与检测对齐”，ringbuf 要覆盖下游处理耗时（以帧计）。
- 示例约定：LCD 模式 `fb_count = MODEL_TIME + 3`；term 模式 `fb_count = MODEL_TIME + 2`。
- 官方原话："If you have no idea how to set them, try with 5 and larger."

### 3. S3 DVP 流水线（最常用）

```cpp
#define MODEL_TIME 2   // 例：人脸检测

who::frame_cap::WhoFrameCap *get_dvp_frame_cap_pipeline() {
    framesize_t frame_size = who::cam::get_cam_frame_size_from_lcd_resolution();
#ifdef BSP_BOARD_ESP32_S3_KORVO_2
    auto cam = new who::cam::WhoS3Cam(PIXFORMAT_RGB565, frame_size, MODEL_TIME + 3, true, true);
#else
    auto cam = new who::cam::WhoS3Cam(PIXFORMAT_RGB565, frame_size, MODEL_TIME + 3);
#endif
    auto frame_cap = new who::frame_cap::WhoFrameCap();
    frame_cap->add_node<who::frame_cap::WhoFetchNode>("FrameCapFetch", cam);
    return frame_cap;
}
```

### 4. P4 MIPI-CSI 流水线（含可选 PPA 缩放）

```cpp
// 不带 PPA（模型内部缩放）
who::frame_cap::WhoFrameCap *get_mipi_csi_frame_cap_pipeline() {
    auto cam = new who::cam::WhoP4Cam(V4L2_PIX_FMT_RGB565, MODEL_TIME + 3);
    auto fc = new who::frame_cap::WhoFrameCap();
    fc->add_node<who::frame_cap::WhoFetchNode>("FrameCapFetch", cam);
    return fc;
}

// 带 PPA 缩放（仅 CONFIG_SOC_PPA_SUPPORTED，即 P4）
who::frame_cap::WhoFrameCap *get_mipi_csi_ppa_frame_cap_pipeline(
        who::frame_cap::WhoFrameCapNode **lcd_disp_frame_cap_node) {
    auto cam = new who::cam::WhoP4Cam(V4L2_PIX_FMT_RGB565, MODEL_TIME + 4);  // 多 1 个 fb
    auto fc = new who::frame_cap::WhoFrameCap();
    fc->add_node<who::frame_cap::WhoFetchNode>("FrameCapFetch", cam);
    fc->add_node<who::frame_cap::WhoPPAResizeNode>(
        "FrameCapPPAResize", MODEL_INPUT_W, MODEL_INPUT_H,
        dl::image::DL_IMAGE_PIX_TYPE_RGB565, MODEL_TIME);
    *lcd_disp_frame_cap_node = fc->get_node("FrameCapFetch");   // 显示用缩放前的帧
    return fc;
}
```

> `WhoPPAResizeNode` 构造签名：`(name, dst_w, dst_h, dst_pix_type, ringbuf_len, out_queue_overwrite=true)`，仅 `CONFIG_SOC_PPA_SUPPORTED` 可用。

### 5. UVC（USB 摄像头）流水线

UVC 输出 MJPEG，需 Decode；可选 PPA 缩放。多节点时各节点 `out_queue_overwrite=false`（阻塞 send，保证不丢）。

```cpp
who::frame_cap::WhoFrameCap *get_uvc_frame_cap_pipeline() {
    auto cam = new who::cam::WhoUVCCam(UVC_VS_FORMAT_MJPEG, 640, 480, 30, 4);
    auto fc = new who::frame_cap::WhoFrameCap();
    fc->add_node<who::frame_cap::WhoFetchNode>("FrameCapFetch", cam, false);
    fc->add_node<who::frame_cap::WhoDecodeNode>(
        "FrameCapDecode", dl::image::DL_IMAGE_PIX_TYPE_RGB565, 2, false);
    fc->add_node<who::frame_cap::WhoPPAResizeNode>(
        "FrameCapPPAResize", 800, 600, dl::image::DL_IMAGE_PIX_TYPE_RGB565, MODEL_TIME + 1);
    return fc;
}
```

`WhoDecodeNode`：`CONFIG_SOC_JPEG_CODEC_SUPPORTED` ? `hw_decode_jpeg` : `sw_decode_jpeg`；解码失败返回 `nullptr`，节点丢弃该帧。

### 6. 喂给 App

流水线尾节点交给 App 构造：
```cpp
auto frame_cap = get_mipi_csi_ppa_frame_cap_pipeline(&lcd_node);
auto app = new who::app::WhoDetectAppLCD({{255,0,0}}, frame_cap, lcd_node);
```

## 常见错误

| 错误 | 原因 | 解决方法 |
|---|---|---|
| `assert(ringbuf_len >= 1)` | `fb_count` 太小导致 FetchNode ringbuf_len≤0 | `fb_count` 至少 4，LCD 模式用 `MODEL_TIME+3` |
| 节点链尾找不到 prev / next | 节点未通过 `add_node` 串联 | 用 `add_node<T>` 而非手动 `new` + 注册 |
| P4 PPA 编译报未定义 | 非 P4 或未定义 `CONFIG_SOC_PPA_SUPPORTED` | PPAResizeNode 仅 P4 可用；S3 用模型内部缩放 |
| UVC 多节点丢帧 | `out_queue_overwrite=true`（默认）覆盖了未消费帧 | 多节点用 `out_queue_overwrite=false`（见示例） |
| 显示帧与检测框错位 | ringbuf_len 不够覆盖检测耗时 | 增大 `fb_count`/`MODEL_TIME`，让 ringbuf 容纳更多帧 |

## 参考

- `examples/object_detect/main/frame_cap_pipeline.cpp`（最全，含 lcd/term/ppa/uvc 全组合）
- `examples/qrcode_recognition/main/frame_cap_pipeline.cpp`
- `components/who_frame_cap/who_frame_cap.hpp`、`who_frame_cap_node.hpp`、`who_frame_cap_node.cpp`
- `resources/api_reference.md` §3 流水线、§4 摄像头
- `resources/state_machine.md` §2 节点链数据流
