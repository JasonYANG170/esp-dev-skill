# 图像/视频存储（SD 卡 / SPI Flash / USB MSC）

> **适用摘要**: 将摄像头采集的帧经 JPEG/H.264 编码后，存储到 SD 卡、SPI Flash，或通过 USB MSC 暴露给主机。`image_storage` 示例分 `sd_card` 与 `usb_msc` 两个子目录。

> Version: selected repo/component version; align with the user project and dependency manifest.
> Validation: draft metadata added from repository routing; verify APIs, Kconfig symbols, and component dependencies against the selected version.

## 触发意图

- "存图片到 SD 卡"
- "录像"
- "抓拍存储"
- "USB MSC 存储"
- "保存 JPEG/H.264"

## 前置条件

| 条件 | 要求 |
|---|---|
| 芯片 | SD/MMC：ESP32-S3 / P4；SPI Flash/USB MSC：P4/S3 |
| Kconfig | 启用对应采集接口 + 编码设备（JPEG/H.264） |
| 公共组件 | `example_video_common`（提供存储挂载 API） |
| 参考示例 | `esp_video/examples/image_storage/sd_card`、`image_storage/usb_msc` |

## 分步说明

### 1. 初始化视频系统与存储

```c
#include "example_video_common.h"

example_video_init();                         /* esp_video_init + 板级 */

example_storage_handle_t storage;
#if CONFIG_EXAMPLE_STORAGE_SDMMC
example_mount_fatfs_to_mmc(&storage);         /* SD 卡挂载到 CONFIG_EXAMPLE_SDMMC_MOUNT_POINT */
#elif CONFIG_EXAMPLE_STORAGE_SPI_FLASH
example_mount_fatfs_to_spiflash(&storage);
#endif
```

`example_video_common` 提供的存储 API（见 `example_video_common.h`）：
- `example_mount_fatfs_to_mmc` / `example_unmount_fatfs_in_mmc`
- `example_mount_fatfs_to_spiflash` / `example_unmount_fatfs_in_spiflash`
- `example_mount_msc_to_mmc` / `example_mount_msc_to_spiflash`（需 `esp_tinyusb` 依赖）
- `example_storage_get_capacity`

### 2. 打开采集 + 编码设备（M2M）

```c
int cap_fd = open(EXAMPLE_CAM_DEV_PATH, O_RDONLY);
int m2m_fd = open(ESP_VIDEO_JPEG_DEVICE_NAME, O_RDONLY);   /* 或 H264 */

/* 设置 JPEG 质量 */
struct v4l2_ext_control ctrl = { .id = V4L2_CID_JPEG_COMPRESSION_QUALITY, .value = 80 };
struct v4l2_ext_controls ctrls = { .ctrl_class = V4L2_CID_JPEG_CLASS, .count = 1, .controls = &ctrl };
ioctl(m2m_fd, VIDIOC_S_EXT_CTRLS, &ctrls);
```

### 3. 采集 → 编码 → 写文件循环（简化）

```c
#define SKIP_STARTUP_FRAME_COUNT 2

/* 启动采集与 M2M 双路流（见 jpeg_h264_codec.md） */
for (int i = 0; i < SKIP_STARTUP_FRAME_COUNT; i++) {
    /* 丢弃启动前几帧，避免 sensor 未稳定 */
}

/* 主循环：DQBUF 原始帧 → 填入 OUTPUT → DQBUF 编码流 → fwrite 到文件 */
char path[64];
snprintf(path, sizeof(path), "%s/img_%04d.jpg",
         CONFIG_EXAMPLE_SDMMC_MOUNT_POINT, frame_idx);
FILE *f = fopen(path, "wb");
fwrite(encoded_buf, 1, encoded_bytesused, f);
fclose(f);
```

> `image_storage/sd_card_main.c` 用 `sd_card_fb_t` 结构（含 `buf`/`buf_bytesused`/`fmt`/`width`/`height`/`timestamp`）统一管理帧缓冲。

### 4. H.264 录像配置（仅 P4）

```c
/* menuconfig: CONFIG_EXAMPLE_FORMAT_H264 */
set_codec_control(m2m_fd, V4L2_CID_CODEC_CLASS, V4L2_CID_MPEG_VIDEO_H264_I_PERIOD, CONFIG_EXAMPLE_H264_I_PERIOD);
set_codec_control(m2m_fd, V4L2_CID_CODEC_CLASS, V4L2_CID_MPEG_VIDEO_BITRATE, CONFIG_EXAMPLE_H264_BITRATE);
set_codec_control(m2m_fd, V4L2_CID_CODEC_CLASS, V4L2_CID_MPEG_VIDEO_H264_MIN_QP, CONFIG_EXAMPLE_H264_MIN_QP);
set_codec_control(m2m_fd, V4L2_CID_CODEC_CLASS, V4L2_CID_MPEG_VIDEO_H264_MAX_QP, CONFIG_EXAMPLE_H264_MAX_QP);
```

### 5. USB MSC 模式

```c
/* 依赖 esp_tinyusb，在 main/idf_component.yml：
   espressif/esp_tinyusb: { version: "~2.1.*", rules: [{ if: "target in [esp32p4,esp32s3]" }] } */

example_mount_msc_to_spiflash(&storage);      /* 或 _to_mmc */
/* 主机插上 USB 后，SPI Flash/SD 卡作为 U 盘出现，可直接拷出图片 */
bool in_use;
example_msc_storage_in_use_by_usb_host(storage, &in_use);
```

### 6. SD/MMC 引脚配置（Customized 板）

```
Example Video Initialization Configuration  --->
    Storage Configuration  --->
        (/sdmmc) SD/MMC Card Mount Point
        SD/MMC Bus Width (4-line mode (D0-D3))
        (44) CMD GPIO Pin Number
        (43) CLK GPIO Pin Number
        (39) D0 GPIO Pin Number
        (40) D1 / (41) D2 / (42) D3
        [*] SD card powered by internal LDO
```

## 常见错误

| 错误 | 原因 | 解决方法 |
|---|---|---|
| `fopen` 失败 | 挂载点错 / 分区未格式化 | 核对 `MOUNT_POINT`；开 `Format Storage If Mount Fails` |
| SD 卡识别失败 | 引脚/总线宽度/LDO 不对 | 按引脚表配置；1-line vs 4-line；内部 LDO ID |
| USB MSC 主机看不到盘 | 未加 `esp_tinyusb` 依赖 | `main/idf_component.yml` 添加依赖 |
| 文件全是黑帧 | sensor 未稳定就编码 | 丢前 `SKIP_STARTUP_FRAME_COUNT` 帧 |
| H.264 文件无法播放 | I_PERIOD/码率不合理 | 调整 H264_I_PERIOD 与 BITRATE |

## 参考项目

- `esp_video/examples/image_storage/sd_card/main/sd_card_main.c` — JPEG/H.264 存 SD 卡完整实现
- `esp_video/examples/image_storage/usb_msc/` — USB MSC 存储
- `esp_video/examples/common_components/example_video_common/example_storage.c` — 存储挂载实现
- `esp_video/examples/common_components/example_video_common/README.md` — Storage Configuration 章节
