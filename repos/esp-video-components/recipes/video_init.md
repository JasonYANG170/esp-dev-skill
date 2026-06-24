# 初始化 esp_video 系统

> **适用摘要**: 配置并初始化 esp-video-components 视频子系统，包括 SCCB（I2C）、复位/掉电引脚、MIPI-CSI/DVP/SPI/USB 接口子设备注册，是所有采集操作的前置步骤。

## 触发意图

- "初始化摄像头"
- "esp_video_init 怎么用"
- "配置 MIPI/DVP/SPI 摄像头引脚"
- "摄像头打开失败 / 设备节点不存在"
- "SCCB I2C 配置"

## 前置条件

| 条件 | 要求 |
|---|---|
| ESP-IDF 版本 | >= 5.4 |
| 依赖 | `idf.py add-dependency esp_video`（或本地 `override_path`） |
| Kconfig | 启用对应 `ESP_VIDEO_ENABLE_*_VIDEO_DEVICE`（见 `resources/config_reference.md`） |
| 参考示例 | `esp_video/examples/common_components/example_video_common/example_init_video.c` |

## 分步说明

### 1. 引入头文件

```c
#include "esp_video_init.h"      // esp_video_init_config_t, esp_video_init()
#include "esp_video_device.h"    // ESP_VIDEO_*_DEVICE_NAME
#include "esp_err.h"
```

### 2. 配置 MIPI-CSI（ESP32-P4）

```c
static const esp_video_init_csi_config_t csi_config = {
    .sccb_config = {
        .init_sccb = true,                       /* 让 esp_video 初��化 I2C */
        .i2c_config = {
            .port    = 0,
            .scl_pin = 8,
            .sda_pin = 7,
        },
        .freq = 100000,                          /* 100 kHz，范围 100K-400K */
    },
    .reset_pin = -1,                             /* 无复位引脚填 -1 */
    .pwdn_pin  = -1,
    /* .dont_init_ldo = true, */                 /* 若 LDO 已在别处初始化 */
};

static const esp_video_init_config_t cam_config = {
    .csi = &csi_config,
};

esp_err_t ret = esp_video_init(&cam_config);     /* 注册 /dev/video0 等 */
```

### 3. 配置 DVP（ESP32-P4/S3）

```c
#include "esp_cam_ctlr_dvp.h"

static const esp_video_init_dvp_config_t dvp_config = {
    .sccb_config = {
        .init_sccb = true,
        .i2c_config = { .port = 0, .scl_pin = 8, .sda_pin = 7 },
        .freq = 100000,
    },
    .reset_pin = -1,
    .pwdn_pin  = -1,
    .dvp_pin = {
        .data_width = CAM_CTLR_DATA_WIDTH_8,
        .data_io = { 2, 32, 33, 23, 3, 6, 5, 21 },   /* D0..D7 */
        .vsync_io = 37,
        .de_io     = 22,
        .pclk_io   = 4,
        .xclk_io   = 20,
    },
    .xclk_freq = 20000000,                       /* 20 MHz */
};
```

> 引脚号来自 `example_video_common/README.md` 的 ESP32-P4-Function-EV-Board V1.5 表。换板子按表替换。

### 4. 配置 SPI（全系列，P4/S3 默认需 menuconfig 启用）

```c
#include "esp_cam_ctlr_spi.h"
#include "esp_cam_sensor_xclk.h"

static const esp_video_init_spi_config_t spi_config = {
    .sccb_config = {
        .init_sccb = true,
        .i2c_config = { .port = 0, .scl_pin = 8, .sda_pin = 7 },
        .freq = 100000,
    },
    .reset_pin = -1,
    .pwdn_pin  = -1,
    .intf      = ESP_CAM_CTLR_SPI_CAM_INTF_SPI,  /* 或 _PARLIO */
    .io_mode   = ESP_CAM_CTLR_SPI_CAM_IO_MODE_1BIT,
    .spi_port     = 2,
    .spi_cs_pin   = 37,
    .spi_sclk_pin = 4,
    .spi_data0_io_pin = 21,
    .spi_data1_io_pin = -1,                      /* 1-bit 模式不用填 -1 */
    .xclk_source = ESP_CAM_SENSOR_XCLK_ESP_CLOCK_ROUTER,
    .xclk_pin    = 20,
    .xclk_freq   = 24000000,
};
```

### 5. 使用应用自建的 I2C 总线（共享 SCCB）

```c
#include "driver/i2c_master.h"

static const i2c_master_bus_config_t bus_cfg = {
    .i2c_port = 0,
    .sda_io_num = 7,
    .scl_io_num = 8,
    .clk_source = I2C_CLK_SRC_DEFAULT,
    .glitch_ignore_cnt = 7,
    .flags.enable_internal_pullup = true,
};
i2c_master_bus_handle_t bus_handle;
i2c_new_master_bus(&bus_cfg, &bus_handle);

esp_video_init_csi_config_t csi_cfg = csi_config;
csi_cfg.sccb_config.init_sccb = false;            /* 关键：不要再初始化 */
csi_cfg.sccb_config.i2c_handle = bus_handle;      /* 直接传句柄 */

esp_video_init_config_t cfg = { .csi = &csi_cfg };
esp_video_init(&cfg);
```

### 6. MIPI-CSI 需要外部 XCLK 时单独分配

```c
#include "esp_cam_sensor_xclk.h"

esp_cam_sensor_xclk_handle_t xclk;
esp_cam_sensor_xclk_config_t xclk_cfg = {
    .esp_clock_router_cfg = { .xclk_pin = 11, .xclk_freq_hz = 24000000 },
};
esp_cam_sensor_xclk_allocate(ESP_CAM_SENSOR_XCLK_ESP_CLOCK_ROUTER, &xclk);
esp_cam_sensor_xclk_start(xclk, &xclk_cfg);
esp_video_init(&cam_config);                       /* XCLK 在 video init 之前启动 */
```

### 7. 按需初始化子集（带 flags）

```c
/* 仅初始化 MIPI-CSI 与 ISP，不初始化 DVP/USB/Codec */
esp_video_init_with_flags(&cam_config,
    ESP_VIDEO_INIT_FLAGS_MIPI_CSI | ESP_VIDEO_INIT_FLAGS_ISP);

/* 反初始化 */
esp_video_deinit();            /* 等价 ESP_VIDEO_INIT_FLAGS_ALL */
esp_video_deinit_with_flags(ESP_VIDEO_INIT_FLAGS_MIPI_CSI | ESP_VIDEO_INIT_FLAGS_ISP);
```

`ESP_VIDEO_INIT_FLAGS_*` 取值：`MIPI_CSI(1<<0)`、`DVP(1<<1)`、`SPI(1<<2)`、`ISP(1<<3)`、`USB_UVC(1<<4)`、`H264(1<<5)`、`JPEG_ENC(1<<6)`、`MOTOR(1<<7)`、`JPEG_DEC(1<<8)`、`ALL`。

## 常见错误

| 错误 | 原因 | 解决方法 |
|---|---|---|
| `open("/dev/video0")` 返回 -1 | 未调用 `esp_video_init` | 先调用 `esp_video_init(&cfg)` |
| SCCB 报错 / sensor 探测不到 | I2C 重复初始化或引脚错 | `init_sccb` 与 `i2c_handle` 二选一；核对引脚表 |
| 编译报 `esp_video_init_csi_config_t` 未定义 | 未启用 `ESP_VIDEO_ENABLE_MIPI_CSI_VIDEO_DEVICE` | menuconfig 启用对应 Kconfig |
| `esp_video_init` 返回非 ESP_OK | LDO/复位引脚配置错 | 检查 `reset_pin`/`pwdn_pin` 是否为 -1、LDO 是否被别处占用 |
| SPI 设备在 P4 上不存在 | P4/S3 默认关闭 SPI video device | menuconfig 启用 `ESP_VIDEO_ENABLE_SPI_VIDEO_DEVICE` |

## 参考项目

- `esp_video/examples/common_components/example_video_common/example_init_video.c` — 完整的 csi/dvp/spi/usb_uvc 初始化模板
- `esp_video/examples/common_components/example_video_common/README.md` — 各开发板引脚表与 menuconfig 步骤
- `esp_video/include/esp_video_init.h` — 所有 init 结构体定义
