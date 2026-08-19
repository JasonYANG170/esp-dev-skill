# 帧差法运动检测（内存 ImageIO 流）

> **适用摘要**: 用 `image.Image.difference()` 做背景差分检测运动，配合 `binary()` 阈值化与形态学清理得到运动掩码；演示内存 `image.ImageIO` 流做短帧历史缓存。对应 SKILL.md 避坑 #2（`snapshot()` 返回可复用缓冲，跨帧使用必须 `.copy()`）。

> Version: selected repo/component version; align with the user project and dependency manifest.
> Evidence: `repos/esp-vision/resources/`, source/examples in `repos/esp-vision/`, and this recipe path `repos/esp-vision/recipes/frame_differencing.md`.
> Validation: draft metadata added from repository routing; verify APIs, Kconfig symbols, and component dependencies against the selected version.

## 触发意图

- "运动检测 / 移动检测"
- "背景差分 / 帧差法"
- "difference 怎么用"
- "ImageIO 内存流"
- "image.ImageIO((w,h,fmt), count)"

## 前置条件

| 条件 | 要求 |
|---|---|
| 参考示例 | `example/02-Image-Processing/03-Frame-Differencing/in_memory_frame_differencing.py` |
| API 文档 | `docs/zh_CN/api-reference/imageio.rst`（内存流构造与 `read`/`seek`/`buffer_size`） |
| 权威签名 | `stubs/imageio.pyi`（`image.ImageIO`）、`stubs/image.pyi`（`difference` / `binary`） |
| 像素格式 | 推荐 `GRAYSCALE`（差分与阈值化在灰度上最直观） |

## 分步说明

帧差法核心：把当前帧与一个"背景帧"逐像素相减（`img.difference(background)`，原地修改），差值大的像素即为运动区域；再 `binary([(min, max)])` 阈值化为二值掩码，必要时 `erode`/`dilate` 去噪。`difference` 是**原地**操作，返回 `self`。

### 基本背景差分（对应示例脚本）

```python
import sensor
import time

THRESHOLDS = [(35, 255)]            # 灰度差值阈值：>35 视为变化

sensor.reset()
sensor.set_pixformat(sensor.GRAYSCALE)
sensor.set_framesize(sensor.QVGA)
sensor.skip_frames(time=1000)

# 关键：snapshot() 返回可复用帧缓冲，跨帧使用必须 copy
background = sensor.snapshot().copy()

while True:
    img = sensor.snapshot()
    img.difference(background)       # 原地：img = |img - background|
    img.binary(THRESHOLDS)           # 差值落入阈值即置为有效二值
    img.draw_string(2, 2, "frame diff", color=255)
    img.flush()
    time.sleep_ms(20)
```

参数说明：
- `difference(image)`：对每像素取绝对差；图像尺寸须一致。原地修改，返回 `self`。
- `binary([(min, max)])`：灰度阈值为 `(min, max)`；落入任一阈值的像素置为有效（最大值），其余置 0。
- `background` 必须 `.copy()`：否则下一帧 `snapshot()` 会覆盖它，差分恒为 0。

### 动态背景更新（慢适应）

固定背景会被光照渐变污染。用指数滑动平均让背景缓慢跟随：

```python
ALPHA = 0.92          # 越大背景越稳；越小越快适应新场景

background = sensor.snapshot().copy()

while True:
    img = sensor.snapshot()
    diff = img.copy()
    diff.difference(background)
    diff.binary([(35, 255)])
    diff.erode(1)
    diff.dilate(1)
    diff.flush()

    # 背景按 ALPHA 加权混合当前帧（in-place；blend 这里需手写逐像素或用 mean 近似）
    background = img.copy()          # 简化：每隔若干帧整体替换；真正 EMA 需逐像素加权
```

> 注：imlib 无内建 `blend`/`lerp`；严格 EMA 需逐像素 `get_pixel`/`set_pixel`（慢）或用 `mean`/`morph` 近似。生产里常用"周期性整帧替换背景 + 中值滤波"替代逐像素加权。

### 用内存 ImageIO 流做帧历史

`imageio.rst` 明确推荐内存流用于"短帧历史或帧差分工作流"。内存流把 `count` 帧预分配在 PSRAM，`seek`/`read` 随机回放，无存储 I/O：

```python
import sensor
import image

HISTORY = 10                         # 缓存帧数
sensor.reset()
sensor.set_pixformat(sensor.GRAYSCALE)
sensor.set_framesize(sensor.QVGA)
sensor.skip_frames(time=1000)

# 内存流构造签名：image.ImageIO((width, height, pixformat), count)
stream = image.ImageIO(
    (sensor.width(), sensor.height(), sensor.RGB565), HISTORY
)
print("type:", stream.type(), "buffer_size:", stream.buffer_size())

# 填满历史
for _ in range(HISTORY):
    stream.write(sensor.snapshot())

# 用最早的帧作为背景做差分；写新帧覆盖最旧
while True:
    img = sensor.snapshot()
    stream.write(img)                # 环形写入；offset 推进
    stream.seek(0)                   # 回到最旧帧
    background = stream.read(pause=False).copy()

    img.difference(background)
    img.binary([(35, 255)])
    img.flush()
```

内存流 API（来源 `stubs/imageio.pyi`）：
- 构造：`image.ImageIO((w, h, pixformat), count)` 预分配 `count` 帧；文件流则是 `image.ImageIO(path, "r"|"w")`。
- `type()` 返回 `image.FILE_STREAM` 或 `image.MEMORY_STREAM`。
- `buffer_size()` 仅内存流返回单帧字节数；文件流为 `None`。
- `count()` 已录制帧数；`offset()` 当前读写游标；`seek(offset)` 移动游标。
- `write(image)` 追加一帧并记录与上一帧的时间间隔；`read(copy_to_fb=True, *, loop=True, pause=True)` 读下一帧。
- `sync()` 文件流刷盘（内存流 no-op）；`close()` 释放。

> 内存流的全部容量在创建时一次性分配到 RAM/PSRAM，帧数过多会 OOM。QVGA 灰度单帧约 76 KB，10 帧约 760 KB——在 P4/S3 的 PSRAM 上宽裕，但需评估与其它分配（模型、显示）的总和。

## 常见错误

| 错误 | 原因 | 解决方法 |
|---|---|---|
| 差分结果全黑 | 背景帧未 `.copy()`，被下一帧覆盖 | `background = sensor.snapshot().copy()` |
| 满屏噪点 | 阈值过低 / 光照抖动 | 提高 `binary` 下限；先 `gaussian(1)` 降噪 |
| 运动区域空洞 / 碎片 | 形态学未清理 | `binary` 后 `erode(1)` 去孤立点，再 `dilate(1)` 恢复 |
| 背景很快失效 | 光照渐变 / 物体长期停留 | 周期性替换背景帧或 EMA 更新 |
| `MemoryError` 分配内存流 | `count` 过大 / PSRAM 不足 | 减少 `count` 或分辨率；关闭其它大分配 |
| `difference` 抛异常 | 两帧尺寸/格式不一致 | 背景与当前帧须同 `pixformat`、同 `framesize` |
| 把 `stream.version()` 当文件版本用 | 内存流 `version()` 返回 `None` | 仅文件流有容器版本；内存流用 `type()` 区分 |

## 参考项目

- `example/02-Image-Processing/03-Frame-Differencing/in_memory_frame_differencing.py`
- `example/02-Image-Processing/01-Image-Filters/difference.py`（`difference` 基础用法）
- `docs/zh_CN/api-reference/imageio.rst`（内存流构造与回放，含本示例的交叉引用）
- `docs/zh_CN/concepts/codec-streaming.rst`（ImageIO 与 JPEG/H.264 路径对比）
- `stubs/imageio.pyi`、`stubs/image.pyi`
