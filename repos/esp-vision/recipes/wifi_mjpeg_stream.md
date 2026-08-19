# Wi-Fi MJPEG 视频流推送

> **适用摘要**: 把采集帧逐帧 JPEG 编码，用板载 HTTP 服务器以 `multipart/x-mixed-replace` 推 MJPEG 流，浏览器打开 `http://<board-ip>/` 即可观看。这是无 H.264 硬件（ESP32-S3 等）时唯一的实时视频推流路径。

> Version: selected repo/component version; align with the user project and dependency manifest.
> Evidence: `repos/esp-vision/resources/`, source/examples in `repos/esp-vision/`, and this recipe path `repos/esp-vision/recipes/wifi_mjpeg_stream.md`.
> Validation: draft metadata added from repository routing; verify APIs, Kconfig symbols, and component dependencies against the selected version.

## 触发意图

- "Wi-Fi 视频流"
- "MJPEG 推流"
- "浏览器看摄像头"
- "S3 怎么推流"
- "HTTP multipart JPEG"

## 前置条件

| 条件 | 要求 |
|---|---|
| 参考示例 | `example/01-Camera/03-MJPEG/wifi_mjpeg_stream.py` |
| 网络 | Wi-Fi STA 可用（`network.WLAN`），即 ESP32-S3 / 带Wi-Fi 的板 |
| 概念文档 | `docs/zh_CN/concepts/codec-streaming.rst`（JPEG 与流传输） |
| 权威签名 | `stubs/image.pyi`（`to_jpeg` / `bytearray`） |

## 分步说明

非 H.264 的板（ESP32-S3 / S31）没有 `h264`/`rtsp` 模块。用 `image.Image.to_jpeg(quality=)` 把每帧编码为 JPEG，再按 HTTP `multipart/x-mixed-replace` 帧格式逐帧写出，浏览器 `<img>` 标签即可显示。`to_jpeg` 在有硬件 JPEG 的板上走硬件，否则走软件编码器 `esp_new_jpeg`（来源 `codec-streaming.rst`）。

```python
import network
import socket
import time

import sensor

SSID = "your-ssid"
PASSWORD = "your-wifi-password"

PORT = 80
JPEG_QUALITY = 35
FRAME_DELAY_MS = 50
CONNECT_TIMEOUT_S = 15
CONNECT_RETRY = 3
BOUNDARY = b"espvision"


def connect_wifi():
    wlan = network.WLAN(network.STA_IF)
    for attempt in range(1, CONNECT_RETRY + 1):
        print("connecting:", SSID, "attempt:", attempt)
        try:
            wlan.active(False)
            time.sleep_ms(300)
        except OSError:
            pass
        wlan.active(True)
        try:
            wlan.connect(SSID, PASSWORD)
        except OSError as exc:
            print("connect failed:", exc)
            time.sleep_ms(500)
            continue
        deadline = time.ticks_add(time.ticks_ms(), CONNECT_TIMEOUT_S * 1000)
        while not wlan.isconnected():
            if time.ticks_diff(deadline, time.ticks_ms()) <= 0:
                break
            time.sleep_ms(200)
        if wlan.isconnected():
            print("wifi connected:", wlan.ifconfig()[0])
            return wlan
    raise OSError("wifi connect failed")


def send_all(sock, data):
    # socket.send 可能只发送部分，需循环直至写完
    view = memoryview(data)
    while view:
        sent = sock.send(view)
        if sent == 0:
            raise OSError("socket closed")
        view = view[sent:]


def send_index(client, ip):
    body = (
        "<!doctype html><html><head>"
        "<meta name='viewport' content='width=device-width,initial-scale=1'>"
        "<title>ESP-VISION MJPEG</title></head><body>"
        "<h3>ESP-VISION MJPEG</h3>"
        "<img src='/stream'>"
        "</body></html>"
    ).encode()
    send_all(client, b"HTTP/1.0 200 OK\r\n")
    send_all(client, b"Content-Type: text/html\r\n")
    send_all(client, b"Connection: close\r\n")
    send_all(client, b"Content-Length: %d\r\n\r\n" % len(body))
    send_all(client, body)


def send_stream(client):
    # multipart/x-mixed-replace：浏览器 <img> 会持续接收每个帧
    send_all(client, b"HTTP/1.0 200 OK\r\n")
    send_all(client, b"Content-Type: multipart/x-mixed-replace; boundary=%s\r\n" % BOUNDARY)
    send_all(client, b"Cache-Control: no-cache\r\n")
    send_all(client, b"Connection: close\r\n\r\n")

    while True:
        img = sensor.snapshot()
        jpg = img.to_jpeg(quality=JPEG_QUALITY)
        data = jpg.bytearray()

        send_all(client, b"--%s\r\n" % BOUNDARY)
        send_all(client, b"Content-Type: image/jpeg\r\n")
        send_all(client, b"Content-Length: %d\r\n\r\n" % len(data))
        send_all(client, data)
        send_all(client, b"\r\n")

        time.sleep_ms(FRAME_DELAY_MS)


# 1. 连 Wi-Fi
wlan = connect_wifi()
ip = wlan.ifconfig()[0]

# 2. 相机初始化（采集顺序固定）
sensor.reset()
sensor.set_pixformat(sensor.RGB565)
sensor.set_framesize(sensor.QVGA)
sensor.skip_frames(time=1000)

# 3. 起 TCP 服务器；客户端请求 /stream 走 MJPEG，其它返回 index 页
server = socket.socket(socket.AF_INET, socket.SOCK_STREAM)
try:
    server.setsockopt(socket.SOL_SOCKET, socket.SO_REUSEADDR, 1)
except OSError:
    pass
server.bind(("0.0.0.0", PORT))
server.listen(1)

print("index: http://%s:%d/" % (ip, PORT))
print("stream: http://%s:%d/stream" % (ip, PORT))

while True:
    client, addr = server.accept()
    try:
        request = client.recv(512)
        if b"GET /stream" in request:
            send_stream(client)
        else:
            send_index(client, ip)
    except OSError as exc:
        print("client closed:", exc)
    finally:
        client.close()
```

浏览器打开 `http://<board-ip>/`（或直接 `http://<board-ip>/stream`）即可看到实时画面。

要点：
- `to_jpeg(quality=N)` 的 `quality` 1-100，权衡体积与清晰度；MJPG 推流常取 30-50 以降带宽。
- `jpg.bytearray()` 取 JPEG 字节，等价于 `jpg.save(path)` 写盘之外的内存访问路径。
- 用 `HTTP/1.0` + `Connection: close` 简化客户端逻辑；`multipart/x-mixed-replace; boundary=<token>` 是浏览器原生支持的 MJPEG 协议。
- `send_all` 必须循环发送：`socket.send` 在非阻塞或大帧时可能只发一部分。
- 客户端断开后 `send_all` 抛 `OSError`，在 `try/finally` 里 `client.close()` 释放后回到 `accept()`。

### 与 RTSP/H.264 的取舍

| 路径 | 编码 | 传输 | 芯片 | 客户端 |
|---|---|---|---|---|
| RTSP（`recipes/rtsp_stream.md`） | H.264（帧间压缩） | RTSP/TCP | 仅 ESP32-P4 | VLC/ffplay |
| MJPEG（本配方） | JPEG（帧内压缩） | HTTP multipart | S3 / S31 / P4 均可 | 任意浏览器 |

JPEG 无帧间压缩，带宽高于 H.264；但兼容性最好，且无需 `h264` 模块。

## 常见错误

| 错误 | 原因 | 解决方法 |
|---|---|---|
| 浏览器画面不刷新 | `send_all` 没循环写完 | 用 `memoryview` 循环 `send` 直至写空 |
| `ConnectionError`/`ECONNABORTED` | 客户端关闭浏览器标签 | `send_stream` 用 `try/except OSError` 捕获，`finally` `client.close()` |
| 帧率很低 / 内存告急 | JPEG quality 过高、分辨率过大 | 降 `JPEG_QUALITY`、用 `QQVGA`、加大 `FRAME_DELAY_MS` |
| Wi-Fi 连不上 | SSID/密码错 / 信号弱 | 示例自带 3 次重试；`isconnected()` 超时即抛错 |
| 画面偏色 | 缺 `skip_frames` | 采集前 `sensor.skip_frames(time=1000)` 等 AE/AWB 收敛 |
| 在 P4 上误用此方案 | P4 有 H.264，MJPG 浪费带宽 | P4 推荐用 `recipes/rtsp_stream.md`（RTSP+H.264） |

## 参考项目

- `example/01-Camera/03-MJPEG/wifi_mjpeg_stream.py`
- `docs/zh_CN/concepts/codec-streaming.rst`（JPEG / ImageIO / H.264 / RTSP / USB CDC 各路径对比）
- `docs/zh_CN/api-reference/image.rst`（`to_jpeg` / `bytearray` / `save`）
