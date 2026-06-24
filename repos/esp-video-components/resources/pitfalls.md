# esp-video-components 综合陷阱清单

> 与 `SKILL.md` 的 Critical Pitfalls 互补，此处按主题汇总，每条含原因与对策。全部基于真实示例代码与头文件。

## A. 初始��顺序

1. **必须先 `esp_video_init` 再 open 设备节点** — 底层子设备（CSI/DVP/SPI/ISP/Codec）在 init 时才注册到 `/dev/videoN`。
2. **MIPI-CSI 的 XCLK 要在 `esp_video_init` 之前 `esp_cam_sensor_xclk_start`** — 见 `example_init_video.c` 的 `failed_2`/`failed_1`/`failed_0` 错误处理顺序。
3. **`esp_video_init_with_flags` 只初始化 flag 指定的子集** — 漏选会导致对应 `/dev/videoN` 不存在；`esp_video_init` 等价 `FLAGS_ALL`。
4. **`esp_video_deinit_with_flags` 反初始化顺序**：JPEG → H.264 → MIPI CSI → DVP → SPI → USB UVC → ISP（见头文件注释），自定义反初始化需注意依赖。

## B. SCCB / I2C

5. **`init_sccb=true` 与 `i2c_handle` 互斥** — 二选一，混用会重复初始化 I2C 总线报错。
6. **共享 SCCB 的板子用 `CONFIG_EXAMPLE_SCCB_I2C_INIT_BY_APP`** — 应用建一次 `i2c_master_bus`，所有 sensor 的 `init_sccb=false` + 同一 `i2c_handle`。
7. **I2C 频率范围 100K-400K** — menuconfig 提示 `SCCB(I2C) Frequency (100K-400K Hz)`，超出可能通信失败。
8. **双 SPI 摄像头若 sensor I2C 地址相同，必须用不同 I2C port** — 见 common README Note 2。

## C. 引脚与硬件

9. **`reset_pin`/`pwdn_pin`/`signal_pin` 无硬件时填 `-1`** — 误填 0 会把 GPIO0 拉低。
10. **DVP 数据线顺序严格 D0..D7** — `data_io[8]` 顺序错会花屏/错位。
11. **SP0A39 只能 Parallel IO 驱动** — 仅 P4/C5 支持，SPI slave 不行；其 GPIO 顺序与 BF3901 不同（见 common README）。
12. **DVP/MIPI 在某些板默认不支持** — P4-Func-EV V1.4、P4-EYE 默认无 DVP，需 Customized 自填。
13. **SD 卡 4-line 需填 D1/D2/D3 引脚** — 选 1-line 时这些不填。

## D. V4L2 采集流程

14. **顺序固定**：S_FMT → REQBUFS → QUERYBUF → mmap → QBUF → STREAMON → DQBUF → QBUF → STREAMOFF。
15. **DQBUF 后必须 QBUF 归还** — 否则几帧后队列耗尽、采集停摆。
16. **出错帧（`V4L2_BUF_FLAG_ERROR`）也要 QBUF** — capture_stream 用 `V4L2_BUF_FLAG_DONE` 判断是否统计，但无论如何都归还。
17. **`VIDIOC_S_FMT` 失败时用 `VIDIOC_ENUM_FMT` + `VIDIOC_ENUM_FRAMESIZES`** — 不要凭空填分辨率。
18. **先 S_FMT 再 S_PARM** — capture_stream 注释明确：“Set format before setting stream parameter to avoid issues”。
19. **USERPTR 缓冲需 `heap_caps_aligned_alloc(64, ..., SPIRAM|CACHE_ALIGNED)`** — 普通 malloc 会 DMA 异常。

## E. 设备节点与路径

20. **用宏不用字符串** — `ESP_VIDEO_MIPI_CSI_DEVICE_NAME` 等，移植性强。
21. **USB-UVC 设备 ID 40-49** — `ESP_VIDEO_USB_UVC_DEVICE_NAME(n)`，n=0..9。
22. **第二路 SPI = `/dev/video4`** — 需 `ESP_VIDEO_ENABLE_THE_SECOND_SPI_VIDEO_DEVICE`。
23. **JPEG 解码输出 `V4L2_PIX_FMT_BGR565`** — esp_video 自定义 fourcc `'B','G','R','P'`，不是标准 BGR565。

## F. 控制类（ext_ctrls）

24. **VFLIP/HFLIP 属于 `V4L2_CTRL_CLASS_USER`** — 不是 ESP_CAM_IOCTL 类。
25. **芯片 ID/寄存器/增益走 `V4L2_CTRL_CLASS_ESP_CAM_IOCTL`** — 且仅支持 `p_u8` + `size` 字段。
26. **JPEG 质量用 `V4L2_CID_JPEG_CLASS` + `V4L2_CID_JPEG_COMPRESSION_QUALITY`**。
27. **H.264 参数用 `V4L2_CID_CODEC_CLASS`** — 且 `MAX_QP > MIN_QP`，否则编译报错。

## G. RAW 传感器与 ISP

28. **RAW 传感器必须启用 ISP Pipeline Controller** — 否则无颜色还原，画面偏紫/黑。
29. **MIPI-CSI 自动 select `ESP_VIDEO_ENABLE_ISP`** — 但 Pipeline Controller 需单独勾。
30. **ISP 仅 ESP32-P4** — S3/C 系列无 ISP 硬件，RAW 传感器无法用此框架处理，需选 YUV/RGB 输出的 sensor。
31. **AF 需马达 + esp_ipa AF 算法** — 三个 Kconfig 都要开：`CAMERA_MOTOR_CONTROLLER`、`ESP_IPA_AF_ALGORITHM`、`ESP_VIDEO_ISP_PIPELINE_CONTROL_CAMERA_MOTOR`。

## H. 编解码（M2M）

32. **M2M 设备有两路缓冲** — `VIDEO_OUTPUT`（输入原始帧）+ `VIDEO_CAPTURE`（输出压缩流），都要 REQBUFS/QBUF/STREAMON。
33. **H.264 仅 ESP32-P4** — S3/S31 即使有 JPEG 也不支持 H.264。
34. **JPEG 编解码需 `SOC_JPEG_CODEC_SUPPORTED`** — P4/S31 支持。

## I. SPI 视频设备

35. **P4/S3/S31 上 SPI video device 默认关闭** — 必须 menuconfig 启用 `ESP_VIDEO_ENABLE_SPI_VIDEO_DEVICE`。
36. **SPI 接口仅支持 1-bit** — 2/4-bit 需 `ESP_CAM_CTLR_SPI_CAM_INTF_PARLIO`。
37. **第二路 SPI 需 `ESP_VIDEO_ENABLE_THE_SECOND_SPI_VIDEO_DEVICE`** — 且 CAM1 的 XCLK 常用 LEDC（见 example_dual SPI 配置）。

## J. 编译/依赖

38. **`idf >= 5.4`** — 低于此版本组件不兼容。
39. **`override_path` 指向本地组件** — 改源码时在 `main/idf_component.yml` 用 `override_path: ../esp_video`。
40. **`esp_tinyusb` 需手动加依赖** — 用 USB MSC 时，`main/idf_component.yml` 加 `espressif/esp_tinyusb: { version: "~2.1.*", rules: [{ if: "target in [esp32p4,esp32s3]" }] }`。
41. **sd_card 组件来自 IDF examples** — SDMMC 调试时 `list(APPEND EXTRA_COMPONENT_DIRS "$ENV{IDF_PATH}/examples/storage/sd_card/sdmmc/components/sd_card")`。

## K. 调试

42. **用 `VIDIOC_QUERYCAP` 打印 driver/card/bus_info** — 验证设备就绪与类型。
43. **读 chip id 验证 SCCB 通信** — `ESP_CAM_SENSOR_IOC_G_CHIP_ID` 拿 `pid`。
44. **`v4l2_cmd` 示例用于在线调试** — 查能力、设控制、抓帧、M2M 转换、ISP BF/CCM/Gamma。
45. **丢前几帧** — sensor 启动未稳定，`image_storage` 用 `SKIP_STARTUP_FRAME_COUNT = 2`。
