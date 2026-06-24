# ESP-VISION API Reference（速查）

> 来源：仓库 `stubs/*.pyi`（类型存根，API 签名权威）、`docs/zh_CN/api-reference/*.rst`、`example/`。仅收录仓库真实存在的 API；不存在则不写。
> MicroPython 标准库（`machine`/`network`/`socket`/`time`/`os` 等）以 MicroPython v1.28.0 文档为准，此处不重复。

## sensor — 摄像头采集（`stubs/sensor.pyi`）

```python
# 模块级常量
sensor.GRAYSCALE: int
sensor.RGB565: int
sensor.QQVGA: int
sensor.QVGA: int

# 模块级函数
sensor.reset() -> None
sensor.shutdown(enable: bool = True) -> None        # True 关闭，False 重启
sensor.set_pixformat(pixformat: int) -> None
sensor.get_pixformat() -> int
sensor.set_framesize(framesize: int) -> None
sensor.get_framesize() -> int
sensor.width() -> int
sensor.height() -> int
sensor.get_id() -> int
sensor.set_hmirror(enable: bool) -> None
sensor.get_hmirror() -> bool
sensor.set_vflip(enable: bool) -> None
sensor.get_vflip() -> bool
sensor.skip_frames(time: int = 0, n: int = 0) -> None
sensor.snapshot(buffer=None) -> image.Image         # 由可复用帧缓冲承载
sensor.status() -> SensorStatus
```

`SensorStatus`（TypedDict）字段：`ready`、`id`、`width`、`height`、`pixformat`、`hmirror`、`vflip`、`raw_width`、`raw_height`、`active_width`、`active_height`。

## image — 图像处理与编解码（`stubs/image.pyi`）

### 模块级常量（节选）
```python
# 像素格式
image.BINARY / GRAYSCALE / RGB565 / BAYER / YUV422 / JPEG / PNG
# JPEG 色度抽样
image.JPEG_SUBSAMPLING_AUTO / _444 / _422 / _420
# AprilTag 家族
image.TAG16H5 / TAG25H7 / TAG25H9 / TAG36H10 / TAG36H11
# 条码符号（find_barcodes().type()）
image.EAN2 / EAN5 / EAN8 / UPCE / ISBN10 / UPA / EAN13 / ISBN13 / I25
image.DATABAR / DATABAR_EXP / CODABAR / CODE39 / PDF417 / CODE93 / CODE128
# 缩放/几何提示（copy/crop/scale/draw_image）
image.AREA / BILINEAR / BICUBIC / HMIRROR / VFLIP / TRANSPOSE
image.ROTATE_90 / ROTATE_180 / ROTATE_270
image.SCALE_ASPECT_KEEP / SCALE_ASPECT_EXPAND / SCALE_ASPECT_IGNORE
image.CENTER / BLACK_BACKGROUND
# 边缘 / 角点
image.SEARCH_EX / SEARCH_DS
image.EDGE_CANNY / EDGE_SIMPLE
image.CORNER_FAST / CORNER_AGAST
# 调色板
image.PALETTE_RAINBOW / PALETTE_IRONBOW / PALETTE_DEPTH / PALETTE_EVT_DARK / PALETTE_EVT_LIGHT
```

类型别名：`Color = int | (int,int,int)`；`Threshold` = 灰度 `(min,max)` 或 LAB 六元组 `(l_min,l_max,a_min,a_max,b_min,b_max)`；`Thresholds = Sequence[Threshold]`。

### Image 类
```python
image.Image(path: str, *, copy_to_fb=False)
image.Image(width, height, pixformat, *, buffer=None, copy_to_fb=False)
image.Image(array, *, buffer=None, copy_to_fb=False)   # 包裹数组/缓冲

# 属性/基础
img.width() / img.height() / img.format() / img.size()
img.bytearray() -> bytearray
img.get_pixel(x, y, *, rgbtuple=True)
img.set_pixel(x, y, pixel) -> Image
img.flush() -> None                 # 主机 USB CDC 预览（EVFRAME JPEG）
img.save(path, *, roi=None, quality=50) -> Image

# 复制 / 缩放 / 转换 / 编码
img.copy(*, roi=None, x_scale=None, y_scale=None, rgb_channel=-1, alpha=255,
         color_palette=None, alpha_palette=None, hint=0, transform=None, copy=True) -> Image
img.crop(*, roi=None, x_scale=None, y_scale=None, hint=0, copy=False) -> Image
img.scale(*, x_scale=None, y_scale=None, roi=None, hint=0, copy=False) -> Image
img.compress(*, quality=90, subsampling=..., roi=None) -> Image   # to_jpeg 别名
img.to_bitmap(*, copy=False, roi=None) -> Image
img.to_grayscale(*, copy=False, roi=None) -> Image
img.to_rgb565(*, copy=False, roi=None) -> Image
img.to_rainbow(*, copy=False, roi=None) -> Image
img.to_ironbow(*, copy=False, roi=None) -> Image
img.to_jpeg(*, quality=90, subsampling=..., copy=False) -> Image
img.to_png(*, copy=False) -> Image

# 绘图（均返回 self）
img.draw_line(x0,y0,x1,y1, color=..., thickness=1) / draw_line(line, ...)
img.draw_rectangle(x,y,w,h, color=..., thickness=1, fill=False) / draw_rectangle(rect, ...)
img.draw_circle(x,y,r, color=..., thickness=1, fill=False) / draw_circle(circle, ...)
img.draw_ellipse(x,y,rx,ry,rotation, color=..., thickness=1, fill=False) / draw_ellipse(ellipse, ...)
img.draw_string(x, y, text, color=..., scale=1.0, x_spacing=0, y_spacing=0,
                mono_space=True, char_rotation=0, char_hmirror=False, char_vflip=False,
                string_rotation=0, string_hmirror=False, string_vflip=False) -> Image
img.draw_cross(x, y, color=..., size=5, thickness=1) -> Image
img.draw_arrow(x0,y0,x1,y1, color=..., size=10, thickness=1) / draw_arrow(line, ...)
img.draw_edges(corners, color=..., size=0, thickness=1, fill=False) -> Image
img.draw_keypoints(keypoints, color=..., size=10, thickness=1, fill=False) -> Image
img.draw_image(image, x=0, y=0, *, x_scale=None, y_scale=None, roi=None,
               rgb_channel=-1, alpha=255, color_palette=None, alpha_palette=None,
               hint=0, transform=None, mask=None) -> Image

# 阈值化 / 形态学 / 点操作
img.binary(thresholds, *, invert=False, zero=False, mask=None, to_bitmap=False, copy=False) -> Image
img.invert() -> Image
img.erode(ksize, *, threshold=0, mask=None) -> Image
img.dilate(ksize, *, threshold=0, mask=None) -> Image
img.open(ksize, *, threshold=0, mask=None) -> Image       # erode -> dilate
img.close(ksize, *, threshold=0, mask=None) -> Image      # dilate -> erode
img.difference(image, x=0, y=0, *, roi=None, mask=None) -> Image
img.histeq(*, adaptive=False, clip_limit=-1, mask=None) -> Image
img.clear(*, mask=None) -> Image

# 邻域/秩/保边滤波（ksize 为核半径；threshold/offset/invert 可把滤波变自适应阈值）
img.mean(ksize, *, mask=None, ...)
img.median(ksize, *, percentile=0.5, mask=None, ...)
img.mode(ksize, *, mask=None, ...)
img.midpoint(ksize, *, bias=0.5, mask=None, ...)
img.gaussian(ksize, *, unsharp=False, mask=None, ...)
img.laplacian(ksize, *, sharpen=False, mask=None, ...)
img.bilateral(ksize, *, color_sigma=..., space_sigma=..., mask=None, ...)
img.morph(ksize, kernel, *, mul=None, add=0, mask=None, ...)   # 任意整数卷积核

# 统计
img.get_histogram(*, roi=None, ...) -> histogram
img.get_statistics(*, roi=None, ...) -> statistics

# 特征检测（部分方法受 imlib_config.h 的 IMLIB_ENABLE_* 控制）
img.find_blobs(thresholds, *, pixels_threshold=..., area_threshold=..., merge=False, ...) -> list[blob]
img.find_lines(*, threshold=..., theta_margin=..., rho_margin=..., roi=None, ...) -> list[line]
img.find_circles(*, threshold=..., r_min=..., r_max=..., r_step=..., roi=None, ...) -> list[circle]
img.find_rects(*, roi=None, ...) -> list[rect]
img.find_qrcodes(*, roi=None, ...) -> list[qrcode]
img.find_barcodes(*, roi=None, ...) -> list[barcode]        # 仅 ESP32-P4（ZXing-C++）
img.find_apriltags(*, families=..., pose=False, roi=None, ...) -> list[apriltag]
```

### 结果对象（节选；方法带括号，apriltag 的属性是字段）
- `blob`：`rect()`/`x()`/`y()`/`w()`/`h()`/`cx()`/`cy()`/`pixels()`/`rotation()`/`code()`/`density()`/`roundness()` ...
- `line`：`line()`/`x1()`/`y1()`/`x2()`/`y2()`/`length()`/`magnitude()`/`theta()`/`rho()`
- `circle`：`circle()`/`x()`/`y()`/`r()`/`magnitude()`
- `rect`：`corners()`/`rect()`/`x()`/`y()`/`w()`/`h()`/`magnitude()`
- `qrcode`：`rect()`/`payload()`/`version()`/`ecc_level()`/`data_type()`/`is_numeric()` ...
- `barcode`：`rect()`/`payload()`/`type()`/`rotation()`/`quality()`
- `apriltag`（字段）：`corners`、`rect`、`x`/`y`/`w`/`h`、`id`、`family`、`name`、`cx`/`cy`、`rotation`、`decision_margin`、`hamming`、`goodness`，以及 `pose=True` 时的 `x_translation`/`y_translation`/`z_translation`/`x_rotation`/`y_rotation`/`z_rotation`
- `histogram`：`bins()`/`l_bins()`/`a_bins()`/`b_bins()`/`get_percentile(p)`/`get_threshold()`/`get_stats()`
- `statistics`：`mean()`/`median()`/`mode()`/`stdev()`/`min()`/`max()`/`lq()`/`uq()` 及 `l_*`/`a_*`/`b_*` LAB 变体

## image.ImageIO — 图像序列流（`stubs/imageio.pyi`，暴露为 `image.ImageIO`）

```python
image.FILE_STREAM: int
image.MEMORY_STREAM: int

# 文件流
image.ImageIO(path: str, mode: "r"|"w")
# 内存流（预分配 count 帧）
image.ImageIO((width, height, pixformat), count)

stream.type() -> int            # FILE_STREAM / MEMORY_STREAM
stream.is_closed() -> bool
stream.count() -> int           # 已录制帧数
stream.offset() -> int          # 当前读写游标
stream.version() -> int | None  # 文件流容器版本；内存流为 None
stream.buffer_size() -> int | None  # 内存流单帧缓冲；文件流为 None
stream.size() -> int            # 总字节数
stream.write(image) -> ImageIO  # 追加一帧（记录与上一帧间隔）
stream.read(copy_to_fb=True, *, loop=True, pause=True) -> Image | None
stream.seek(offset) -> ImageIO
stream.sync() -> ImageIO        # 文件流刷盘；内存流 no-op
stream.close() -> None
```

## display — 板载 LCD（`stubs/display.pyi`）

```python
display.ESP32Display(width=0, height=0, refresh=60, *, backlight=100)
display.Display = display.ESP32Display      # 别名

lcd.deinit() -> None
lcd.width() -> int
lcd.height() -> int
lcd.clear(display_off=False) -> None
lcd.backlight(value: int | None = None) -> int | None   # None 返回当前；否则设 0-100
lcd.write(image, *, x=0, y=0, x_scale=None, y_scale=None,
          roi=None, fit=True) -> None
```

## espdl — 模型推理（`stubs/espdl.pyi`，C++ 模块）

```python
espdl.load_model(path: str, *, profile=False) -> bool

# 结果元组
Detection        = (x, y, w, h, score, category)
PoseDetection    = (x, y, w, h, score, category, list[(x, y)])   # 17 COCO 关键点
Classification   = (label, score)

class espdl.ESPDet(path, *, score=None, nms=None, mean=None, std=None)
    .detect(image) -> list[Detection]
    .set_thresholds(*, score=None, nms=None) -> ESPDet
    .deinit() -> None

class espdl.YOLO11(path, *, score=None, nms=None, topk=10, mean=None, std=None)
    .detect(image) -> list[Detection]
    .set_thresholds(*, score=None, nms=None) -> YOLO11
    .deinit() -> None

class espdl.YOLO11nPose(path, *, score=None, nms=None, topk=10, mean=None, std=None)
    .detect(image) -> list[PoseDetection]
    .set_thresholds(*, score=None, nms=None) -> YOLO11nPose
    .deinit() -> None

class espdl.ImageNetCls(path, *, topk=5, score=None, mean=None, std=None, softmax=True)
    .classify(image) -> list[Classification]
    .set_thresholds(*, topk=None, score=None) -> ImageNetCls
    .deinit() -> None
```

## h264 — H.264 编码（仅 ESP32-P4，`stubs/h264.pyi`）

```python
class h264.H264Encoder(width, height, *, fps=15, gop=0, bitrate=0, qp_min=25, qp_max=45)
    .encode(image) -> bytes       # Annex-B NAL 单元
    .keyframe() -> bool           # 最近一帧是否 IDR/I
    .close() -> None              # 之后使用抛 OSError
```

## rtsp — RTSP 推流（仅 ESP32-P4，`stubs/rtsp.pyi`）

```python
class rtsp.RTSPServer(width, height, *, fps=15, listen_port=8554, max_frame_len=0)
    .send(nal: bytes) -> None     # 入队一帧；慢客户端/缺席整帧整丢；超 max_frame_len 忽略
    .stop() -> None               # 停服、join 任务、释放队列
```

## Wi-Fi MJPEG / 云端 AI（无专用 C 模块；基于 MicroPython 标准库 + `image` 编解码）

以下两类应用不引入新 C 模块，而是组合 `image.Image.to_jpeg()` + MicroPython 标准库（`network`/`socket`/`ssl`/`binascii`/`json`）。完整可运行实现见对应配方，此处仅列关键 API 与约定。

### JPEG 帧 → 字节（`stubs/image.pyi`）

```python
jpg = img.to_jpeg(quality=35, subsampling=image.JPEG_SUBSAMPLING_420)  # 返回 JPEG Image
data = jpg.bytearray()        # 取编码字节（也可 jpg.save(path) 落盘）
```

- `quality` 1-100，权衡体积与清晰度（MJPG 推流常取 30-50）。
- `subsampling`：`JPEG_SUBSAMPLING_420`（最小，彩色）/ `_422` / `_444`（最佳质量）；`JPEG_SUBSAMPLING_AUTO` 由编码器选。
- 有硬件 JPEG 的板走硬件，否则走软件 `esp_new_jpeg`（来源 `docs/zh_CN/concepts/codec-streaming.rst`）。

### HTTP MJPEG 流（`recipes/wifi_mjpeg_stream.md`）

浏览器原生支持 `Content-Type: multipart/x-mixed-replace; boundary=<token>`，每帧按以下格式逐字节写出（`socket.send` 可能只发部分，需循环 `send` 至写空）：

```
--<boundary>\r\n
Content-Type: image/jpeg\r\n
Content-Length: <n>\r\n
\r\n
<jpeg bytes>\r\n
```

### HTTPS POST + TLS + chunked 解码（`recipes/cloud_ai_vision.md`）

```python
# MicroPython v1.28.0；优先 ssl（标准），回退 tls
import ssl as tls                          # 或: import tls
context = tls.SSLContext(tls.PROTOCOL_TLS_CLIENT)
context.verify_mode = tls.CERT_NONE        # 设备无根 CA 时关闭校验（信任网络前提）
sock = context.wrap_socket(sock, server_hostname=host)

# OpenAI 响应常用 Transfer-Encoding: chunked，必须按 <hex-size>\r\n<data>\r\n 解析后再 json.loads
```

OpenAI 兼容 vision API 的 JSON 结构：`messages[-1].content` 为数组，含 `{"type":"image_url","image_url":{"url":"data:image/jpeg;base64,<b64>"}}` 与 `{"type":"text","text":<prompt>}`。base64 用 `binascii.b2a_base64(jpg.bytearray()).decode().strip()`。

