# 仓库示例与板级引脚一览

> 全部路径与说明来自 `esp_video/examples/README.md` 与 `common_components/example_video_common/README.md`，原文照录。

## 1. 示例目录

| 示例路径 | 一句话说明 | Supported Targets |
|---|---|---|
| `esp_video/examples/capture_stream` | 打开视频设备并采集图像数据，遍历所有格式/分辨率/帧率并打印 FPS | P4/S3/C3/C6/C5（见各 README） |
| `esp_video/examples/image_storage/sd_card` | 将 JPEG/H.264 编码后的图像与视频流存到 SD 卡 | P4/S3（SDMMC） |
| `esp_video/examples/image_storage/usb_msc` | 将图像存到 SPI Flash 并以 USB MSC 暴露给主机 | P4/S3 |
| `esp_video/examples/simple_video_server` | 多端口 HTTP 服务器：抓拍（`/api/capture_image`）、MJPEG 流（`:81/:82/stream`）、参数配置 | P4/S3/C3/C6/C5 |
| `esp_video/examples/uvc` | 把 ESP32 实现为 USB UVC 摄像头设备（webcam） | P4/S3 |
| `esp_video/examples/v4l2_cmd` | 类 `v4l2-utils` 命令行：`v4l2-ctl`/`v4l2-bf`/`v4l2-ccm`/`v4l2-gamma` | P4/S3/C3/C6/C5 |
| `esp_video/examples/video_custom_format` | 用自定义寄存器序列 + `VIDIOC_S_SENSOR_FMT` 初始化非内置格式 | P4/S3/C3/C6/C61/C5 |
| `esp_video/examples/common_components/example_video_common` | 板级初始化公共组件（引脚/SCCB/存储），被所有示例依赖 | 全部 |

## 2. 公共组件关键文件

| 文件 | 作用 |
|---|---|
| `example_video_common/include/example_video_common.h` | 对外 API（init/encoder/storage）+ 接口宏 |
| `example_video_common/example_init_video.c` | `esp_video_init_config_t` 填充与 `example_video_init/deinit` 实现 |
| `example_video_common/example_encoder.c` | 软件编码器辅助（example_encoder_*） |
| `example_video_common/example_storage.c` | FATFS/SDMMC/USB MSC 挂载实现 |
| `example_video_common/Kconfig.projbuild` | 板级选择与引脚 menuconfig |
| `example_video_common/include/boards/<board>/example_video_common_board.h` | 各板引脚定义 |

### 支持的板子头文件目录

- `boards/esp32-p4-function-ev-board-v1.4/`
- `boards/esp32-p4-function-ev-board-v1.5/`
- `boards/esp32-p4-eye/`
- `boards/esp32-s3-eye/`
- `boards/esp32-s31-korvo/`
- `boards/customized/`

## 3. 各开发板引脚表（节选自 example_video_common README）

### MIPI-CSI / DVP / SPI 公共引脚

| 硬件 | P4-Func-EV V1.4 | P4-Func-EV V1.5 | P4-EYE | S3-EYE | S31-Korvo |
|---|:-:|:-:|:-:|:-:|:-:|
| MIPI-CSI I2C SCL | 8 | 8 | 13 | NA | NA |
| MIPI-CSI I2C SDA | 7 | 7 | 14 | NA | NA |
| MIPI-CSI Reset | NA | NA | 26 | NA | NA |
| MIPI-CSI PWDN | NA | NA | 12 | NA | NA |
| MIPI-CSI XCLK | NA | NA | 11 | NA | NA |
| DVP I2C SCL | NA | 8 | NA | 5 | 1 |
| DVP I2C SDA | NA | 7 | NA | 4 | 0 |
| DVP Reset | NA | 36 | NA | NA | NA |
| DVP PWDN | NA | 38 | NA | NA | NA |
| DVP XCLK | NA | 20 | NA | 15 | 55 |
| DVP PCLK | NA | 4 | NA | 13 | 54 |
| DVP VSYNC | NA | 37 | NA | 6 | 56 |
| DVP DE | NA | 22 | NA | 7 | 57 |
| DVP D0..D7 | NA | 2,32,33,23,3,6,5,21 | NA | 11,9,8,10,12,18,17,16 | 46..53 |
| SPI I2C SCL | NA | 8 | NA | 5 | 1 |
| SPI I2C SDA | NA | 7 | NA | 4 | 0 |
| SPI XCLK | NA | 20 | NA | 15 | 55 |
| SPI CS | NA | 37 | NA | 6 | 56 |
| SPI SCLK | NA | 4 | NA | 13 | 54 |
| SPI Data0 | NA | 21 | NA | 16 | 53 |
| SPI Data1 | NA | 5 | NA | NA | NA |

### SDMMC 引脚

| 硬件 | P4-Func-EV V1.4/V1.5 | P4-EYE | S3-EYE | S31-Korvo |
|---|:-:|:-:|:-:|:-:|
| Data Bus Width | 4 | 4 | 1 | 4 |
| CMD | 44 | 44 | 38 | 25 |
| CLK | 43 | 43 | 39 | 24 |
| D0 | 39 | 39 | 40 | 20 |
| D1 | 40 | 40 | NA | 21 |
| D2 | 41 | 41 | NA | 22 |
| D3 | 42 | 42 | NA | 23 |

### Customized Development Board 默认引脚（DevKitC）

| 硬件 | P4 | S3 | C3 | C6 | C61 | C5 | S31 |
|---|:-:|:-:|:-:|:-:|:-:|:-:|:-:|
| MIPI-CSI SCL/SDA | 8/7 | NA | NA | NA | NA | NA | NA |
| DVP SCL/SDA | 8/7 | 5/4 | NA | NA | NA | NA | 1/0 |
| DVP XCLK/PCLK/VSYNC/DE | 20/4/37/22 | 15/13/6/7 | NA | NA | NA | NA | 55/54/56/57 |
| SPI CAM0 SCL/SDA | 8/7 | 5/4 | 5/4 | 5/4 | 5/4 | 5/4 | 1/0 |
| SPI CAM0 XCLK | 20 | 15 | 8 | 0 | 0 | 0 | 55 |
| SPI CAM0 CS/SCLK/Data0 | 37/4/21 | 6/13/16 | 10/6/7 | 1/6/7 | 8/6/7 | 10/6/7 | 56/54/53 |
| SPI CAM1 SCL/SDA | 5/6 | 1/2 | NA | NA | NA | NA | NA |
| SPI CAM1 XCLK/CS/SCLK/Data0 | 23/38/22/3 | 39/42/41/40 | NA | NA | NA | NA | NA |

> 完整表见 `esp_video/examples/common_components/example_video_common/README.md`（含 SP0A39 特殊引脚、PSRAM、各 menuconfig 步骤）。

## 4. 支持的 sensor 驱动（esp_cam_sensor/sensors/）

仓库内置 27 款 sensor 驱动目录（具体能力见各驱动源码）：

`arducam` · `bf20a6` · `bf3045` · `bf3901` · `bf3925` · `bf3a03` · `gc0308` · `gc2145` · `lt6911` · `mira220` · `mt9d111` · `os02n10` · `os04c10` · `ov2640` · `ov2710` · `ov3660` · `ov5640` · `ov5645` · `ov5647` · `ov9281` · `sc030iot` · `sc035hgs` · `sc101iot` · `sc121at` · `sc202cs` · `sc2336` · `sp0a39` · `sti2250`

> 各 sensor 支持的接口（MIPI/DVP/SPI）与格式由其驱动内的 `query_support_formats` 决定，应用层通过 `VIDIOC_ENUM_FMT` 枚举。

## 5. 外部相关示例（非本仓库，README 中提及）

- `esp-iot-solution/examples/camera/video_lcd_display` — 摄像头图像显示到 LCD
- `esp-webrtc-solution` — WebRTC 应用
- `esp-who` — 图像处理开发平台
- `esp-iot-solution/examples/usb/host` — USB 摄像头示例
