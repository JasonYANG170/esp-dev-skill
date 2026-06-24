# esp-video-components 配置项速查

> 全部来自 `esp_video/Kconfig` 与各示例 `Kconfig.projbuild`，原文照录。menuconfig 路径：`Component config -> Espressif Video Configuration`。

## 1. video device 启用开关（esp_video/Kconfig）

| Kconfig | 默认 | 依赖 | 说明 |
|---|:---:|---|---|
| `ESP_VIDEO_ENABLE_MIPI_CSI_VIDEO_DEVICE` | y | `SOC_MIPI_CSI_SUPPORTED` | MIPI-CSI（仅 P4），自动 `select ESP_VIDEO_ENABLE_ISP` |
| `ESP_VIDEO_DISABLE_MIPI_CSI_DRIVER_BACKUP_BUFFER` | y | — | 禁用 MIPI-CSI 备份缓冲（缓冲数需 >1） |
| `ESP_VIDEO_ENABLE_DVP_VIDEO_DEVICE` | y | `SOC_LCDCAM_CAM_SUPPORTED` | DVP（P4/S3） |
| `ESP_VIDEO_ENABLE_SPI_VIDEO_DEVICE` | n(P4/S3/S31) / y(其他) | `CAM_CTRL_SPI_ENABLE` | SPI 视频设备；select `DATA_PREPROCESSING` 等 |
| `ESP_VIDEO_ENABLE_THE_SECOND_SPI_VIDEO_DEVICE` | n | 上者 + P4/S3/S31 | 第二路 SPI（`/dev/video4`） |
| `ESP_VIDEO_ENABLE_USB_UVC_VIDEO_DEVICE` | n | `SOC_USB_OTG_SUPPORTED` | USB UVC host |
| `USB_UVC_VIDEO_DEVICE_URB_SIZE` | 10240 | 上者 | 单个 URB 字节 |
| `USB_UVC_INIT_TIMEOUT_MS` | 10000 (500-60000) | 上者 | UVC 枚举超时 |
| `ESP_VIDEO_ENABLE_HW_H264_VIDEO_DEVICE` | n | `IDF_TARGET_ESP32P4` | H.264 编码（仅 P4），select `ESP_VIDEO_ENABLE_H264_VIDEO_DEVICE` |
| `ESP_VIDEO_ENABLE_HW_JPEG_ENC_VIDEO_DEVICE` | n | `SOC_JPEG_CODEC_SUPPORTED` | JPEG 硬件编码 |
| `ESP_VIDEO_ENABLE_HW_JPEG_DEC_VIDEO_DEVICE` | n | `SOC_JPEG_CODEC_SUPPORTED` | JPEG 硬件解码 |
| `ESP_VIDEO_ENABLE_ISP_VIDEO_DEVICE` | y | `SOC_ISP_SUPPORTED` | ISP 视频 device，select `ESP_VIDEO_ENABLE_ISP` |
| `ESP_VIDEO_ENABLE_ISP_PIPELINE_CONTROLLER` | n | 上者 | 启用 ISP Pipeline 自动 AE/AWB/AF 任务 |
| `ISP_PIPELINE_CONTROLLER_TASK_STACK_USE_PSRAM` | n | 上者 + SPIRAM | ISP 任务栈用 PSRAM |
| `ESP_VIDEO_ISP_PIPELINE_CONTROL_CAMERA_MOTOR` | y | 上者 + MOTOR + `ESP_IPA_AF_ALGORITHM` | ISP 自动控制对焦马达 |
| `ESP_VIDEO_ENABLE_CAMERA_MOTOR_CONTROLLER` | y | `CAMERA_MOTOR_DEVICE_USED` + ISP device | 摄像头马达（AF） |
| `ESP_VIDEO_ENABLE_BITSCRAMBLER` | n | `SOC_BITSCRAMBLER_SUPPORTED` | bitscrambler |
| `ESP_VIDEO_ENABLE_DATA_PREPROCESSING` | n | — | 数据预处理 |
| `ESP_VIDEO_CHECK_PARAMETERS` | y | — | 运行时参数校验（生产可关以省体积） |

## 2. 内部 select/隐式选项（不可直接配，仅供理解）

```c
config ESP_VIDEO_ENABLE_H264_VIDEO_DEVICE      // 由 HW_H264 select
config ESP_VIDEO_ENABLE_JPEG_ENC_VIDEO_DEVICE  // 由 HW_JPEG_ENC select
config ESP_VIDEO_ENABLE_JPEG_DEC_VIDEO_DEVICE  // 由 HW_JPEG_DEC select
config ESP_VIDEO_ENABLE_ISP                    // 由 MIPI_CSI / ISP_VIDEO_DEVICE select
```

## 3. 各接口的 SoC 能力依赖（决定 Kconfig 是否可见）

| 能力宏 | 含义 | 决定的接口 |
|---|---|---|
| `SOC_MIPI_CSI_SUPPORTED` | MIPI-CSI 硬件 | MIPI-CSI |
| `SOC_LCDCAM_CAM_SUPPORTED` | LCD_CAM DVP | DVP |
| `SOC_USB_OTG_SUPPORTED` | USB OTG | USB-UVC |
| `SOC_ISP_SUPPORTED` | ISP 硬件 | ISP / ISP Pipeline |
| `SOC_JPEG_CODEC_SUPPORTED` | JPEG 编解码硬件 | JPEG enc/dec |
| `SOC_BITSCRAMBLER_SUPPORTED` | bitscrambler | bitscrambler |
| `CAM_CTRL_SPI_ENABLE` | SPI CAM 控制器 | SPI video device |

> ESP32-P4 满足全部；S3 有 DVP/USB；C3/C5/C6/C61 仅 SPI。

## 4. idf_component.yml 依赖与 target rules（esp_video/idf_component.yml）

```yaml
version: "2.2.0"
targets: [esp32p4, esp32s3, esp32c3, esp32c5, esp32c6, esp32c61, esp32s31]
dependencies:
  idf: ">=5.4"
  esp_h264:     { version: "1.3.0", rules: [{ if: "target in [esp32p4]" }] }
  usb_host_uvc: { version: "2.5.*", rules: [{ if: "target in [esp32p4,esp32s3,esp32s31]" }] }
  esp_cam_sensor: { version: "2.2.*", override_path: ../esp_cam_sensor }
  esp_ipa:        { version: "2.1.*", override_path: ../esp_ipa,
                    rules: [{ if: "target in [esp32p4]" }] }
```

## 5. XCLK 时钟源 Kconfig（esp_cam_sensor）

```c
CONFIG_CAMERA_XCLK_USE_LEDC              // 用 LEDC 生成 XCLK
CONFIG_CAMERA_XCLK_USE_ESP_CLOCK_ROUTER  // 用 esp_clock_output 生成 XCLK
```

对应枚举 `ESP_CAM_SENSOR_XCLK_LEDC` / `ESP_CAM_SENSOR_XCLK_ESP_CLOCK_ROUTER`。

## 6. 示例级 Kconfig（capture_stream 节选）

| Kconfig | 说明 |
|---|---|
| `CONFIG_EXAMPLE_VIDEO_BUFFER_TYPE_USER` | 选 USERPTR 模式（否则 MMAP） |
| `CONFIG_EXAMPLE_ENABLE_CAM_SENSOR_PIC_VFLIP` | 启用垂直翻转 |
| `CONFIG_EXAMPLE_ENABLE_CAM_SENSOR_PIC_HFLIP` | 启用水平翻转 |
| `CONFIG_EXAMPLE_SCCB_I2C_INIT_BY_APP` | 应用自建 I2C 总线（不交给 esp_video） |
| `CONFIG_EXAMPLE_ENABLE_*_CAM_SENSOR` | 选择 MIPI/DVP/SPI/USB 接口 |
| `EXAMPLE_CAM_DEV_PATH` | 由所选接口自动赋值的设备路径宏 |

## 7. image_storage 示例 Kconfig 节选

| Kconfig | 说明 |
|---|---|
| `CONFIG_EXAMPLE_FORMAT_MJPEG` | JPEG 编码存储 |
| `CONFIG_EXAMPLE_FORMAT_H264` | H.264 编码存储（要求 MAX_QP > MIN_QP） |
| `CONFIG_EXAMPLE_JPEG_COMPRESSION_QUALITY` | JPEG 质量 0-100 |
| `CONFIG_EXAMPLE_H264_I_PERIOD` / `_BITRATE` / `_MIN_QP` / `_MAX_QP` | H.264 参数 |
| `CONFIG_EXAMPLE_SDMMC_MOUNT_POINT` | SD 卡挂载点 |
| `CONFIG_EXAMPLE_DISABLE_USB_MSC_STORAGE` | 禁用 USB MSC 存储 |
| `CONFIG_TINYUSB_MSC_ENABLED` | TinyUSB MSC 启用（影响 `EXAMPLE_TINYUSB_MSC_STORAGE`） |

## 8. 板级 menuconfig（example_video_common）

```
Example Video Initialization Configuration
├── Select Target Development Board
│   ├── ESP32-P4-Function-EV-Board V1.4 / V1.5
│   ├── ESP32-P4-EYE
│   ├── ESP32-S3-EYE
│   ├── ESP32-S31-Korvo
│   └── Customized Development Board       # 自填引脚
├── Select and Set Camera Sensor Interface
│   ├── MIPI-CSI  (port/SCL/SDA/reset/pwdn/xclk)
│   ├── DVP       (xclk_freq + 8 data + vsync/de/pclk/xclk 引脚)
│   ├── SPI Camera Sensor 0/1 (spi_port/cs/sclk/data0..3 + xclk)
│   └── Use Pre-initialized SCCB(I2C) Bus for All ...
└── Storage Configuration
    ├── SPI Flash Mount Point / Partition Label
    ├── SD/MMC Mount Point / Speed Mode / Bus Width
    ├── Format Storage If Mount Fails / Always Format At Startup
    └── SD/MMC 引脚 (CMD/CLK/D0..D3) + 内部 LDO
```

## 9. 数据预处理子配置

`esp_video/src/data_reprocessing/Kconfig.data_reprocessing` 被 `rsource` 引入，用于 SPI 等接口的数据格式转换预处理。具体项见该文件。
