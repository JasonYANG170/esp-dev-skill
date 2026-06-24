# 目标检测（ESPDet / YOLO11）

> **适用摘要**: 用 ESP-DL 的 `ESPDet` / `YOLO11` 封装类在采集帧上做目标检测，绘制检测框，并在运行时调整 `score` / `nms` 阈值。模型只加载一次、跨帧复用。

## 触发意图

- "目标检测"
- "人脸检测"
- "YOLO 检测"
- "ESP-DL 推理"
- "espdl.ESPDet / YOLO11"

## 前置条件

| 条件 | 要求 |
|---|---|
| 参考示例 | `example/03-Machine-Learning/00-ESP-DL/espdet_pico.py`、`yolo11.py` |
| 模型文件 | `.espdl` 模型放在 `/sdcard/...` 或 `/flash/...`（运行时加载，不编入固件） |
| 像素格式 | `RGB565`（检测器期望 RGB） |

## 分步说明

ESP-VISION 用面向任务的类封装 ESP-DL；检测坐标会映射回输入图像，可直接用于绘图。

### ESPDet（人脸/小目标检测）

```python
import espdl
import sensor
import time

MODEL = "/sdcard/espdet_pico_224_224_face.espdl"

sensor.reset()
sensor.set_pixformat(sensor.RGB565)
sensor.set_framesize(sensor.QVGA)
sensor.skip_frames(time=1000)

det = espdl.ESPDet(MODEL, score=0.5, nms=0.7)

try:
    while True:
        img = sensor.snapshot()
        for x, y, w, h, score, category in det.detect(img):
            img.draw_rectangle(x, y, w, h, color=(255, 0, 0), thickness=2)
            img.draw_string(x, max(0, y - 12), "%.2f:%d" % (score, category), color=(255, 0, 0))
        img.flush()
        time.sleep_ms(20)
finally:
    det.deinit()
```

### YOLO11（COCO 目标检测）

```python
import espdl
import sensor
import time

MODEL = "/sdcard/espdet_yolo11n_160_160_coco.espdl"

sensor.reset()
sensor.set_pixformat(sensor.RGB565)
sensor.set_framesize(sensor.QVGA)
sensor.skip_frames(time=1000)

det = espdl.YOLO11(MODEL, score=0.35, nms=0.7, topk=10)   # topk 限制每帧结果数

try:
    while True:
        img = sensor.snapshot()
        for x, y, w, h, score, category in det.detect(img):
            img.draw_rectangle(x, y, w, h, color=(255, 0, 0), thickness=2)
            img.draw_string(x, max(0, y - 12), "%.2f:%d" % (score, category), color=(255, 0, 0))
        img.flush()
        time.sleep_ms(20)
finally:
    det.deinit()
```

### 运行时调整阈值（无需重新加载模型）

```python
det.set_thresholds(score=0.65, nms=0.6)
```

参数与结果（来源 `stubs/espdl.pyi`）：
- 构造参数：`path`、`score`（置信度阈值）、`nms`（非极大值抑制阈值）、`topk`（仅 YOLO11，每帧最大结果数，默认 10）、可选 `mean` / `std`（RGB 预处理）。
- `detect(img)` 返回 `list[(x, y, w, h, score, category)]`。
- `set_thresholds(score=None, nms=None)`：传 `None` 保留当前值，返回 `self`。

## 常见错误

| 错误 | 原因 | 解决方法 |
|---|---|---|
| 逐帧重建模型导致极慢/OOM | 每次构造都加载权重到 PSRAM | 模型只在循环外构造一次，`finally` 中 `deinit()` |
| 模型路径找不到 | 文件未放到存储 | 确认 `/sdcard` 挂载并拷贝 `.espdl`；或放 `/flash` |
| 预处理不匹配 | 模型需要特定 mean/std | 构造函数传 `mean=(...)`、`std=(...)` |
| 漏检/误检多 | 阈值不当 | `set_thresholds(score=, nms=)` 动态整定 |
| 帧率低 | 输入分辨率大、模型大 | 降 `set_framesize`；用更小模型（如 `espdet_pico`） |

## 参考

- `example/03-Machine-Learning/00-ESP-DL/espdet_pico.py`
- `example/03-Machine-Learning/00-ESP-DL/yolo11.py`
- `docs/zh_CN/api-reference/espdl.rst`
- `docs/zh_CN/concepts/ai-inference.rst`
- `docs/zh_CN/how-to/add-model.rst`
- `stubs/espdl.pyi`
