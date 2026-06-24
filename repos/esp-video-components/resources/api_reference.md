# esp-video-components API 速查

> 全部来自仓库真实头文件。来源标注于每节末尾。API 名、结构体、宏均原文照录，未做臆造。

## 1. esp_video 初始化 API（esp_video/include/esp_video_init.h）

```c
/* 初始化 / 反初始化（全部子设备） */
esp_err_t esp_video_init(const esp_video_init_config_t *config);
esp_err_t esp_video_deinit(void);

/* 按标志位初始化 / 反初始化子集 */
esp_err_t esp_video_init_with_flags(const esp_video_init_config_t *config, uint32_t flags);
esp_err_t esp_video_deinit_with_flags(uint32_t flags);
```

### 初始化标志位

```c
#define ESP_VIDEO_INIT_FLAGS_MIPI_CSI   (1 << 0)
#define ESP_VIDEO_INIT_FLAGS_DVP        (1 << 1)
#define ESP_VIDEO_INIT_FLAGS_SPI        (1 << 2)
#define ESP_VIDEO_INIT_FLAGS_ISP        (1 << 3)
#define ESP_VIDEO_INIT_FLAGS_USB_UVC    (1 << 4)
#define ESP_VIDEO_INIT_FLAGS_H264       (1 << 5)
#define ESP_VIDEO_INIT_FLAGS_JPEG_ENC   (1 << 6)
#define ESP_VIDEO_INIT_FLAGS_MOTOR      (1 << 7)
#define ESP_VIDEO_INIT_FLAGS_JPEG_DEC   (1 << 8)
#define ESP_VIDEO_INIT_FLAGS_ALL        (MIPI_CSI|DVP|SPI|ISP|USB_UVC|H264|JPEG_ENC|MOTOR|JPEG_DEC)
#define ESP_VIDEO_INIT_FLAGS_JPEG       ESP_VIDEO_INIT_FLAGS_JPEG_ENC  /* 向后兼容 */
```

### esp_video_init_config_t 字段（按启用条件存在）

| 字段 | 类型 | 启用条件 |
|---|---|---|
| `csi` | `const esp_video_init_csi_config_t *` | `ESP_VIDEO_ENABLE_MIPI_CSI_VIDEO_DEVICE` |
| `dvp` | `const esp_video_init_dvp_config_t *` | `ESP_VIDEO_ENABLE_DVP_VIDEO_DEVICE` |
| `jpeg_enc`（`jpeg` 别名） | `const esp_video_init_jpeg_enc_config_t *` | `ESP_VIDEO_ENABLE_HW_JPEG_ENC_VIDEO_DEVICE` |
| `jpeg_dec` | `const esp_video_init_jpeg_dec_config_t *` | `ESP_VIDEO_ENABLE_HW_JPEG_DEC_VIDEO_DEVICE` |
| `cam_motor` | `const esp_video_init_cam_motor_config_t *` | `ESP_VIDEO_ENABLE_CAMERA_MOTOR_CONTROLLER` |
| `spi` | `const esp_video_init_spi_config_t *` | `ESP_VIDEO_ENABLE_SPI_VIDEO_DEVICE` |
| `usb_uvc` | `const esp_video_init_usb_uvc_config_t *` | `ESP_VIDEO_ENABLE_USB_UVC_VIDEO_DEVICE` |

### 各 init 配置结构体要点

**SCCB（公共）** `esp_video_init_sccb_config_t`：
```c
bool init_sccb;                 /* true=esp_video 初始化 I2C; false=用 i2c_handle */
union {
    struct { uint8_t port; gpio_num_t scl_pin; gpio_num_t sda_pin; } i2c_config;
    i2c_master_bus_handle_t i2c_handle;
};
uint32_t freq;                  /* I2C 频率 */
```

**CSI** `esp_video_init_csi_config_t`：`sccb_config` + `reset_pin` + `pwdn_pin` + `bool dont_init_ldo`

**DVP** `esp_video_init_dvp_config_t`：`sccb_config` + `reset_pin` + `pwdn_pin` + `esp_cam_ctlr_dvp_pin_config_t dvp_pin` + `uint32_t xclk_freq`

**SPI** `esp_video_init_spi_config_t`：`sccb_config` + reset/pwdn + `esp_cam_ctlr_spi_cam_intf_t intf` + `esp_cam_ctlr_spi_cam_io_mode_t io_mode` + spi_port/cs/sclk/data0..data3 + `esp_cam_sensor_xclk_source_t xclk_source` + xclk_freq/xclk_pin +（LEDC 模式）`xclk_ledc_cfg`

**USB UVC** `esp_video_init_usb_uvc_config_t`：`.uvc{ uvc_dev_num, task_stack, task_priority, task_affinity }` + `.usb{ init_usb_host_lib, peripheral_map, task_stack, ... }`

**JPEG enc/dec**：`jpeg_encoder_handle_t enc_handle` / `jpeg_decoder_handle_t dec_handle`（NULL 则内部创建）

**Cam motor** `esp_video_init_cam_motor_config_t`：`sccb_config` + reset/pwdn/signal_pin

> 来源：`esp_video/include/esp_video_init.h`

## 2. 视频设备节点（esp_video/include/esp_video_device.h）

```c
#define ESP_VIDEO_MIPI_CSI_DEVICE_NAME   "/dev/video0"   /* ID 0 */
#define ESP_VIDEO_ISP_DVP_DEVICE_NAME    "/dev/video1"   /* ID 1 */
#define ESP_VIDEO_DVP_DEVICE_NAME        "/dev/video2"   /* ID 2 */
#define ESP_VIDEO_SPI_DEVICE_NAME        "/dev/video3"   /* ID 3 (=SPI_DEVICE_0) */
#define ESP_VIDEO_SPI_DEVICE_1_NAME      "/dev/video4"   /* ID 4 */

#define ESP_VIDEO_USB_UVC_DEVICE_ID_MIN  40
#define ESP_VIDEO_USB_UVC_DEVICE_ID_MAX  49
#define ESP_VIDEO_USB_UVC_NAME_PREFIX    "/dev/video4"
#define ESP_VIDEO_USB_UVC_DEVICE_NAME(_num)  /* _num 0..9 → /dev/video40..49 */

#define ESP_VIDEO_JPEG_DEVICE_NAME       "/dev/video10"  /* ID 10 (=JPEG_ENC) */
#define ESP_VIDEO_H264_DEVICE_NAME       "/dev/video11"  /* ID 11 */
#define ESP_VIDEO_JPEG_DEC_DEVICE_NAME   "/dev/video12"  /* ID 12 */
#define ESP_VIDEO_ISP1_DEVICE_NAME       "/dev/video20"  /* ID 20 */
```

## 3. esp_video 私有 ioctl（esp_video/include/esp_video_ioctl.h）

```c
#define VIDIOC_S_SENSOR_FMT      _IOWR('V', BASE_VIDIOC_PRIVATE+1, esp_cam_sensor_format_t)
#define VIDIOC_G_SENSOR_FMT      _IOWR('V', BASE_VIDIOC_PRIVATE+2, esp_cam_sensor_format_t)
#define VIDIOC_SET_OWNER         _IOWR('V', BASE_VIDIOC_PRIVATE+3, int)
#define VIDIOC_S_MOTOR_FMT       _IOWR('V', BASE_VIDIOC_PRIVATE+4, esp_cam_motor_format_t)
#define VIDIOC_G_MOTOR_FMT       _IOWR('V', BASE_VIDIOC_PRIVATE+5, esp_cam_motor_format_t)
#define VIDIOC_S_DQBUF_TIMEOUT   _IOWR('V', BASE_VIDIOC_PRIVATE+6, struct timeval)
#define VIDIOC_G_DQBUF_TIMEOUT   _IOWR('V', BASE_VIDIOC_PRIVATE+7, struct timeval)

/* V4L2 control id 扩展 */
#define V4L2_CID_CAMERA_AE_LEVEL     (V4L2_CID_CAMERA_CLASS_BASE + 40)
#define V4L2_CID_CAMERA_STATS        (V4L2_CID_CAMERA_CLASS_BASE + 41)
#define V4L2_CID_CAMERA_GROUP        (V4L2_CID_CAMERA_CLASS_BASE + 42)
#define V4L2_CID_MOTOR_START_TIME    (V4L2_CID_CAMERA_CLASS_BASE + 43)

/* sensor ioctl 直通控制类（仅支持 p_u8 + size） */
#define V4L2_CTRL_CLASS_ESP_CAM_IOCTL  (0x00a70000)

/* JPEG 解码输出 BGR565 */
#define V4L2_PIX_FMT_BGR565  v4l2_fourcc('B','G','R','P')

#define V4L2_FMT_STR       "%c%c%c%c"
#define V4L2_FMT_STR_ARG(fmt)  (uint8_t)((fmt)&0xFF), (uint8_t)(((fmt)>>8)&0xFF), \
    (uint8_t)(((fmt)>>16)&0xFF), (uint8_t)(((fmt)>>24)&0xFF)
```

## 4. V4L2 POSIX 采集 API（linux/videodev2.h，仓库自带）

```c
/* 文件操作 */
int  open(const char *path, int flags);     /* O_RDONLY */
int  close(int fd);

/* ioctl 命令 */
VIDIOC_QUERYCAP          /* 查询能力 v4l2_capability */
VIDIOC_ENUM_FMT          /* 枚举格式 v4l2_fmtdesc */
VIDIOC_ENUM_FRAMESIZES   /* 枚举分辨率 v4l2_frmsizeenum */
VIDIOC_ENUM_FRAMEINTERVALS /* 枚举帧间隔 v4l2_frmivalenum */
VIDIOC_G_FMT / VIDIOC_S_FMT        /* 读/设格式 v4l2_format */
VIDIOC_G_PARM / VIDIOC_S_PARM      /* 读/设流参数（timeperframe）v4l2_streamparm */
VIDIOC_REQBUFS           /* 申请缓冲 v4l2_requestbuffers */
VIDIOC_QUERYBUF          /* 查询缓冲 v4l2_buffer */
VIDIOC_QBUF / VIDIOC_DQBUF         /* 入队/出队 v4l2_buffer */
VIDIOC_STREAMON / VIDIOC_STREAMOFF /* 启动/停止流 */
VIDIOC_G_EXT_CTRLS / VIDIOC_S_EXT_CTRLS /* 扩展控制 v4l2_ext_controls */

/* 缓冲类型 */
V4L2_BUF_TYPE_VIDEO_CAPTURE     /* 采集（M2M 输出） */
V4L2_BUF_TYPE_VIDEO_OUTPUT      /* M2M 输入 */

/* 内存模式 */
V4L2_MEMORY_MMAP                /* 内核映射（默认） */
V4L2_MEMORY_USERPTR             /* 用户指针（PSRAM） */

/* 能力位 */
V4L2_CAP_VIDEO_CAPTURE / V4L2_CAP_READWRITE / V4L2_CAP_ASYNCIO
V4L2_CAP_STREAMING / V4L2_CAP_META_OUTPUT / V4L2_CAP_TIMEPERFRAME
V4L2_CAP_DEVICE_CAPS

/* 缓冲标志 */
V4L2_BUF_FLAG_DONE    /* 正常完成 */
V4L2_BUF_FLAG_ERROR   /* 出错帧 */

/* mmap */
void *mmap(void *addr, size_t len, int prot, int flags, int fd, off_t offset);

/* 帧大小/间隔类型 */
V4L2_FRMSIZE_TYPE_DISCRETE
V4L2_FRMIVAL_TYPE_DISCRETE
```

> 来源：`esp_video/include/linux/videodev2.h`（仓库自带，非系统 glibc）

## 5. 标准 V4L2 控制 id（USER / JPEG / CODEC 类）

```c
V4L2_CTRL_CLASS_USER                /* VFLIP/HFLIP 等 */
V4L2_CID_VFLIP / V4L2_CID_HFLIP

V4L2_CTRL_CLASS_JPEG                /* JPEG 编码 */
V4L2_CID_JPEG_COMPRESSION_QUALITY   /* 0-100 */

V4L2_CTRL_CLASS_CODEC               /* H.264 */
V4L2_CID_MPEG_VIDEO_H264_I_PERIOD
V4L2_CID_MPEG_VIDEO_BITRATE
V4L2_CID_MPEG_VIDEO_H264_MIN_QP / V4L2_CID_MPEG_VIDEO_H264_MAX_QP
```

## 6. esp_cam_sensor API（esp_cam_sensor/include/esp_cam_sensor.h）

```c
esp_err_t esp_cam_sensor_query_para_desc(esp_cam_sensor_device_t *dev, esp_cam_sensor_param_desc_t *qdesc);
esp_err_t esp_cam_sensor_get_para_value(esp_cam_sensor_device_t *dev, uint32_t id, void *arg, size_t size);
esp_err_t esp_cam_sensor_set_para_value(esp_cam_sensor_device_t *dev, uint32_t id, const void *arg, size_t size);
esp_err_t esp_cam_sensor_get_capability(esp_cam_sensor_device_t *dev, esp_cam_sensor_capability_t *caps);
esp_err_t esp_cam_sensor_query_format(esp_cam_sensor_device_t *dev, esp_cam_sensor_format_array_t *format_array);
esp_err_t esp_cam_sensor_set_format(esp_cam_sensor_device_t *dev, const esp_cam_sensor_format_t *format);
esp_err_t esp_cam_sensor_get_format(esp_cam_sensor_device_t *dev, esp_cam_sensor_format_t *format);
esp_err_t esp_cam_sensor_ioctl(esp_cam_sensor_device_t *dev, uint32_t cmd, void *arg);
const char *esp_cam_sensor_get_name(esp_cam_sensor_device_t *dev);
esp_err_t esp_cam_sensor_del_dev(esp_cam_sensor_device_t *dev);
```

> 应用层一般不直接调这些（由 esp_video 内部使用），但理解其语义有助于调试与自定义驱动。

## 7. esp_cam_sensor ioctl 命令（esp_cam_sensor_types.h）

```c
#define ESP_CAM_SENSOR_IOC_HW_RESET        ESP_CAM_SENSOR_IOC(0x01, 0)
#define ESP_CAM_SENSOR_IOC_SW_RESET        ESP_CAM_SENSOR_IOC(0x02, 0)
#define ESP_CAM_SENSOR_IOC_S_TEST_PATTERN  ESP_CAM_SENSOR_IOC(0x03, sizeof(int))
#define ESP_CAM_SENSOR_IOC_S_STREAM        ESP_CAM_SENSOR_IOC(0x04, sizeof(int))
#define ESP_CAM_SENSOR_IOC_S_SUSPEND       ESP_CAM_SENSOR_IOC(0x05, sizeof(int))
#define ESP_CAM_SENSOR_IOC_G_CHIP_ID       ESP_CAM_SENSOR_IOC(0x06, sizeof(esp_cam_sensor_id_t))
#define ESP_CAM_SENSOR_IOC_S_REG           ESP_CAM_SENSOR_IOC(0x07, sizeof(esp_cam_sensor_reg_val_t))
#define ESP_CAM_SENSOR_IOC_G_REG           ESP_CAM_SENSOR_IOC(0x08, sizeof(esp_cam_sensor_reg_val_t))
#define ESP_CAM_SENSOR_IOC_S_GAIN          ESP_CAM_SENSOR_IOC(0x09, sizeof(uint8_t))
```

## 8. esp_cam_sensor 关键类型（esp_cam_sensor_types.h）

```c
/* 输出格式 */
typedef enum {
    ESP_CAM_SENSOR_PIXFORMAT_RGB565 = 1,  /* = _LE */
    ESP_CAM_SENSOR_PIXFORMAT_RGB565_BE,
    ESP_CAM_SENSOR_PIXFORMAT_YUV422,       /* = _UYVY */
    ESP_CAM_SENSOR_PIXFORMAT_YUV422_YUYV,
    ESP_CAM_SENSOR_PIXFORMAT_YUV420,
    ESP_CAM_SENSOR_PIXFORMAT_RGB888,
    ESP_CAM_SENSOR_PIXFORMAT_RGB444, _RGB555, _BGR888,
    ESP_CAM_SENSOR_PIXFORMAT_RAW8, _RAW10, _RAW12,
    ESP_CAM_SENSOR_PIXFORMAT_GRAYSCALE, _JPEG
} esp_cam_sensor_output_format_t;

/* 端口 */
typedef enum { ESP_CAM_SENSOR_DVP, ESP_CAM_SENSOR_MIPI_CSI, ESP_CAM_SENSOR_SPI } esp_cam_sensor_port_t;

/* sensor 设备 */
typedef struct {
    char *name;
    esp_sccb_io_handle_t sccb_handle;
    gpio_num_t xclk_pin, reset_pin, pwdn_pin;
    esp_cam_sensor_port_t sensor_port;
    const esp_cam_sensor_format_t *cur_format;
    esp_cam_sensor_id_t id;
    uint8_t stream_status;
    const esp_cam_sensor_ops_t *ops;
    void *priv;
} esp_cam_sensor_device_t;

/* 芯片 ID */
typedef struct { uint8_t midh, midl; uint16_t pid; uint8_t ver; } esp_cam_sensor_id_t;

/* 寄存器值 */
typedef struct { uint32_t regaddr; uint32_t value; } esp_cam_sensor_reg_val_t;

/* 格式描述（VIDIOC_S_SENSOR_FMT 用） */
typedef struct _cam_sensor_format_struct {
    const char *name;
    esp_cam_sensor_output_format_t format;
    esp_cam_sensor_port_t port;
    int xclk;
    uint16_t width, height;
    const void *regs; int regs_size;
    uint8_t fps;
    const esp_cam_sensor_isp_info_t *isp_info;
    union { esp_cam_sensor_mipi_info_t mipi_info; esp_cam_sensor_spi_info_t spi_info; };
    void *reserved;
} esp_cam_sensor_format_t;

/* MIPI 信息 */
typedef struct {
    uint32_t mipi_clk, hs_settle, lane_num; bool line_sync_en;
} esp_cam_sensor_mipi_info_t;
```

### sensor control id（部分，用于 3A/镜头/马达）

```c
/* default 类 */
ESP_CAM_SENSOR_POWER, _XCLK, _SENSOR_MODE, _FPS, _BRIGHTNESS, _CONTRAST,
_SATURATION, _HUE, _GAMMA, _HMIRROR, _VFLIP, _SHARPNESS, _DENOISE,
_JPEG_QUALITY, _BLC, _SPECIAL_EFFECT, _SCENE, _DATA_SEQ
/* 3A 类 */
ESP_CAM_SENSOR_AWB, _EXPOSURE_VAL, _DGAIN, _ANGAIN, _AE_CONTROL, _AGC,
_AF_AUTO, _AF_INIT, _AF_RELEASE, _AF_START, _AF_STOP, _AF_STATUS,
_WB, _3A_LOCK, _INT_TIME, _AE_LEVEL, _GAIN, _STATS, _AE_FLICKER,
_GROUP_EXP_GAIN, _EXPOSURE_US, _AUTO_N_PRESET_WB
/* lens / led / motor 类 */
ESP_CAM_SENSOR_LENS, _FLASH_LED
```

## 9. esp_cam_sensor_xclk API（esp_cam_sensor/include/esp_cam_sensor_xclk.h）

```c
typedef void *esp_cam_sensor_xclk_handle_t;

typedef enum {
    ESP_CAM_SENSOR_XCLK_LEDC,                /* 需 CONFIG_CAMERA_XCLK_USE_LEDC */
    ESP_CAM_SENSOR_XCLK_ESP_CLOCK_ROUTER,    /* 需 CONFIG_CAMERA_XCLK_USE_ESP_CLOCK_ROUTER */
} esp_cam_sensor_xclk_source_t;

typedef struct esp_cam_sensor_xclk_config {
    union {
        struct { ledc_timer_t timer; ledc_clk_cfg_t clk_cfg; ledc_channel_t channel;
                 uint32_t xclk_freq_hz; gpio_num_t xclk_pin; } ledc_cfg;
        struct { gpio_num_t xclk_pin; uint32_t xclk_freq_hz; } esp_clock_router_cfg;
    };
} esp_cam_sensor_xclk_config_t;

esp_err_t esp_cam_sensor_xclk_allocate(esp_cam_sensor_xclk_source_t source, esp_cam_sensor_xclk_handle_t *ret_handle);
esp_err_t esp_cam_sensor_xclk_start(esp_cam_sensor_xclk_handle_t xclk_handle, const esp_cam_sensor_xclk_config_t *config);
/* stop / free 见头文件 */
```

## 10. esp_sccb_intf API（esp_sccb_intf/include/esp_sccb_intf.h）

```c
/* 按寄存器地址/值位宽组合的读写 */
esp_err_t esp_sccb_transmit_reg_a8v8 (esp_sccb_io_handle_t io_handle, uint8_t  reg_addr, uint8_t  reg_val);
esp_err_t esp_sccb_transmit_reg_a16v8(esp_sccb_io_handle_t io_handle, uint16_t reg_addr, uint8_t  reg_val);
esp_err_t esp_sccb_transmit_reg_a8v16(esp_sccb_io_handle_t io_handle, uint8_t  reg_addr, uint16_t reg_val);
esp_err_t esp_sccb_transmit_reg_a16v16(esp_sccb_io_handle_t io_handle, uint16_t reg_addr, uint16_t reg_val);
/* 对应 receive_reg_* 见头文件 */
```

> 应用层通常不直接用 SCCB（由 esp_video/sensor 驱动使用）；自定义 sensor 移植时使用。

## 11. SPI CAM 控制器枚举（esp_cam_sensor/include/esp_cam_ctlr_spi.h）

```c
typedef enum {
    ESP_CAM_CTLR_SPI_CAM_INTF_SPI,        /* SPI 接口（仅 1-bit） */
    ESP_CAM_CTLR_SPI_CAM_INTF_PARLIO,     /* Parallel IO（支持 2/4-bit） */
} esp_cam_ctlr_spi_cam_intf_t;

typedef enum {
    ESP_CAM_CTLR_SPI_CAM_IO_MODE_1BIT,
    ESP_CAM_CTLR_SPI_CAM_IO_MODE_2BIT,    /* 仅 PARLIO */
    ESP_CAM_CTLR_SPI_CAM_IO_MODE_4BIT,    /* 仅 PARLIO */
} esp_cam_ctlr_spi_cam_io_mode_t;
```

## 12. DVP 引脚配置（esp_cam_ctlr_dvp.h，由 IDF 提供）

```c
typedef struct {
    uint32_t data_width;            /* CAM_CTLR_DATA_WIDTH_8 */
    gpio_num_t data_io[8];          /* D0..D7 */
    gpio_num_t vsync_io, de_io, pclk_io, xclk_io;
} esp_cam_ctlr_dvp_pin_config_t;
```

## 13. esp_ipa（图像处理算法库）

`esp_ipa/include/` 提供 `esp_ipa.h`、`esp_ipa_types.h`、`esp_ipa_detect.h`、`esp_ipa_version.h`。该库由 ISP Pipeline Controller 在内部调用（AE/AWB/AF），应用层通常不直接调用。仅在移植/扩展算法时参考其头文件。

## 14. 仓库内置公共组件 API（example_video_common.h）

```c
esp_err_t example_video_init(void);
esp_err_t example_video_deinit(void);

/* 编码器（软编辅助） */
esp_err_t example_encoder_init(example_encoder_config_t *config, example_encoder_handle_t *ret_handle);
esp_err_t example_encoder_alloc_output_buffer(example_encoder_handle_t handle, uint8_t **buf, uint32_t *size);
esp_err_t example_encoder_free_output_buffer(example_encoder_handle_t handle, uint8_t *buf);
esp_err_t example_encoder_process(example_encoder_handle_t handle, uint8_t *src, uint32_t src_size,
                                  uint8_t *dst, uint32_t dst_size, uint32_t *dst_size_out);
esp_err_t example_encoder_set_jpeg_quality(example_encoder_handle_t handle, uint8_t quality);
esp_err_t example_encoder_deinit(example_encoder_handle_t handle);

/* 存储挂载 */
esp_err_t example_mount_fatfs_to_spiflash(example_storage_handle_t *h);
esp_err_t example_unmount_fatfs_in_spiflash(example_storage_handle_t h);
esp_err_t example_mount_fatfs_to_mmc(example_storage_handle_t *h);     /* 需 SOC_SDMMC_HOST_SUPPORTED */
esp_err_t example_unmount_fatfs_in_mmc(example_storage_handle_t h);
esp_err_t example_mount_msc_to_spiflash(example_storage_handle_t *h);  /* 需 esp_tinyusb */
esp_err_t example_mount_msc_to_mmc(example_storage_handle_t *h);
esp_err_t example_msc_storage_in_use_by_usb_host(example_storage_handle_t h, bool *is_in_use);
esp_err_t example_storage_get_capacity(example_storage_handle_t h, uint64_t *capacity);

typedef struct example_encoder_config {
    uint32_t width, height, pixel_format;  /* V4L2 格式 */
    uint8_t quality;
} example_encoder_config_t;
```

> 来源：`esp_video/examples/common_components/example_video_common/include/example_video_common.h`
