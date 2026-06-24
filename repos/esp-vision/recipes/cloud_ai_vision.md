# 云端 AI 视觉推理（OpenAI 兼容 vision API）

> **适用摘要**: 采集一帧，JPEG 编码后 base64，通过 HTTPS POST 到 OpenAI 兼容 vision API（如 `gpt-4o-mini`），打印返回的图像描述。补足端侧 ESP-DL 之外的"云大脑"路径。

## 触发意图

- "云 AI 图像识别"
- "GPT Vision / gpt-4o"
- "把摄像头画面发给大模型"
- "OpenAI 兼容 API"
- "按键拍照上传识别"

## 前置条件

| 条件 | 要求 |
|---|---|
| 参考示例 | `example/07-Network/01-Cloud-AI/openai_compatible_vision.py` |
| 网络 | Wi-Fi STA（`network.WLAN`），且目标 API 走 HTTPS 时固件需带 TLS（`ssl`/`tls` 模块） |
| 触发源 | 板载按键（ESP32_P4X_EYE 用户键为 GPIO3；ESP32_S31_KORVO BOOT 键为 GPIO61） |
| 概念文档 | `docs/zh_CN/concepts/packages.rst`（运行时模块与第三方包） |
| 权威签名 | `stubs/image.pyi`（`to_jpeg` / `bytearray`） |

## 分步说明

典型链路：`snapshot()` → `to_jpeg(quality=)` → `bytearray()` → `binascii.b2a_base64` → 组 JSON（OpenAI `messages.content[].image_url.url = "data:image/jpeg;base64,..."`）→ `ssl` 包装的 socket POST HTTPS → 解析 chunked 响应 → 打印 `choices[0].message.content`。

```python
import binascii
import gc
import json
import network
import socket
import time
from machine import Pin

import sensor

SSID = "your-ssid"
PASSWORD = "your-wifi-password"

# ESP32_P4X_EYE 用户键 GPIO3；ESP32_S31_KORVO BOOT 键 GPIO61
BUTTON_PIN = 3
BUTTON_PRESSED_LEVEL = 0

API_URL = "https://api.openai.com/v1/chat/completions"
API_KEY = "your-api-key"          # 直连 OpenAI 必填；走本地代理可留空
MODEL = "gpt-4o-mini"
SYSTEM_PROMPT = "You are an AI vision assistant."
PROMPT = "Describe this image and list the visible objects. Keep it short."

JPEG_QUALITY = 35
MAX_TOKENS = 120
TLS_VERIFY = False                # 设备一般无根证书，关闭校验


def connect_wifi():
    wlan = network.WLAN(network.STA_IF)
    wlan.active(True)
    wlan.connect(SSID, PASSWORD)
    while not wlan.isconnected():
        time.sleep_ms(200)
    print("wifi connected:", wlan.ifconfig()[0])
    return wlan


def _wrap_tls(sock, host, verify):
    # MicroPython v1.28.0：优先 ssl（标准），回退 tls（老接口）
    try:
        import ssl as tls
    except ImportError:
        import tls
    context = tls.SSLContext(tls.PROTOCOL_TLS_CLIENT)
    if not verify:
        context.verify_mode = tls.CERT_NONE     # 不校验证书（设备无根 CA）
    return context.wrap_socket(sock, server_hostname=host)


def post_json(url, headers, body, tls_verify=False):
    # 解析 URL
    if url.startswith("https://"):
        scheme, rest, port = "https", url[8:], 443
    else:
        scheme, rest, port = "http", url[7:], 80
    slash = rest.find("/")
    host_port = rest if slash < 0 else rest[:slash]
    path = "/" if slash < 0 else rest[slash:]
    colon = host_port.rfind(":")
    host = host_port if colon < 0 else host_port[:colon]
    port = port if colon < 0 else int(host_port[colon + 1:])

    ai = socket.getaddrinfo(host, port, 0, socket.SOCK_STREAM)[0]
    sock = socket.socket(ai[0], ai[1], ai[2])
    try:
        sock.connect(ai[-1])
        if scheme == "https":
            sock = _wrap_tls(sock, host, tls_verify)

        body_bytes = body.encode()
        request = (
            "POST %s HTTP/1.1\r\nHost: %s\r\nUser-Agent: esp-vision\r\n"
            "Connection: close\r\nContent-Length: %d\r\n"
        ) % (path, host, len(body_bytes))
        for k, v in headers.items():
            request += "%s: %s\r\n" % (k, v)
        request += "\r\n"
        sock.write(request.encode())
        sock.write(body_bytes)

        status_code = int(sock.readline().decode().split(" ")[1])
        resp_headers = {}
        while True:
            line = sock.readline()
            if line in (b"\r\n", b"\n", b""):
                break
            name, value = line.decode().split(":", 1)
            resp_headers[name.lower()] = value.strip()

        response_body = sock.read()
        # OpenAI 用 Transfer-Encoding: chunked，必须按 chunk 解码
        if resp_headers.get("transfer-encoding", "").lower() == "chunked":
            response_body = _decode_chunked(response_body)
        return status_code, response_body.decode()
    finally:
        sock.close()


def _decode_chunked(data):
    out = bytearray()
    offset = 0
    while True:
        end = data.find(b"\r\n", offset)
        if end < 0:
            break
        size_line = data[offset:end].split(b";")[0]
        size = int(size_line, 16)
        offset = end + 2
        if size == 0:
            break
        out.extend(data[offset:offset + size])
        offset += size + 2
    return bytes(out)


def describe_image(image_b64, prompt):
    payload = json.dumps({
        "model": MODEL,
        "messages": [
            {"role": "system", "content": SYSTEM_PROMPT},
            {"role": "user", "content": [
                {"type": "image_url",
                 "image_url": {"url": "data:image/jpeg;base64," + image_b64}},
                {"type": "text", "text": prompt},
            ]},
        ],
        "max_tokens": MAX_TOKENS,
    })
    headers = {"Content-Type": "application/json"}
    if API_KEY:
        headers["Authorization"] = "Bearer " + API_KEY
    status, resp = post_json(API_URL, headers, payload, tls_verify=TLS_VERIFY)
    if status != 200:
        return resp
    choices = json.loads(resp).get("choices", [])
    if choices and "message" in choices[0]:
        return choices[0]["message"].get("content", "")
    return resp


# 1. Wi-Fi + 相机 + 按键
connect_wifi()
sensor.reset()
sensor.set_pixformat(sensor.RGB565)
sensor.set_framesize(sensor.QQVGA)        # 小分辨率省上传带宽
sensor.skip_frames(time=1000)
button = Pin(BUTTON_PIN, Pin.IN, Pin.PULL_UP)
pressed_prev = button.value() == BUTTON_PRESSED_LEVEL

print("ready: press button on GPIO%d" % BUTTON_PIN)

# 2. 持续预览；按键边沿触发一次上传
while True:
    img = sensor.snapshot()
    img.flush()

    pressed_now = button.value() == BUTTON_PRESSED_LEVEL
    trigger = pressed_now and not pressed_prev     # 下降沿触发
    pressed_prev = pressed_now

    if trigger:
        try:
            time.sleep_ms(20)                      # 消抖
            if button.value() != BUTTON_PRESSED_LEVEL:
                continue
            jpg = img.to_jpeg(quality=JPEG_QUALITY)
            image_b64 = binascii.b2a_base64(jpg.bytearray()).decode().strip()
            print("jpeg bytes:", len(jpg.bytearray()))
            print(describe_image(image_b64, PROMPT))
        except Exception as exc:
            print("cloud ai failed:", exc)
        finally:
            gc.collect()                          # 大字符串释放
            print("ready: press button on GPIO%d" % BUTTON_PIN)

    time.sleep_ms(20)
```

要点：
- **JSON 结构**遵循 OpenAI 视觉 API：`content` 是数组，含 `image_url.url = "data:image/jpeg;base64,<b64>"` 与一段 `text`。任何 OpenAI 兼容服务（含本地 LLM 代理）均适用。
- **HTTPS/TLS**：`ssl.SSLContext(PROTOCOL_TLS_CLIENT)` + `verify_mode = CERT_NONE`（设备无根证书时关闭校验，仅适合受信网络/代理；要校验需把根 CA 烧进设备）。无 `ssl` 模块时回退到 `tls`。
- **chunked 解码**：OpenAI 响应常用 `Transfer-Encoding: chunked`，必须按 `<hex-size>\r\n<data>\r\n` 解析，不能直接 `json.loads(sock.read())`。
- **触发**：用按键边沿 + 20ms 消抖，避免长按重复上传；上传完 `gc.collect()` 回收 base64 字符串。
- **降分辨率**：`QQVGA` + `JPEG_QUALITY=35` 把单帧 base64 压到数 KB，控制在 `max_tokens` 之外的请求体大小。
- 直连 `api.openai.com` 必须有 `API_KEY`；若用本地 LLM 网关（HTTP 或自签 HTTPS），可把 `API_KEY` 留空并相应改 `API_URL`。

### 与端侧 ESP-DL 推理的取舍

| 路径 | 推理位置 | 延迟 | 隐私 | 依赖 |
|---|---|---|---|---|
| ESP-DL（`recipes/object_detection.md`） | 设备 | 毫秒级 | 全本地 | `.espdl` 模型 + PSRAM |
| 云 vision LLM（本配方） | 云端 | 秒级 | 需上传 | Wi-Fi + API key |

云 LLM 适合"描述场景""回答开放式问题"等端侧模型做不到的语义任务；固定类别检测仍推荐 ESP-DL。

## 常见错误

| 错误 | 原因 | 解决方法 |
|---|---|---|
| `RuntimeError: HTTPS needs TLS` | 固件未含 `ssl`/`tls` | 用 release/v5.5 IDF 构建（SSL/TLS 在该分支启用），或改用 HTTP 代理端点 |
| 证书校验失败 | 默认 `CERT_REQUIRED` | 设 `context.verify_mode = tls.CERT_NONE`（信任网络前提），或烧入根 CA |
| `status: 400` / `401` | API key 缺失 / model 名错 | 直连 OpenAI 必须设 `API_KEY`；确认 `MODEL` 名称服务端支持 |
| JSON 解析失败 | 响应是 chunked 未解码 | 必须先 `_decode_chunked` 再 `json.loads` |
| 上传后内存不足 | base64 字符串过大 | 降 `JPEG_QUALITY`/分辨率；上传后 `gc.collect()` |
| 按键连发 | 未做边沿检测/消抖 | 比较前后状态 + `time.sleep_ms(20)` 二次确认 |
| 按键无反应 | 引脚号不对 | ESP32_P4X_EYE 用 GPIO3；ESP32_S31_KORVO BOOT 键是 GPIO61 |

## 参考项目

- `example/07-Network/01-Cloud-AI/openai_compatible_vision.py`
- `docs/zh_CN/concepts/codec-streaming.rst`（JPEG 编码路径）
- `docs/zh_CN/concepts/packages.rst`（运行时第三方模块/`mip` 限制）
- `docs/zh_CN/api-reference/image.rst`（`to_jpeg` / `bytearray`）
