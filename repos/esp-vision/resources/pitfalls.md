# ESP-VISION 避坑汇总

> 来源：仓库 `docs/zh_CN/`（API 参考、概念、how-to）、`stubs/*.pyi`、`example/`。下列为高频错误与正确写法对照。SKILL.md 的“Critical Pitfalls”为此文件的精简版。

## 1. 缺 `skip_frames` → 前几帧偏色/过曝

配置后必须丢弃若干帧让 AE/AWB 收敛。
```python
# ✅
sensor.reset(); sensor.set_pixformat(sensor.RGB565); sensor.set_framesize(sensor.QVGA)
sensor.skip_frames(time=1000)   # 或 sensor.skip_frames(n=3)
```

## 2. `snapshot()` 返回可复用帧缓冲 → 跨帧使用前未 copy

下一帧采集会覆盖当前帧缓冲。背景帧、缓存帧、传给线程/异步的帧都要 `.copy()`。
```python
# ✅ 帧差分背景
background = sensor.snapshot().copy()
while True:
    img = sensor.snapshot()
    img.difference(background)
```

## 3. 仅两档像素格式 / 两档分辨率常量

`set_pixformat` 只接受 `sensor.GRAYSCALE` / `sensor.RGB565`；`set_framesize` 只接受 `sensor.QQVGA` / `sensor.QVGA`（来源 `stubs/sensor.pyi`）。不要假设 OpenMV 全集（如 `JPEG`/`VGA`/`HQVGA`）可用。

## 4. `h264` / `rtsp` 仅 ESP32-P4

`micropython.cmake` 仅在 `IDF_TARGET == esp32p4` 时加入这两个模块。S3/S31 上 `import h264` 会失败，改用 MJPEG（JPEG 流），见 `example/01-Camera/03-MJPEG`。

## 5. 调用未启用的 imlib 算法

`boards/<BOARD>/imlib_config.h` 的 `IMLIB_ENABLE_*` 决定哪些方法可用；未启用调用时抛异常。`OMV_NO_GPL` 必须保留（排除 GPL 路径，如 `agast.c`）。P4X_EYE 默认启用了上述滤波/几何/标记算法，其它板可能不同。

## 6. ESP-DL 模型逐帧重建

模型权重/激活在 PSRAM，逐帧构造会极慢并耗尽内存。模型只在循环外构造一次，`finally` 中 `deinit()`。
```python
det = espdl.ESPDet(MODEL, score=0.5, nms=0.7)
try:
    while True:
        for x,y,w,h,score,cat in det.detect(sensor.snapshot()): ...
finally:
    det.deinit()
```

## 7. H.264 编码器尺寸与输入帧不一致

`H264Encoder(width, height, ...)` 后每帧 `encode()` 的图像尺寸必须一致。分辨率变了要重建编码器。用首帧尺寸构造最稳妥。

## 8. 资源未在 `finally` 释放

`espdl.*.deinit()`、`h264.H264Encoder.close()`、`rtsp.RTSPServer.stop()`、`image.ImageIO.close()`、文件 `close()` 都应在 `finally` 调用。`RTSPServer` 释放网络/编码器前必须 `stop()`。

## 9. `display.Display()` 重复创建

显示对象只创建一次并复用；循环里反复构造会重复初始化屏幕资源。`deinit()` 后不得再 `write()`。

## 10. AprilTag 属性是字段，不是方法

`apriltag` 的 `rect`/`corners`/`x`/`y`/`id`/`name`/`cx`/`cy`/`decision_margin`/`hamming`/`goodness`（以及 `pose=True` 时的 `*_translation`/`*_rotation`）是字段，直接访问。`qrcode`/`barcode`/`blob`/`line`/`circle`/`rect` 才是方法（带括号）。

## 11. `boot.py` 放死循环 → 阻塞 USB REPL

`boot.py` 必须返回，只做短时确定性初始化（如启动网络）。ESP-VISION 在 `boot.py` 完成后才初始化 USB 设备，阻塞会导致 USB REPL 与主机工具不可用。应用循环放 `/main.py`。

## 12. 以 CPython 文档判断 MicroPython 行为

ESP-VISION 固定 MicroPython v1.28.0，应以 `https://docs.micropython.org/en/v1.28.0/` 为准。测量间隔用 `time.ticks_ms()` + `time.ticks_diff()`（处理回绕），而非直接相减。

## 13. 设备无法启动（坏 `main.py` / `boot.py`）

REPL 连接后 `Ctrl-C`，重命名/删除启动文件再软复位：
```python
import os
os.rename("/main.py", "/main.disabled.py")
# Ctrl-D 软复位
```
不行则 `idf.py --board <BOARD> -p <PORT> erase-flash` 后重烧（注意会清空全部 Flash 含文件系统与用户数据）。不 `erase-flash` 的重烧通常保留数据文件系统，可能也保留了坏启动文件。

## 14. RTSP 客户端慢/缺席

`RTSPServer.send()` 不阻塞，慢客户端整帧丢弃以避免阻塞采集循环；帧超过 `max_frame_len` 被忽略。丢 P 帧会花屏直到下一个关键帧（IDR/I）。必要时降码率/分辨率或调大 `max_frame_len`。

## 15. 并发访问视觉对象

`asyncio` task 是协作调度，仅在 `await` 时切换；`sensor.snapshot()`、推理、`img.flush()` 是同步操作，执行时其它协程不能运行——每帧后 `await asyncio.sleep_ms(0)`。除非 API 明确说明线程安全，摄像头/显示/编解码器/模型/帧缓冲应始终由同一线程/协程持有；多线程用 `_thread.allocate_lock()`，IRQ/线程用 `asyncio.ThreadSafeFlag` 通知事件循环。

## 16. 模块导入失败但不确定是否可用

REPL 执行 `help("modules")` 与 `help(<module>)` 确认可用性。标准 MicroPython API 可用性由 `overlay/micropython/ports/esp32/mpconfigport.h` + 板级 `mpconfigboard.h`/`mpconfigboard.cmake` + IDF 版本 + SoC 能力宏共同决定，跨板脚本用 `hasattr()` 或受保护 import。
