# 图像滤波与二值化

> **适用摘要**: 用邻域/秩/保边滤波降噪或增强边缘，配合 `binary` 做阈值分割与形态学清理。

> Version: selected repo/component version; align with the user project and dependency manifest.
> Evidence: `repos/esp-vision/resources/`, source/examples in `repos/esp-vision/`, and this recipe path `repos/esp-vision/recipes/image_filters.md`.
> Validation: draft metadata added from repository routing; verify APIs, Kconfig symbols, and component dependencies against the selected version.

## 触发意图

- "图像滤波"
- "高斯模糊 / 中值滤波 / 双边滤波"
- "二值化阈值"
- "腐蚀膨胀"
- "直方图均衡 histeq"

## 前置条件

| 条件 | 要求 |
|---|---|
| 参考示例 | `example/02-Image-Processing/01-Image-Filters/filters.py`、`binary_ops.py`、`morphology.py` |
| 板级配置 | `boards/<BOARD>/imlib_config.h` 已 `#define IMLIB_ENABLE_*` 对应算法（P4X_EYE 默认启用 mean/median/mode/midpoint/morph/gaussian/laplacian/bilateral） |

## 分步说明

滤波方法在 `(2*ksize+1)` 方形邻域上运算，**原地修改**图像，多数返回 `self` 可链式调用。

```python
import sensor
import time

OPS = ("mean", "median", "mode", "midpoint", "gaussian", "laplacian", "bilateral", "histeq")

sensor.reset()
sensor.set_pixformat(sensor.GRAYSCALE)
sensor.set_framesize(sensor.QVGA)
sensor.skip_frames(time=1000)

frame = 0
while True:
    name = OPS[(frame // 30) % len(OPS)]
    img = sensor.snapshot()

    if name == "mean":
        img.mean(1)
    elif name == "median":
        img.median(1)
    elif name == "mode":
        img.mode(1)
    elif name == "midpoint":
        img.midpoint(1)
    elif name == "gaussian":
        img.gaussian(1)
    elif name == "laplacian":
        img.laplacian(1)
    elif name == "bilateral":
        img.bilateral(1)
    elif name == "histeq":
        img.histeq()

    img.draw_string(2, 2, name, color=255)
    img.flush()
    frame += 1
    time.sleep_ms(20)
```

各滤波用途（来源 `docs/zh_CN/concepts/image-processing.rst`）：
- `mean` — 盒式平均，快速模糊。
- `gaussian(ksize, unsharp=False)` — 二项式近似高斯降噪；`unsharp=True` 改为锐化。
- `laplacian(ksize, sharpen=False)` — 二阶导数边缘响应；`sharpen=True` 叠加回原图增强边缘。
- `median` — 取百分位（默认 0.5），对椒盐噪声效果极佳。
- `mode` — 取最常见值。
- `midpoint(bias=)` — 在最小最大值间按 `bias` 混合。
- `bilateral(ksize, color_sigma=, space_sigma=)` — 按空间距离与颜色相似度加权，保边平滑。
- `morph(ksize, kernel, mul=, add=)` — 任意整数卷积核。

### 滤波 + 二值化 + 形态学（典型分割流水线）

```python
sensor.set_pixformat(sensor.GRAYSCALE)
img = sensor.snapshot()
img.gaussian(1)        # 阈值前抑制高频噪声
img.binary([(80, 255)])
img.erode(1)           # 去孤立像素
img.dilate(1)          # 恢复主体区域
img.flush()
```

`binary(thresholds)` 把匹配阈值（灰度为 `(min, max)`）的像素转为有效二值。形态学还有 `open()`（先腐蚀再膨胀，去小噪点）与 `close()`（先膨胀再腐蚀，填小孔）。

### 运行时自动整定阈值

```python
img = sensor.snapshot()
hist = img.get_histogram()
stat = img.get_statistics()
print("mean:", stat.mean(), "stdev:", stat.stdev())
thr = hist.get_threshold()    # Otsu 阈值
img.binary([(thr.value(), 255)])
img.flush()
```

## 常见错误

| 错误 | 原因 | 解决方法 |
|---|---|---|
| 调用滤波抛异常 | 该算法未在 `imlib_config.h` 启用 | 检查 `boards/<BOARD>/imlib_config.h` 的 `IMLIB_ENABLE_*` |
| 链式调用报错 | 某些方法在特定 pixformat 下不可用 | 灰度滤波需先 `to_grayscale()` 或直接采集 `GRAYSCALE` |
| 二值化后全黑/全白 | 阈值范围不对 | 用 `get_statistics()` 整定；或 Otsu `get_threshold()` |
| 帧率太低 | ksize 过大 | 多数场景 `ksize=1` 已足够 |

## 参考

- `example/02-Image-Processing/01-Image-Filters/filters.py`
- `example/02-Image-Processing/01-Image-Filters/binary_ops.py`
- `example/02-Image-Processing/01-Image-Filters/morphology.py`
- `docs/zh_CN/concepts/image-processing.rst`
- `boards/ESP32_P4X_EYE/imlib_config.h`
