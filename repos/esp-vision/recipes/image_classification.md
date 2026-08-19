# 图像分类（ImageNetCls）

> **适用摘要**: 用 `espdl.ImageNetCls` 对图像做分类，返回最多 `topk` 个 `(label, score)`；以文件图像分类为例，必要时先 `to_rgb565(copy=True)` 转换格式。

> Version: selected repo/component version; align with the user project and dependency manifest.
> Evidence: `repos/esp-vision/resources/`, source/examples in `repos/esp-vision/`, and this recipe path `repos/esp-vision/recipes/image_classification.md`.
> Validation: draft metadata added from repository routing; verify APIs, Kconfig symbols, and component dependencies against the selected version.

## 触发意图

- "图像分类"
- "ImageNet 分类"
- "猫狗识别"
- "espdl.ImageNetCls"

## 前置条件

| 条件 | 要求 |
|---|---|
| 参考示例 | `example/03-Machine-Learning/00-ESP-DL/imagenet_cls.py` |
| 模型文件 | `/sdcard/imagenet_cls_mobilenetv2_s8_v1.espdl` |
| 待分类图像 | `/sdcard/cat.jpg`（示例） |

## 分步说明

分类结果按模型输出顺序返回最多 `topk` 个 `(label, score)` 元组。文件图像需确保输入格式可被封装层接受（必要时转 RGB565）。

```python
import espdl
import image

MODEL = "/sdcard/imagenet_cls_mobilenetv2_s8_v1.espdl"
IMAGE = "/sdcard/cat.jpg"

img = image.Image(IMAGE).to_rgb565(copy=True)
cls = espdl.ImageNetCls(MODEL, topk=5, score=0.0)

try:
    for name, score in cls.classify(img):
        print(name, "%.4f" % score)
finally:
    cls.deinit()

print("done")
```

参数与结果（来源 `stubs/espdl.pyi`）：
- 构造：`ImageNetCls(path, *, topk=5, score=None, mean=None, std=None, softmax=True)`。
- `classify(img)` 返回 `list[(label, score)]`。
- `set_thresholds(topk=None, score=None)` 运行时调整。

### 对实时采集帧分类

```python
import espdl, sensor

sensor.reset()
sensor.set_pixformat(sensor.RGB565)
sensor.set_framesize(sensor.QVGA)
sensor.skip_frames(time=1000)

cls = espdl.ImageNetCls("/sdcard/imagenet_cls_mobilenetv2_s8_v1.espdl", topk=3)
try:
    while True:
        img = sensor.snapshot()
        for label, score in cls.classify(img):
            img.draw_string(2, 2, "%s:%.2f" % (label, score), color=(255, 255, 255))
            break   # 只显示 top-1
        img.flush()
finally:
    cls.deinit()
```

> 可选性能分析：用 `espdl.load_model(path, profile=True)` 输出 ESP-DL 性能信息（见 `docs/zh_CN/how-to/add-model.rst`）。

## 常见错误

| 错误 | 原因 | 解决方法 |
|---|---|---|
| 文件图格式不被接受 | 直接用 JPEG/PNG 传入 | `.to_rgb565(copy=True)` 转换 |
| 标签乱码或为索引 | 模型未带 label 映射 | 以模型自带 label 为准；分类返回的 `label` 由封装层提供 |
| 全部分数接近 0 | 预处理不匹配（mean/std） | 构造函数传正确 `mean` / `std` |
| 模型加载失败 | 路径/芯片不匹配 | 确认 `.espdl` 与目标芯片（P4/S3/S31）匹配 |

## 参考

- `example/03-Machine-Learning/00-ESP-DL/imagenet_cls.py`
- `docs/zh_CN/api-reference/espdl.rst`
- `docs/zh_CN/how-to/add-model.rst`
- `stubs/espdl.pyi`
