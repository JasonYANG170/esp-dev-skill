# 首个摄像头脚本

> **适用摘要**: 复位摄像头、选择像素格式与分辨率、等待自动曝光/白平衡稳定后连续采集，并通过 `img.flush()` 在主机预览。

## 触发意图

- "摄像头采集"
- "第一个 ESP-VISION 脚本"
- "sensor.snapshot 怎么用"
- "怎么预览画面"
- "OpenMV 脚本移植到 ESP-VISION"

## 前置条件

| 条件 | 要求 |
|---|---|
| 参考示例 | `example/00-HelloWorld/helloworld.py` |
| 固件 | 已按所选开发板烧录 ESP-VISION 固件 |
| 主机预览 | VSCode 扩展 / Web IDE，或串口 REPL（`img.flush()` 经 USB CDC 推送 EVFRAME JPEG） |

## 分步说明

ESP-VISION 的 `sensor` 模块接口与 OpenMV 一致。典型流程：复位 → 选像素格式与分辨率 → 丢弃若干帧等稳定 → 连续采集。

```python
import sensor
import time

sensor.reset()
sensor.set_pixformat(sensor.RGB565)   # 显示、绘图与多数 AI 工作流用 RGB565；GRAYSCALE 省内存
sensor.set_framesize(sensor.QVGA)     # 仅 QQVGA / QVGA 可选
sensor.skip_frames(time=1000)         # 等待 AE/AWB 收敛，避免前几帧偏色

while True:
    img = sensor.snapshot()
    img.flush()                       # 推到主机预览
    time.sleep_ms(20)
```

要点：
- `sensor.snapshot()` 返回由**可复用帧缓冲**承载的 `image.Image`，下一帧采集会覆盖它。需要在采集后继续使用该帧时调 `.copy()`。
- `RGB565` 适合显示与绘图；`GRAYSCALE` 可降低许多检测算法的内存与处理开销。
- 像素格式常量只有 `sensor.GRAYSCALE` / `sensor.RGB565`；帧尺寸常量只有 `sensor.QQVGA` / `sensor.QVGA`（来源 `stubs/sensor.pyi`）。

### 把脚本设为产品入口

```python
# /main.py
import sys
import my_app

try:
    my_app.main()
except KeyboardInterrupt:
    raise            # Ctrl-C 进友好 REPL
except Exception as error:
    print("Fatal application error:")
    sys.print_exception(error)
```

> `boot.py` 只做短时确定性初始化（如启动网络），**不能**放死循环，否则 USB REPL 与主机工具不可用。

## 常见错误

| 错误 | 原因 | 解决方法 |
|---|---|---|
| 前几帧偏色/过曝 | 未调用 `skip_frames` | 配置后 `sensor.skip_frames(time=1000)` |
| `set_pixformat(sensor.JPEG)` 报错 | 该常量不存在 | 仅 `GRAYSCALE` / `RGB565` |
| `set_framesize(sensor.VGA)` 报错 | 该常量不存在 | 仅 `QQVGA` / `QVGA` |
| 跨帧引用 `snapshot()` 内容错乱 | 帧缓冲可复用，被覆盖 | 需要 `.copy()` |
| 主机看不到预览 | 未调用 `img.flush()` | 循环里加 `img.flush()` |

## 参考

- `example/00-HelloWorld/helloworld.py`
- `docs/zh_CN/api-reference/sensor.rst`
- `docs/zh_CN/concepts/camera-pipeline.rst`
