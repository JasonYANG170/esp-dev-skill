# 二维码 / 条码 / AprilTag 检测

> **适用摘要**: 用 `find_qrcodes` / `find_barcodes`（P4 ZXing）/ `find_apriltags` 检测并解码标记，绘制框与角点。灰度输入通常可降低处理开销。

> Version: selected repo/component version; align with the user project and dependency manifest.
> Evidence: `repos/esp-vision/resources/`, source/examples in `repos/esp-vision/`, and this recipe path `repos/esp-vision/recipes/qrcode_barcode_apriltag.md`.
> Validation: draft metadata added from repository routing; verify APIs, Kconfig symbols, and component dependencies against the selected version.

## 触发意图

- "二维码识别"
- "扫描 QR code"
- "1D 条码 / barcode"
- "AprilTag 定位"
- "fiducial marker"

## 前置条件

| 条件 | 要求 |
|---|---|
| 参考示例 | `example/04-Barcodes/find_qrcodes.py`、`example/05-Feature-Detection/find_barcodes.py`、`example/05-Feature-Detection/find_apriltags.py` |
| 条码（ZXing-C++） | 仅 ESP32-P4 板级 `board.cmake` 启用 `ESP_VISION_ENABLE_BARCODE` |
| AprilTag | 板级 `imlib_config.h` 启用 `IMLIB_ENABLE_APRILTAGS` 及具体家族（如 `TAG36H11`） |

## 分步说明

### 二维码（通用，灰度）

```python
import sensor
import time

sensor.reset()
sensor.set_pixformat(sensor.GRAYSCALE)
sensor.set_framesize(sensor.QVGA)
sensor.skip_frames(time=1000)

while True:
    img = sensor.snapshot()
    codes = img.find_qrcodes()
    for code in codes:
        img.draw_rectangle(code.rect(), color=255, thickness=2)
        print(code.payload())
    img.flush()
    time.sleep_ms(20)
```

`qrcode` 方法（见 `stubs/image.pyi`）：`rect()`、`payload()`、`version()`、`ecc_level()`、`data_type()`、`is_numeric()` 等。如果二维码只出现在固定区域，可指定较小 ROI 降低计算量。

### 1D/2D 条码（仅 ESP32-P4）

```python
sensor.reset()
sensor.set_pixformat(sensor.GRAYSCALE)
sensor.set_framesize(sensor.QVGA)
sensor.skip_frames(time=1000)

while True:
    img = sensor.snapshot()
    codes = img.find_barcodes()
    for code in codes:
        img.draw_rectangle(code.rect(), color=255, thickness=2)
        img.draw_string(code.x(), max(0, code.y() - 12), code.payload(), color=255)
        print("barcode:", code.payload(), "type:", code.type(), "rect:", code.rect())
    img.flush()
    time.sleep_ms(20)
```

`barcode.type()` 返回条码符号常量（`image.EAN13`、`image.CODE128`、`image.UPCA`、`image.PDF417` 等，见 `stubs/image.pyi`）。

### AprilTag（注意属性是字段，不是方法）

```python
import image
import sensor
import time

sensor.reset()
sensor.set_pixformat(sensor.GRAYSCALE)
sensor.set_framesize(sensor.QQVGA)      # AprilTag 常用较小分辨率省算力
sensor.skip_frames(time=1000)

while True:
    img = sensor.snapshot()
    tags = img.find_apriltags(families=image.TAG36H11, pose=False)

    for tag in tags:
        img.draw_rectangle(tag.rect, color=255, thickness=2)
        for x, y in tag.corners:
            img.draw_cross(x, y, color=255, size=5, thickness=1)
        img.draw_string(tag.x, max(0, tag.y - 10), "%s:%d" % (tag.name, tag.id), color=255)
        print("tag:", tag.name, "id:", tag.id, "margin:", tag.decision_margin)

    img.flush()
    time.sleep_ms(20)
```

`apriltag` 的 `rect`、`corners`、`x`、`y`、`id`、`name`、`cx`、`cy`、`decision_margin`、`hamming` 都是**字段**（直接访问，不加括号）。提供相机内参时设 `pose=True` 可得到 `x_translation` / `x_rotation` 等 6 自由度位姿字段。

## 常见错误

| 错误 | 原因 | 解决方法 |
|---|---|---|
| `find_barcodes` ImportError/异常 | 非 P4 板，或未启用 `ESP_VISION_ENABLE_BARCODE` | 仅 ESP32-P4 可用；S3 改用 `find_qrcodes` |
| `find_apriltags` 异常 | `imlib_config.h` 未启用 `IMLIB_ENABLE_APRILTAGS` 或该家族 | 检查并启用对应 `IMLIB_ENABLE_APRILTAGS_TAG36H11` 等 |
| `tag.id()` 报错 | `apriltag` 属性是字段 | `tag.id`（不加括号）；qrcode/barcode 才是方法 |
| 识别率低 | 分辨率过大/过小、光照差 | 用 `QQVGA`；先 `gaussian(1)` 降噪；调整距离 |

## 参考

- `example/04-Barcodes/find_qrcodes.py`
- `example/05-Feature-Detection/find_barcodes.py`
- `example/05-Feature-Detection/find_apriltags.py`
- `docs/zh_CN/api-reference/image.rst`
- `docs/zh_CN/concepts/image-processing.rst`（特征检测段）
- `stubs/image.pyi`（`qrcode` / `barcode` / `apriltag` 类）
