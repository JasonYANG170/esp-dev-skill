# 图像绘图原语总览

> **适用摘要**: 汇总 `image.Image` 的 `draw_*` 方法（line / rectangle / circle / ellipse / cross / arrow / string / image），覆盖颜色与线宽约定、RGB565 与 GRAYSCALE 的颜色传参差异，以及 `fill` 实心填充。所有方法**原地修改**并返回 `self`，可链式调用。

## 触发意图

- "在图像上画框 / 画圆 / 画线"
- "draw_rectangle / draw_circle / draw_ellipse / draw_arrow"
- "画文字标注"
- "OpenMV 绘图"
- "RGB565 颜色怎么传"

## 前置条件

| 条件 | 要求 |
|---|---|
| 参考示例 | `example/02-Image-Processing/00-Drawing/drawing.py`、`shape_drawing.py`、`line_drawing.py` |
| API 文档 | `docs/zh_CN/api-reference/image.rst`（Drawing and Color-Blob Tracking） |
| 权威签名 | `stubs/image.pyi`（`draw_*` 系列） |

## 分步说明

绘图方法用于在采集帧或检测结果上叠加标注。颜色参数 `Color` 是 `int | (int,int,int)`：**RGB565 图像接受 `(r, g, b)` 三元组**（每通道 0-255），**GRAYSCALE 图像接受单个灰度标量**（0-255）——这是常见踩坑点。所有 `draw_*` 原地修改图像并返回 `self`。

### 一次画全（对应 `drawing.py`）

```python
import sensor
import time

sensor.reset()
sensor.set_pixformat(sensor.RGB565)          # 彩色：颜色用 (r,g,b)
sensor.set_framesize(sensor.QVGA)
sensor.skip_frames(time=1000)

frame = 0
while True:
    img = sensor.snapshot()

    img.draw_line(10, 10, 200, 70, color=(255, 0, 0), thickness=2)
    img.draw_rectangle(20, 20, 80, 60, color=(0, 255, 0), thickness=2)
    img.draw_circle(160, 120, 30, color=(0, 0, 255), thickness=2)
    img.draw_cross(240, 60, color=(255, 255, 0), size=12, thickness=2)
    img.draw_arrow(20, 220, 200, 200, color=(255, 0, 255), thickness=2)
    img.draw_string(10, 90, "imlib draw %d" % frame, color=(255, 255, 255))

    img.flush()
    frame = (frame + 1) % 1000
    time.sleep_ms(20)
```

### 椭圆与旋转（对应 `shape_drawing.py`）

```python
img.draw_ellipse(245, 145, 45, 25, angle, color=(0, 0, 255), thickness=2)
# 参数：x, y, rx, ry, rotation(度), color, thickness, fill
```

`draw_ellipse` 签名（来源 `stubs/image.pyi`）：
```python
img.draw_ellipse(x, y, rx, ry, rotation, color=..., thickness=1, fill=False) -> Image
```
`rotation` 单位为度；`fill=True` 画实心椭圆。

### 实心矩形与圆

```python
img.draw_rectangle(50, 50, 40, 40, color=(255, 0, 0), thickness=2, fill=True)
img.draw_circle(120, 120, 25, color=(0, 255, 0), thickness=1, fill=True)
```

`fill=True` 忽略 `thickness` 直接填满；适合高亮 ROI。

### 灰度图绘图（颜色传标量）

```python
sensor.set_pixformat(sensor.GRAYSCALE)
img = sensor.snapshot()
img.draw_rectangle(10, 10, 50, 50, color=255, thickness=2)   # 标量，不是 (255,255,255)
img.draw_string(10, 70, "gray", color=255)
```

在 GRAYSCALE 图上把 `color` 写成 `(r,g,b)` 会报错或行为异常——必须传单整数。

### 用结果对象绘图

检测结果返回的对象可直接喂给对应绘图方法（多为元组重载）：

```python
for blob in img.find_blobs([(30, 100, 15, 127, 15, 127)]):
    img.draw_rectangle(blob.rect(), color=(255, 0, 0), thickness=2)   # 传 (x,y,w,h) 元组
    img.draw_cross(blob.cx(), blob.cy(), color=(0, 255, 0))

for line in img.find_lines(threshold=1400):
    img.draw_line(line.line(), color=255, thickness=2)                # 传 (x1,y1,x2,y2) 元组

for circle in img.find_circles(threshold=2500):
    img.draw_circle(circle.x(), circle.y(), circle.r(), color=255, thickness=2)
```

`draw_line`、`draw_rectangle`、`draw_circle`、`draw_arrow`、`draw_ellipse` 均有"散参"与"元组对象"两种重载（见 `stubs/image.pyi` 的 `@overload`）。

### 叠加另一张图

```python
img.draw_image(other, x=10, y=10, *, x_scale=None, y_scale=None,
               roi=None, alpha=255, hint=0, transform=None, mask=None)
```

`draw_image` 支持缩放、ROI、alpha 混合与几何 `transform`（`image.HMIRROR`/`VFLIP`/`ROTATE_90` 等）。

## 方法清单与签名（来源 `stubs/image.pyi`）

| 方法 | 签名（节选） | 说明 |
|---|---|---|
| `draw_line` | `(x0,y0,x1,y1, color, thickness=1)` 或 `(line, ...)` | 直线 |
| `draw_rectangle` | `(x,y,w,h, color, thickness=1, fill=False)` 或 `(rect, ...)` | 矩形；`fill` 实心 |
| `draw_circle` | `(x,y,r, color, thickness=1, fill=False)` 或 `(circle, ...)` | 圆 |
| `draw_ellipse` | `(x,y,rx,ry,rotation, color, thickness=1, fill=False)` 或 `(ellipse, ...)` | 椭圆，`rotation` 度 |
| `draw_cross` | `(x,y, color, size=5, thickness=1)` | 十字标记，`size` 臂长 |
| `draw_arrow` | `(x0,y0,x1,y1, color, size=10, thickness=1)` 或 `(line, ...)` | 箭头，`size` 箭头尺寸 |
| `draw_string` | `(x,y,text, color, scale=1.0, x_spacing=0, y_spacing=0, mono_space=True, char_rotation=0, ...)` | 内置位图字体文字 |
| `draw_edges` | `(corners, color, size=0, thickness=1, fill=False)` | 按 corners 连多边形边 |
| `draw_keypoints` | `(keypoints, color, size=10, thickness=1, fill=False)` | 关键点标注（姿态等） |
| `draw_image` | `(image, x=0, y=0, *, x_scale, y_scale, roi, alpha=255, hint, transform, mask)` | 叠加/缩放/变换另一张图 |

> 所有方法原地修改并返回 `self`，故可 `img.draw_rectangle(...).draw_string(...).flush()`。

## 常见错误

| 错误 | 原因 | 解决方法 |
|---|---|---|
| 灰度图上传 `(r,g,b)` 报错 | GRAYSCALE 只接受标量颜色 | 改传单整数 `color=255` |
| 彩色图上颜色全黑/全白 | 传了标量被当作 RGB565 打包值 | 用 `(r,g,b)` 三元组 |
| 看不到标注 | `thickness=0` 或颜色与背景同 | `thickness>=1`；选与背景对比的颜色 |
| 椭圆形状奇怪 | `rotation` 单位搞错 | `draw_ellipse` 的 `rotation` 是度，非弧度 |
| 实心框覆盖了目标 | 误用 `fill=True` | 仅高亮 ROI 才 `fill=True`，描边用默认 `fill=False` |
| `draw_string` 中文乱码 | 内置位图字体不支持中文 | 用英文/ASCII；中文需自绘字模或 `draw_image` 贴图 |
| 链式调用中途断 | 某步返回 `None` | 所有 `draw_*` 都返回 `self`；若断了检查是否调成了非绘图方法 |

## 参考项目

- `example/02-Image-Processing/00-Drawing/drawing.py`（line/rectangle/circle/cross/arrow/string 综合演示）
- `example/02-Image-Processing/00-Drawing/shape_drawing.py`（rectangle/circle/ellipse/cross）
- `example/02-Image-Processing/00-Drawing/line_drawing.py`（line/arrow）
- `docs/zh_CN/concepts/image-processing.rst`（绘图所属的图像处理总论）
- `docs/zh_CN/api-reference/image.rst`（Drawing and Color-Blob Tracking 一节）
- `stubs/image.pyi`（`draw_*` 完整签名与 `@overload`）
