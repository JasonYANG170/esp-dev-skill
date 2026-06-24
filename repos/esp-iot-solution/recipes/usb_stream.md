# USB UVC/UAC 流（usb_stream）

> **适用摘要**: 使用 `usb_stream` 组件在 ESP32-S2 / ESP32-S3 上以 USB Host 方式接入 UVC 摄像头与 UAC 音频设备，配置 UVC 帧回调、UAC 麦克风/扬声器流，并对流进行暂停/恢复与音量/静音控制。`usb_stream` 仅支持 ESP32-S2/ESP32-S3，且 UVC 摄像头须兼容 USB1.1 全速 + MJPEG 输出，UAC 须兼容 UAC1.0。

## 触发意图

- "USB 摄像头采集"
- "usb_stream"
- "UVC 帧回调"
- "USB 麦克风 / 扬声器"
- "uvc_streaming_config / uac_streaming_config"

## 前置条件

| 条件 | 要求 |
|---|---|
| 目标芯片 | ESP32-S2 或 ESP32-S3（须有 USB OTG，VBUS 可供电） |
| IDF 环境 | ESP-IDF v5.0+（master 分支需 v5.3+） |
| 组件依赖 | `espressif/usb_stream` |
| 摄像头约束 | USB1.1 全速、MJPEG；等时模式带宽 < 4Mbps（500KB/s），bulk 模式 < 8.8Mbps（1100KB/s）；等时模式接口 max packet size ≤ 512 字节 |
| 音频约束 | 兼容 UAC1.0 |
| 参考示例 | `examples/usb/host/usb_camera_mic_spk/`、`examples/usb/host/usb_audio_player/`、组件自带 `components/usb/usb_stream/examples/usb_stream_lcd_display/`、`usb_stream_mic_spk/` |

## 分步说明

### 1. 添加组件依赖

```bash
idf.py set-target esp32s3
idf.py add-dependency "espressif/usb_stream"
```

### 2. 配置 UVC（摄像头）

`uvc_config_t` 中带 `(optional)` 标记的字段可全部置 0，由驱动从设备描述符自动解析。`FPS2INTERVAL(fps)` 宏把帧率换算成 100ns 单位的 frame interval。

```c
#include "usb_stream.h"

static void camera_frame_cb(uvc_frame_t *frame, void *ptr)
{
    // frame->data 为一帧 MJPEG 数据；本回调运行在独立任务，**可以阻塞**
    ESP_LOGI(TAG, "frame %ux%u, %u bytes", frame->width, frame->height, frame->data_bytes);
}

uvc_config_t uvc_config = {
    .frame_width       = 320,
    .frame_height      = 240,
    .frame_interval    = FPS2INTERVAL(15),
    .xfer_buffer_size  = 32 * 1024,   // 须大于一帧，双缓冲
    .xfer_buffer_a     = malloc(32 * 1024),
    .xfer_buffer_b     = malloc(32 * 1024),
    .frame_buffer_size = 32 * 1024,
    .frame_buffer      = malloc(32 * 1024),
    .frame_cb          = camera_frame_cb,
    .frame_cb_arg      = NULL,
};
ESP_ERROR_CHECK(uvc_streaming_config(&uvc_config));
```

### 3. 配置 UAC（麦克风 / 扬声器）

```c
uac_config_t uac_config = {
    .mic_bit_resolution    = 16,
    .mic_samples_frequence = 16000,
    .spk_bit_resolution    = 16,
    .spk_samples_frequence = 16000,
    .spk_buf_size          = 16000,   // 应为 spk_ep_mps 的整数倍
    .mic_buf_size          = 0,       // 0 表示不启用 mic 内部缓冲
    .mic_cb                = mic_frame_cb,  // mic 回调**禁止阻塞**，否则丢帧
    .mic_cb_arg            = NULL,
};
ESP_ERROR_CHECK(uac_streaming_config(&uac_config));
```

> 想阻塞处理 mic 数据时，不要用回调，改用轮询接口 `uac_mic_streaming_read(buf, buf_size, &data_bytes, timeout_ms)`。

### 4. 注册连接状态回调并启动

```c
static void stream_state_cb(usb_stream_state_t state, void *ptr)
{
    ESP_LOGI(TAG, "usb device %s", state == STREAM_CONNECTED ? "connected" : "disconnected");
}
ESP_ERROR_CHECK(usb_streaming_state_register(stream_state_cb, NULL));
ESP_ERROR_CHECK(usb_streaming_start());
ESP_ERROR_CHECK(usb_streaming_connect_wait(10000));  // 阻塞等待设备接入
```

### 5. 流控制（暂停 / 恢复 / 音量 / 静音）

```c
// 暂停摄像头流
usb_streaming_control(STREAM_UVC,    CTRL_SUSPEND, NULL);
// 恢复
usb_streaming_control(STREAM_UVC,    CTRL_RESUME,  NULL);

// 扬声器音量（0~100）、静音
int volume = 60;
bool mute  = false;
usb_streaming_control(STREAM_UAC_SPK, CTRL_UAC_VOLUME, &volume);
usb_streaming_control(STREAM_UAC_SPK, CTRL_UAC_MUTE,   &mute);
```

`usb_stream_t` 枚举值：`STREAM_UVC`、`STREAM_UAC_SPK`、`STREAM_UAC_MIC`；`stream_ctrl_t`：`CTRL_SUSPEND`、`CTRL_RESUME`、`CTRL_UAC_MUTE`、`CTRL_UAC_VOLUME`。

### 6. 扬声器写 / 麦克风读（环形缓冲）

```c
// 扬声器：把音频写入驱动环形缓冲，USB 空闲时自动送出
uac_spk_streaming_write(pcm, pcm_bytes, pdMS_TO_TICKS(100));

// 麦克风：轮询读取（替代回调，可阻塞）
size_t got = 0;
uac_mic_streaming_read(buf, sizeof(buf), &got, pdMS_TO_TICKS(100));
```

### 7. 查询 / 重置帧参数

```c
uvc_frame_size_t fsize[8];
size_t list_size = sizeof(fsize) / sizeof(fsize[0]);
size_t cur = 0;
uvc_frame_size_list_get(fsize, &list_size, &cur);   // 列出摄像头支持的分辨率/帧率

// 重置分辨率前需先 SUSPEND，RESUME 后生效
usb_streaming_control(STREAM_UVC, CTRL_SUSPEND, NULL);
uvc_frame_size_reset(640, 480, FPS2INTERVAL(10));
usb_streaming_control(STREAM_UVC, CTRL_RESUME, NULL);
```

### 8. 停止并释放

```c
usb_streaming_stop();   // 删除内部任务，释放 USB 资源
```

## 常见错误

| 错误 | 原因 | 解决方法 |
|---|---|---|
| 链接/运行报错且目标为 ESP32 | ESP32 无 USB OTG，`usb_stream` 不支持 | 改用 `idf.py set-target esp32s2` 或 `esp32s3` |
| `usb_streaming_start` 后无帧回调 | 摄像头不满足全速/MJPEG/带宽约束 | 换 USB1.1 全速 + MJPEG 摄像头；核对等时带宽 < 4Mbps、bulk < 8.8Mbps |
| `ESP_ERR_INVALID_STATE` 启动失败 | 未先 `uvc_streaming_config`/`uac_streaming_config`，或已在运行 | 先 config 再 start；已运行时须先 `usb_streaming_stop()` |
| mic 回调丢帧 | 在 `mic_cb` 中阻塞 | mic 回调禁止阻塞；改用 `uac_mic_streaming_read` 轮询 |
| 扬声器无声 | `spk_buf_size` 非 `spk_ep_mps` 整数倍 | 设为端点 MPS 的整数倍；`uac_frame_size_list_get` 查支持的采样率 |
| `usb_streaming_control` 返回 `ESP_ERR_NOT_SUPPORTED` | 设备无对应 feature unit | 换支持音量/静音控制的设备 |
| 连接不稳定 / 花屏 | VBUS 供电不足或线缆差 | 确保开发板 USB Host 口能输出电压；换短而粗的线缆 |

## 参考

- 组件头文件：`components/usb/usb_stream/include/usb_stream.h`、`libuvc_def.h`
- 在线文档：`docs/en/usb/usb_host/usb_stream.rst`
- 真实示例：`examples/usb/host/usb_camera_mic_spk/`、`examples/usb/host/usb_audio_player/`、`examples/usb/host/usb_camera_lcd_display/`
- 组件自带示例：`components/usb/usb_stream/examples/usb_stream_lcd_display/`、`usb_stream_mic_spk/`
