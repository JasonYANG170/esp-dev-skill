# 颜色色块追踪

> **适用摘要**: 用 LAB 阈值配合 `find_blobs` 做 8 连通色块检测，绘制外接框与十字，并用 `pixels_threshold` / `area_threshold` 过滤噪声。

> Version: selected repo/component version; align with the user project and dependency manifest.
> Evidence: `repos/esp-vision/resources/`, source/examples in `repos/esp-vision/`, and this recipe path `repos/esp-vision/recipes/color_blob_tracking.md`.
> Validation: draft metadata added from repository routing; verify APIs, Kconfig symbols, and component dependencies against the selected version.

## 触发意图

- "颜色追踪"
- "find_blobs 怎么用"
- "检测红色/绿色物体"
- "OpenMV 色块检测"
- "多色块追踪"

## 前置条件

| 条件 | 要求 |
|---|---|
| 参考示例 | `example/02-Image-Processing/02-Color-Tracking/find_blobs.py` |
| 像素格式 | `RGB565`（LAB 阈值需要彩色） |

## 分步说明

RGB565 阈值是六个 LAB 范围值：`(L_min, L_max, A_min, A_max, B_min, B_max)`。像素落入**任一**阈值即视为“有效”。

```python
import sensor
import time

THRESHOLDS = [(30, 100, 15, 127, 15, 127)]   # 示例：偏红区域

sensor.reset()
sensor.set_pixformat(sensor.RGB565)
sensor.set_framesize(sensor.QVGA)
sensor.skip_frames(time=1000)

while True:
    img = sensor.snapshot()
    blobs = img.find_blobs(THRESHOLDS, pixels_threshold=80, area_threshold=80, merge=True)
    for blob in blobs:
        img.draw_rectangle(blob.rect(), color=(255, 0, 0), thickness=2)
        img.draw_cross(blob.cx(), blob.cy(), color=(0, 255, 0))
    img.flush()
    time.sleep_ms(20)
```

参数说明：
- `pixels_threshold` 过滤匹配像素过少的区域；`area_threshold` 过滤外接矩形面积过小的区域。
- `merge=True` 合并相邻同色块。
- `blob.rect()` 返回 `(x, y, w, h)`；`blob.cx()` / `blob.cy()` 为质心，坐标以源图像像素为单位，可直接传给绘图方法。
- `blob` 还提供 `pixels()`、`rotation()`、`roundness()`、`density()`、`code()`（匹配阈值位掩码）等形状描述子（见 `stubs/image.pyi` 的 `blob` 类）。

### 多色追踪（多组阈值）

```python
RED   = [(30, 100, 15, 127, 15, 127)]
GREEN = [(30, 100, -128, -15, 15, 127)]
THRESHOLDS = RED + GREEN

for blob in img.find_blobs(THRESHOLDS, pixels_threshold=80, area_threshold=80):
    # blob.code() 的位指示命中的是哪一组阈值
    img.draw_rectangle(blob.rect(), color=(255, 255, 0))
```

## 常见错误

| 错误 | 原因 | 解决方法 |
|---|---|---|
| 检测不到目标 | LAB 阈值范围不对 | 先用 `img.get_statistics()` 在目标区域采样，按统计量整定阈值 |
| 满屏噪点 | 阈值过宽 / 未过滤 | 收紧阈值；提高 `pixels_threshold` / `area_threshold` |
| 把 `blob.rect` 当方法 | `blob` 的方法是 `rect()` | `blob.rect()`（圆括号）；注意 `apriltag` 的 `rect` 才是字段 |
| 检测结果闪烁、不稳定 | 光照变化 / 阈值边界 | 调 `merge=True`；先 `gaussian(1)` 降噪 |

## 参考

- `example/02-Image-Processing/02-Color-Tracking/find_blobs.py`
- `example/02-Image-Processing/02-Color-Tracking/statistics.py`
- `docs/zh_CN/api-reference/image.rst`
- `docs/zh_CN/concepts/image-processing.rst`（阈值化与分割）
