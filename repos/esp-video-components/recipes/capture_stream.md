# V4L2 POSIX 采集流程

> **适用摘要**: 用标准 V4L2 POSIX API（open/ioctl/mmap）从 `/dev/videoN` 采集图像流，包含设置格式、申请/映射缓冲、启动流、出队入队循环、停止流的全过程。

## 触发意图

- "采集摄像头数据"
- "V4L2 采集"
- "VIDIOC_S_FMT / REQBUFS / QBUF / DQBUF"
- "摄像头抓帧"
- "获取视频流"

## 前置条件

| 条件 | 要求 |
|---|---|
| 系统初始化 | 已调用 `esp_video_init()`（见 `recipes/video_init.md`） |
| 参考示例 | `esp_video/examples/capture_stream/main/capture_stream_main.c` |

## 分步说明

### 1. 打开设备并查询能力

```c
#include <fcntl.h>
#include <sys/ioctl.h>
#include <sys/mman.h>
#include "esp_video_device.h"
#include "linux/videodev2.h"
#include "esp_video_ioctl.h"
#include "esp_cam_sensor_types.h"

int fd = open(ESP_VIDEO_MIPI_CSI_DEVICE_NAME, O_RDONLY);   /* /dev/video0 */
if (fd < 0) { /* 失败 */ }

struct v4l2_capability cap;
ioctl(fd, VIDIOC_QUERYCAP, &cap);
/* cap.driver / cap.card / cap.bus_info / cap.capabilities */
```

### 2. 设置输出格式

```c
const int type = V4L2_BUF_TYPE_VIDEO_CAPTURE;

struct v4l2_format format = {
    .type = type,
    .fmt.pix.width  = 800,
    .fmt.pix.height = 600,
    .fmt.pix.pixelformat = V4L2_PIX_FMT_RGB565,
};
if (ioctl(fd, VIDIOC_S_FMT, &format) != 0) { /* 失败 */ }
```

### 3. 枚举真实支持的格式/分辨率/帧率

不要凭空填分辨率，先枚举：

```c
for (int i = 0; ; i++) {
    struct v4l2_fmtdesc fmtdesc = { .index = i, .type = type };
    if (ioctl(fd, VIDIOC_ENUM_FMT, &fmtdesc) != 0) break;

    struct v4l2_frmsizeenum frmsize = { .index = 0, .pixel_format = fmtdesc.pixelformat };
    if (ioctl(fd, VIDIOC_ENUM_FRAMESIZES, &frmsize) != 0) continue;

    for (int j = 0; ; j++) {
        memset(&frmsize, 0, sizeof(frmsize));
        frmsize.index = j;
        frmsize.pixel_format = fmtdesc.pixelformat;
        if (ioctl(fd, VIDIOC_ENUM_FRAMESIZES, &frmsize) != 0) break;
        if (frmsize.type != V4L2_FRMSIZE_TYPE_DISCRETE) break;
        /* frmsize.discrete.width / .height */
    }
}
```

### 4. 申请并映射缓冲（MMAP 模式）

```c
#define BUFFER_COUNT 2
uint8_t *buffers[BUFFER_COUNT];

struct v4l2_requestbuffers req = {
    .count  = BUFFER_COUNT,
    .type   = type,
    .memory = V4L2_MEMORY_MMAP,
};
ioctl(fd, VIDIOC_REQBUFS, &req);

for (int i = 0; i < BUFFER_COUNT; i++) {
    struct v4l2_buffer buf = {
        .type = type, .memory = V4L2_MEMORY_MMAP, .index = i,
    };
    ioctl(fd, VIDIOC_QUERYBUF, &buf);
    buffers[i] = mmap(NULL, buf.length, PROT_READ | PROT_WRITE, MAP_SHARED,
                      fd, buf.m.offset);
    ioctl(fd, VIDIOC_QBUF, &buf);                  /* 初始入队 */
}
```

### 5. 启动流并采集循环

```c
#include "esp_timer.h"

ioctl(fd, VIDIOC_STREAMON, &type);

uint32_t frame_count = 0;
int64_t start_us = esp_timer_get_time();
while (esp_timer_get_time() - start_us < 3 * 1000 * 1000) {   /* 采 3 秒 */
    struct v4l2_buffer buf = { .type = type, .memory = V4L2_MEMORY_MMAP };
    if (ioctl(fd, VIDIOC_DQBUF, &buf) != 0) break;

    if (buf.flags & V4L2_BUF_FLAG_DONE) {
        /* buffers[buf.index] 含有效数据，长度 buf.bytesused */
        frame_count++;
    }
    ioctl(fd, VIDIOC_QBUF, &buf);                  /* 必须归还 */
}

ioctl(fd, VIDIOC_STREAMOFF, &type);
```

### 6. USERPTR 模式（缓冲放 PSRAM，需对齐）

```c
#include "esp_heap_caps.h"
#define MEMORY_ALIGN 64

buffer[i] = heap_caps_aligned_alloc(MEMORY_ALIGN, buf.length,
                                    MALLOC_CAP_SPIRAM | MALLOC_CAP_CACHE_ALIGNED);
buf.m.userptr = (unsigned long)buffer[i];
/* memory 改为 V4L2_MEMORY_USERPTR */
```

### 7. 读取 sensor 芯片 ID（验证通信）

```c
esp_cam_sensor_id_t chip_id;
struct v4l2_ext_control control[1];
struct v4l2_ext_controls controls = {
    .ctrl_class = V4L2_CTRL_CLASS_ESP_CAM_IOCTL,
    .count = 1,
    .controls = control,
};
control[0].id   = ESP_CAM_SENSOR_IOC_G_CHIP_ID;
control[0].p_u8 = (uint8_t *)&chip_id;
control[0].size = sizeof(chip_id);
ioctl(fd, VIDIOC_G_EXT_CTRLS, &controls);
/* chip_id.pid 为产品 ID */
```

## 常见错误

| 错误 | 原因 | 解决方法 |
|---|---|---|
| 采集几帧后卡住 | DQBUF 后未 QBUF 归还 | 循环内务必 `ioctl(fd, VIDIOC_QBUF, &buf)` |
| `VIDIOC_S_FMT` 失败 | 格式/分辨率 sensor 不支持 | 用 `VIDIOC_ENUM_FMT`/`ENUM_FRAMESIZES` 枚举真实项 |
| `buf.flags & V4L2_BUF_FLAG_ERROR` | sensor/ISP 出错帧 | 跳过统计但仍要 QBUF 归还 |
| USERPTR 模式 DMA 异常 | 缓冲未对齐 PSRAM 缓存行 | 用 `heap_caps_aligned_alloc(64, ...)` |
| FPS 为 0 | STREAMON 未调用或 XCLK 未启动 | 检查初始化顺序与 XCLK |

## 参考项目

- `esp_video/examples/capture_stream/main/capture_stream_main.c` — 完整采集（含格式枚举、CHIP_ID、镜像控制）
- `esp_video/examples/capture_stream/main/Kconfig.projbuild` — 缓冲类型/格式选项
