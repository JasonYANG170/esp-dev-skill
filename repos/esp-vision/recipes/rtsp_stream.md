# RTSP 推流（仅 ESP32-P4）

> **适用摘要**: 起以太网（P4-Function-EV-Board 的 IP101 RMII PHY），用 `h264.H264Encoder` 编码、`rtsp.RTSPServer` 通过 RTSP 推送 H.264，VLC/ffplay 在 `rtsp://<board-ip>:8554/` 观看。`rtsp` 仅 ESP32-P4 构建。

> Version: selected repo/component version; align with the user project and dependency manifest.
> Evidence: `repos/esp-vision/resources/`, source/examples in `repos/esp-vision/`, and this recipe path `repos/esp-vision/recipes/rtsp_stream.md`.
> Validation: draft metadata added from repository routing; verify APIs, Kconfig symbols, and component dependencies against the selected version.

## 触发意图

- "RTSP 推流"
- "网络视频流"
- "VLC 看摄像头"
- "rtsp.RTSPServer"

## 前置条件

| 条件 | 要求 |
|---|---|
| 参考示例 | `example/01-Camera/02-RTSP/stream_rtsp.py` |
| 芯片 | ESP32-P4（`h264` + `rtsp` 仅 P4） |
| 网络 | 带以太网 PHY 的板（示例引脚对应 ESP32-P4-Function-EV-Board，IP101 over RMII） |
| 权威签名 | `stubs/rtsp.pyi` |

## 分步说明

```python
import network
import time
from machine import Pin

import sensor
import h264
import rtsp

WIDTH, HEIGHT, FPS = 320, 240, 30

# 1. 起以太网（IP101 over RMII；引脚按 P4-Function-EV-Board）
lan = network.LAN(
    mdc=Pin(31),
    mdio=Pin(52),
    reset=Pin(51),
    phy_addr=1,
    phy_type=network.PHY_IP101,
)
lan.active(True)

print("waiting for ethernet link...")
for _ in range(100):           # 最多约 10s
    if lan.isconnected():
        break
    time.sleep_ms(100)

if not lan.isconnected():
    raise OSError("ethernet not connected - check cable / DHCP / RMII clock")

print("RTSP at rtsp://%s:8554/" % lan.ifconfig()[0])

# 2. 相机 + 编码器 + RTSP 服务器
sensor.reset()
sensor.set_pixformat(sensor.RGB565)
sensor.set_framesize(sensor.QVGA)
sensor.skip_frames(time=1000)

enc = h264.H264Encoder(WIDTH, HEIGHT, fps=FPS, bitrate=3_000_000, qp_max=32)
server = rtsp.RTSPServer(WIDTH, HEIGHT, fps=FPS, listen_port=8554)

# 3. 采集 -> (可选绘制) -> 编码 -> 推送
try:
    while True:
        img = sensor.snapshot()
        server.send(enc.encode(img))
finally:
    server.stop()
    enc.close()
```

主机播放：
```bash
ffplay rtsp://<board-ip>:8554/   # 或 VLC / PotPlayer 打开该 URL
```

`RTSPServer` 构造与用法（来源 `stubs/rtsp.pyi`）：
- `RTSPServer(width, height, *, fps=15, listen_port=8554, max_frame_len=0)`。`max_frame_len=0` 用 `width*height` 作为安全上限。
- `send(nal)`：把一帧 `H264Encoder.encode()` 的结果入队，按序发送；客户端慢或缺席时整帧丢弃，不阻塞调用；超过 `max_frame_len` 的帧被忽略。
- `stop()`：停止服务、join 其任务并释放队列帧。释放网络/编码器前必须调用。

要点：
- `RTSPServer` 公布的宽/高/帧率必须与编码器配置一致。
- 仅视频、无音频。
- 其它产品板的网络 PHY 引脚不同，应使用对应的 `network.LAN(...)` 或 `network.WLAN(...)` 配置。

## 常见错误

| 错误 | 原因 | 解决方法 |
|---|---|---|
| `import rtsp` 失败 | 非 P4 板 | `rtsp` 仅 ESP32-P4；S3 改用 MJPEG |
| 客户端连不上 | 以太网未连/端口被占 | 确认 `lan.isconnected()` 与 IP；检查 `listen_port` |
| 画面花屏/卡顿 | 丢 P 帧后到下个关键帧才恢复 | 缩短 `gop`；降码率；必要时调大 `max_frame_len` |
| 资源泄漏 | 异常时未 `stop()`/`close()` | `try/finally` 中清理 |
| PHY 引脚不对 | 用了错误的板配置 | 按开发板原理图设 `mdc/mdio/reset/phy_addr/phy_type` |

## 参考

- `example/01-Camera/02-RTSP/stream_rtsp.py`
- `example/01-Camera/03-MJPEG/wifi_mjpeg_stream.py`（无 H.264 时的替代方案）
- `docs/zh_CN/api-reference/rtsp.rst`
- `docs/zh_CN/concepts/codec-streaming.rst`
- `stubs/rtsp.pyi`
