# 板载 LCD 显示

> **适用摘要**: 创建 `display.Display()`（即 `ESP32Display`）并把采集帧用 `write()` 送到屏幕，支持 `fit` 缩放、显式 `x_scale`/`y_scale`、`roi` 裁剪与背光调节。显示对象只创建一次并复用。

> Version: selected repo/component version; align with the user project and dependency manifest.
> Evidence: `repos/esp-vision/resources/`, source/examples in `repos/esp-vision/`, and this recipe path `repos/esp-vision/recipes/lcd_display.md`.
> Validation: draft metadata added from repository routing; verify APIs, Kconfig symbols, and component dependencies against the selected version.

## 触发意图

- "LCD 显示"
- "屏幕显示画面"
- "display.write"
- "画面缩放/裁剪显示"
- "调节背光"

## 前置条件

| 条件 | 要求 |
|---|---|
| 参考示例 | `example/06-Peripherals/01-Display/lcd_preview.py` |
| 权威签名 | `stubs/display.pyi` |
| 硬件 | 开发板带 LCD（如 ESP32-P4X-EYE / ESP32-S3-EYE） |

## 分步说明

`display.Display()` 创建并初始化屏幕（`ESP32Display` 的别名）。用 `write(image, ...)` 送帧。

### 基本预览（fit 缩放）

```python
import display
import sensor
import time

lcd = display.Display()

sensor.reset()
sensor.set_pixformat(sensor.RGB565)
sensor.set_framesize(sensor.QVGA)
sensor.skip_frames(time=1000)

while True:
    img = sensor.snapshot()
    lcd.write(img)            # 默认 fit=True，自动缩放适配屏幕
    time.sleep_ms(20)
```

### 定位 / 显式缩放 / ROI 裁剪

```python
img = sensor.snapshot()
lcd.write(
    img,
    x=20,                     # 目标左上 x（显示像素）
    y=10,                     # 目标左上 y
    x_scale=0.5,              # 显式水平缩放
    y_scale=0.5,              # 显式垂直缩放
    roi=(40, 30, 160, 120),   # 源图像区域 (x, y, w, h)
    fit=False,                # 由显式位置/缩放控制
)
```

`fit=True` 时源图像缩放适配屏幕并沿用默认放置；`fit=False` 时由 `x`/`y`/`x_scale`/`y_scale`/`roi` 控制。ROI 用源图像坐标，缩小 ROI 还能减少送入显示链路的数据量。

### 背光与资源释放

```python
lcd = display.Display(backlight=80)   # 0-100 百分比
print("panel:", lcd.width(), "x", lcd.height())
lcd.backlight(30)                     # 设置背光
lcd.clear()                           # 清屏
lcd.deinit()                          # 永久释放显示设备；之后不得再 write
```

`backlight(value=None)`：传 `None` 返回当前亮度，否则设置 0-100。应用永久释放显示设备时调 `deinit()`。

## 常见错误

| 错误 | 原因 | 解决方法 |
|---|---|---|
| 屏幕反复初始化闪烁 | 循环里反复 `display.Display()` | 显示对象只创建一次并复用 |
| `deinit()` 后再写入报错 | 驱动已反初始化 | 不要在 `deinit()` 后 `write` |
| 画面比例失调 | `fit=False` 且缩放比例不当 | 用 `fit=True` 或设等比 `x_scale`/`y_scale` |
| 只显示局部 | ROI 设得过小 | 调整 `roi` 或去掉 `roi` |
| 背光没变化 | 数值越界 | 限制 0-100 |

## 参考

- `example/06-Peripherals/01-Display/lcd_preview.py`
- `docs/zh_CN/api-reference/display.rst`
- `stubs/display.pyi`
