# USB 主机：UVC 主机驱动（USB 摄像头视频流）

> **适用摘要**: 用 UVC 主机驱动从 USB 摄像头采集视频流，处理设备连接回调、格式协商、帧回调（Frame Buffer），并在任务中取帧与归还帧。支持等时/批量传输、PSRAM 帧缓冲、多路流、运行时改格式。

> Version: selected repo/component version; align with the user project and dependency manifest.
> Evidence: `repos/esp-usb/resources/`, source/examples in `repos/esp-usb/`, and this recipe path `repos/esp-usb/recipes/host_uvc.md`.
> Validation: draft metadata added from repository routing; verify APIs, Kconfig symbols, and component dependencies against the selected version.

## 触发意图

- "USB 摄像头"
- "UVC 主机"
- "usb host uvc"
- "UVC frame callback"
- "USB 摄像头 MJPEG / YUY2 流"
- "uvc_host"

## 前置条件

| 条件 | 要求 |
|---|---|
| 组件依赖 | `idf.py add-dependency "espressif/usb_host_uvc^2.5.1"`（会带 `usb`） |
| ESP-IDF | `>= 5.5.3` |
| 内存 | 大帧（VGA 以上）建议使能 PSRAM：`CONFIG_SPIRAM=y`，并在 `frame_heap_caps` 用 `MALLOC_CAP_SPIRAM` |
| 参考头文件 | `host/class/uvc/usb_host_uvc/include/usb/uvc_host.h` |

> UVC 是 **Host** 类驱动，构建在 USB Host Library 之上。必须先 `usb_host_install()` 并起 Daemon Task，再 `uvc_host_install()`。

## 分步说明

### 1. 安装 USB Host Library 与 Daemon Task

UVC 驱动不替代 Host Library；应用仍需自建 Daemon Task 调 `usb_host_lib_handle_events()`。

```c
#include "usb/usb_host.h"
#include "usb/uvc_host.h"

static void usb_lib_task(void *arg) {
    while (1) {
        uint32_t event_flags;
        usb_host_lib_handle_events(portMAX_DELAY, &event_flags);
        if (event_flags & USB_HOST_LIB_EVENT_FLAGS_NO_CLIENTS) {
            usb_host_device_free_all();   // 允许设备重连
        }
    }
}

// app_main 中：
const usb_host_config_t host_config = {
    .skip_phy_setup = false,
    .intr_flags = ESP_INTR_FLAG_LOWMED,
};
ESP_ERROR_CHECK(usb_host_install(&host_config));
xTaskCreatePinnedToCore(usb_lib_task, "usb_lib", 4096, NULL,
                        15, NULL, tskNO_AFFINITY);
```

### 2. 安装 UVC 驱动并注册驱动事件

`create_background_task = true` 时，驱动自建任务驱动内部 client；否则应用须周期调 `uvc_host_handle_events(timeout)`。设备连入会回调 `UVC_HOST_DRIVER_EVENT_DEVICE_CONNECTED`，给出 `dev_addr`、`uvc_stream_index` 和可用的帧格式数 `frame_info_num`。

```c
static void uvc_event_cb(const uvc_host_driver_event_data_t *event, void *user_ctx) {
    if (event->type == UVC_HOST_DRIVER_EVENT_DEVICE_CONNECTED) {
        uint8_t dev_addr = event->device_connected.dev_addr;
        uint8_t uvc_stream_index = event->device_connected.uvc_stream_index;
        // 查询设备支持的全部帧格式（见下一步）
    }
}

const uvc_host_driver_config_t uvc_driver_config = {
    .driver_task_stack_size = 4 * 1024,
    .driver_task_priority   = 16,                 // 高于 Daemon Task
    .xCoreID                = tskNO_AFFINITY,
    .create_background_task = true,
    .event_cb               = uvc_event_cb,
};
ESP_ERROR_CHECK(uvc_host_install(&uvc_driver_config));
```

### 3. 枚举设备支持的帧格式

`uvc_host_get_frame_list()` 返回每个 UVC 流接口支持的帧描述符（格式、分辨率、帧间隔）。先传 `NULL` 查数量，再分配数组取值。

```c
size_t list_size = 0;
// 先查数量
uvc_host_get_frame_list(dev_addr, uvc_stream_index, NULL, &list_size);

uvc_host_frame_info_t (*frame_info_list)[] =
    calloc(list_size, sizeof(uvc_host_frame_info_t));
uvc_host_get_frame_list(dev_addr, uvc_stream_index, &frame_info_list, &list_size);

// UVC 帧间隔单位 100ns，转 FPS：10000000 / dwFrameInterval
#define UVC_DWFRAMEINTERVAL_TO_FPS(dw) (((dw) != 0) ? 10000000 / ((float)(dw)) : 0)

for (size_t i = 0; i < list_size; i++) {
    printf("fmt=%d %ux%u @ %.1f fps\n",
           (*frame_info_list)[i].format,
           (*frame_info_list)[i].h_res,
           (*frame_info_list)[i].v_res,
           UVC_DWFRAMEINTERVAL_TO_FPS((*frame_info_list)[i].default_interval));
}
```

### 4. 打开 UVC 流

`uvc_host_stream_open()` 按 `usb`（VID/PID/地址/流索引）和 `vs_format`（分辨率/FPS/编码）匹配并协商格式。`advanced.frame_size = 0` 表示采用设备上报的 `dwMaxVideoFrameSize`；为省内存可显式给一个更小的值（见常见错误）。

```c
uvc_host_stream_config_t stream_config = {
    .event_cb  = stream_event_cb,
    .frame_cb  = frame_callback,
    .user_ctx  = &my_ctx,
    .usb = {
        .vid = UVC_HOST_ANY_VID,           // 0 = 任意
        .pid = UVC_HOST_ANY_PID,
        .dev_addr = UVC_HOST_ANY_DEV_ADDR, // 0 = 任意
        .uvc_stream_index = 0,             // 第一个 UVC 功能
    },
    .vs_format = {
        .h_res = 320,
        .v_res = 240,
        .fps   = 15.0f,                    // 0 = 设备默认
        .format = UVC_VS_FORMAT_MJPEG,     // 或 UVC_VS_FORMAT_YUY2 等
    },
    .advanced = {
        .number_of_frame_buffers = 3,      // 建议三缓冲
        .frame_size     = 0,               // 0 = 取 dwMaxVideoFrameSize
        .frame_heap_caps = MALLOC_CAP_SPIRAM, // 大帧放 PSRAM
        .number_of_urbs = 4,               // 等时/批量 URB 个数
        .urb_size       = 10 * 1024,       // 0 = 默认 4×MPS
    },
};

uvc_host_stream_hdl_t uvc_stream;
ESP_ERROR_CHECK(uvc_host_stream_open(&stream_config, pdMS_TO_TICKS(5000), &uvc_stream));
```

> 自 v2.4.0 起，可在 `advanced.user_frame_buffers` 传入用户预分配的 `number_of_frame_buffers` 个缓冲（每个至少 `frame_size` 字节），驱动将不再自行 `heap_caps_malloc`。

### 5. 帧回调（Frame Buffer 所有权）

驱动每收到一帧就调 `frame_cb`。**返回 `true`** 表示该帧立即归驱动所有（适合只读即丢）；**返回 `false`** 表示��户保留帧，事后必须用 `uvc_host_frame_return()` 归还。常见做法：把帧指针塞进队列，在另一任务处理后再归还。

```c
// 返回 false：帧交给用户任务，稍后归还
static bool frame_callback(const uvc_host_frame_t *frame, void *user_ctx) {
    QueueHandle_t q = *(QueueHandle_t *)user_ctx;
    if (xQueueSendToBack(q, &frame, 0) != pdPASS) {
        return true;    // 队列满，丢弃此帧并立即归还
    }
    return false;       // 保留，稍后 uvc_host_frame_return()
}
```

### 6. 启动/停止/改格式

```c
ESP_ERROR_CHECK(uvc_host_stream_start(uvc_stream));
// 此后 frame_callback 持续触发

// 运行时改分辨率/格式：若流在运行，驱动会先 stop、重新协商再 start
uvc_host_stream_format_t new_fmt = { .h_res = 640, .v_res = 480,
                                     .fps = 0, .format = UVC_VS_FORMAT_MJPEG };
uvc_host_stream_format_select(uvc_stream, &new_fmt);

ESP_ERROR_CHECK(uvc_host_stream_stop(uvc_stream));
```

### 7. 流事件回调

```c
static void stream_event_cb(const uvc_host_stream_event_data_t *event, void *ctx) {
    switch (event->type) {
    case UVC_HOST_TRANSFER_ERROR:
        ESP_LOGE(TAG, "USB err %d", event->transfer_error.error);
        break;
    case UVC_HOST_DEVICE_DISCONNECTED:
        // 设备被拔：流已自动 stop，必须 close
        uvc_host_stream_close(event->device_disconnected.stream_hdl);
        break;
    case UVC_HOST_FRAME_BUFFER_OVERFLOW:   // 帧超出缓冲被丢
    case UVC_HOST_FRAME_BUFFER_UNDERFLOW:  // 无空闲帧缓冲，帧被丢
        break;
#ifdef UVC_HOST_SUSPEND_RESUME_API_SUPPORTED
    case UVC_HOST_DEVICE_SUSPENDED:
    case UVC_HOST_DEVICE_RESUMED:
        break;
#endif
    default: break;
    }
}
```

### 8. 取帧、归还、关闭、卸载

```c
// 用户任务：从队列取帧处理，然后归还
uvc_host_frame_t *frame;
if (xQueueReceive(frame_q, &frame, portMAX_DELAY) == pdPASS) {
    // frame->data, frame->data_len, frame->vs_format.{h_res,v_res,format}
    process_frame(frame->data, frame->data_len);
    uvc_host_frame_return(uvc_stream, frame);   // 归还（仅当回调返回 false 时）
}

// 关闭与卸载（所有流必须先 close）
uvc_host_stream_close(uvc_stream);
uvc_host_uninstall();
```

## 常见错误

| 错误 | 原因 | 解决方法 |
|---|---|---|
| 永远收不到 `DEVICE_CONNECTED` | Daemon Task 没起或 `usb_host_install` 未调 | 先装 Host Library 并起 `usb_lib_task` 再 `uvc_host_install` |
| `ESP_ERR_NOT_FOUND` 打开失败 | 格式/分辨率/FPS 设备不支持 | 先用 `uvc_host_get_frame_list` 查实际支持项再填 `vs_format` |
| 频繁 `FRAME_BUFFER_OVERFLOW` | `frame_size` 设得比实际帧小 | 用 `dwMaxVideoFrameSize`（`frame_size=0`），或按 FAQ 表给足 MJPEG 压缩后尺寸 |
| `FRAME_BUFFER_UNDERFLOW` 丢帧 | 帧缓冲数太少或处理太慢 | `number_of_frame_buffers` 设 3（三缓冲），在另一任务异步处理并尽快归还 |
| 内存不足 / 帧分配失败 | 大帧占用过多内部 RAM | 使能 PSRAM，`frame_heap_caps = MALLOC_CAP_SPIRAM`，或提供 `user_frame_buffers` |
| `ESP_ERR_INVALID_STATE` close 失败 | 有帧未归还 | 确保每个 `frame_cb` 返回 `false` 的帧都已 `uvc_host_frame_return` |
| 卸载失败 | 仍有未关闭的流 | 对每个 handle 调 `uvc_host_stream_close` 后再 `uvc_host_uninstall` |
| `create_background_task=false` 时无事件 | 没有轮询 | 周期调用 `uvc_host_handle_events(timeout)` |

## 参考

- `host/class/uvc/usb_host_uvc/include/usb/uvc_host.h` — UVC 主机 API、`uvc_host_stream_config_t`、`uvc_host_frame_t`
- `host/class/uvc/usb_host_uvc/README.md` — 特性总览与 API 序列图
- `host/class/uvc/usb_host_uvc/docs/arch_notes.md` — URB / FB 双缓冲架构、用户帧缓冲（v2.4.0+）
- `host/class/uvc/usb_host_uvc/docs/FAQ.md` — `dwMaxVideoFrameSize` 过大时的帧缓冲尺寸表（各分辨率 MJPEG 压缩比对应字节数）
- 示例：`host/class/uvc/usb_host_uvc/examples/basic_uvc_stream/`（`main/basic_uvc_stream.c`：Daemon Task、`uvc_host_get_frame_list`、打开/启动/帧队列/归还/`stream_event_cb`）
- 示例：`host/class/uvc/usb_host_uvc/examples/camera_display/`（UVC 帧显示）
