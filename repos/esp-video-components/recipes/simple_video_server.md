# 本地 HTTP 视频服务器（抓拍与 MJPEG 流）

> **适用摘要**: 在 ESP32 上建立多端口 HTTP 服务器，通过浏览器进行图像抓拍、MJPEG 视频流预览与相机参数配置。`simple_video_server` 示例同时支持多摄像头。

## 触发意图

- "视频服务器"
- "HTTP 摄像头预览"
- "MJPEG 流"
- "网页看摄像头"
- "局域网视频"

## 前置条件

| 条件 | 要求 |
|---|---|
| 芯片 | ESP32-P4 / S3 / C3 / C5 / C6（见示例 README Supported Targets） |
| 网络 | Wi-Fi STA 或 SoftAP（menuconfig `Example Connection Configuration`） |
| 编码 | JPEG 编码设备（`/dev/video10`）用于输出 MJPEG/JPEG |
| 参考示例 | `esp_video/examples/simple_video_server` |

## 分步说明

### 1. 网络与视频初始化

```c
/* menuconfig:
   Example Connection Configuration -> Wi-Fi SSID/Password（或 SoftAP）
*/
example_video_init();                  /* esp_video_init + 采集接口 + JPEG 编码 */
/* 网络连接（STA/SoftAP）由示例内部完成，并启动 mDNS "esp-web.local" */
```

### 2. REST API 端点（示例提供）

| 端口 | 端点 | 方法 | 说明 |
|:---:|---|:---:|---|
| 80 | `/` | GET | 主 HTML 页面（浏览器视频显示） |
| 80 | `/api/capture_image?source={n}` | GET | 返回第 n 路摄像头的 JPEG（n=0 第一路） |
| 80 | `/api/capture_binary?source={n}` | GET | 返回原始二进制图像 |
| 80 | `/api/get_camera_info` | GET | 所有摄像头分辨率与 JPEG 设置 |
| 80 | `/api/set_camera_config` | POST | 配置分辨率与 JPEG 压缩 |
| 81 | `/stream` | GET | 第一路 MJPEG 持续流 |
| 82 | `/stream` | GET | 第二路 MJPEG 持续流 |

### 3. 抓拍一帧 JPEG（原理）

```c
/* 内部：open 采集 + JPEG M2M → S_FMT → REQBUFS → DQBUF 原始 → 编码 → HTTP 响应 */
/* 应用层一般直接用示例提供的服务，按需修改分辨率： */
/* POST /api/set_camera_config  body: source=0&width=1280&height=720&quality=80 */
```

### 4. 访问方式

- 域名（mDNS）：`http://esp-web.local`、`http://esp-web.local/api/capture_image?source=0`
- IP 直连：`http://<设备IP>/`
- 浏览器访问 `/stream` 查看 MJPEG 实时流

> 流端口（81/82）后台持续推送 JPEG，浏览器另存图片可能不是实时帧。

### 5. 多摄像头

示例支持 `source=0` / `source=1` 访问两路摄像头（需配置双采集接口，如双 SPI 或 MIPI+DVP）。

## 常见错误

| 错误 | 原因 | 解决方法 |
|---|---|---|
| 浏览器打不开页面 | Wi-Fi 未连 / mDNS 不支持 | 用 IP 直连；核对 SSID/密码 |
| `/stream` 卡顿 | JPEG 编码或网络带宽不足 | 降分辨率/质量；减少 FPS |
| 抓拍全黑 | sensor 未稳定 / ISP 未开 | 丢启动帧；RAW 传感器开 ISP Pipeline |
| 第二路 source=1 无图 | 仅配置了一路摄像头 | 配置双采集接口 |
| 响应慢 | HTTP 任务栈/优先级低 | 调整任务栈与优先级 |

## 参考项目

- `esp_video/examples/simple_video_server/` — 完整服务器实现（frontend/ + main/）
- `esp_video/examples/simple_video_server/README.md` — API 端点表与网络配置
- `esp_video/examples/simple_video_server/main/Kconfig.projbuild` — 网络/服务器配置项
