# ISP Pipeline 自动图像处理

> **适用摘要**: 为输出 RAW 格式的传感器（SC2336、OV5640 RAW 等）启用 ISP Pipeline Controller，自动执行 AE（自动曝光）、AWB（自动白平衡）、AF（自动对焦，需电机）算法，获得正常颜色与亮度的图像。

> Version: selected repo/component version; align with the user project and dependency manifest.
> Evidence: `repos/esp-video-components/resources/`, source/examples in `repos/esp-video-components/`, and this recipe path `repos/esp-video-components/recipes/isp_pipeline.md`.
> Validation: draft metadata added from repository routing; verify APIs, Kconfig symbols, and component dependencies against the selected version.

## 触发意图

- "RAW 传感器偏色"
- "自动曝光 AE"
- "自动白平衡 AWB"
- "自动对焦 AF / 摄像头马达"
- "ISP pipeline"
- "esp_ipa"

## 前置条件

| 条件 | 要求 |
|---|---|
| 芯片 | ESP32-P4（需 `SOC_ISP_SUPPORTED`） |
| Kconfig | `ESP_VIDEO_ENABLE_ISP_VIDEO_DEVICE=y` + `ESP_VIDEO_ENABLE_ISP_PIPELINE_CONTROLLER=y` |
| 依赖 | `esp_ipa`（idf_component.yml rules: target in [esp32p4]） |
| AF（可选） | `ESP_VIDEO_ENABLE_CAMERA_MOTOR_CONTROLLER=y` + `ESP_IPA_AF_ALGORITHM` + `ESP_VIDEO_ISP_PIPELINE_CONTROL_CAMERA_MOTOR=y` |
| 参考示例 | `esp_video/examples/v4l2_cmd`（含 ISP BF/CCM/Gamma 控制） |

## 分步说明

### 1. menuconfig 启用 ISP Pipeline

```
Component config  --->
    Espressif Video Configuration  --->
        [*] Enable ISP based Video Device
            [*] Enable ISP Pipeline Controller
                [ ] ISP Controller Task Stack Use PSRAM   # 按需
                [*] ISP Pipeline Control Camera Motor    # 有 AF 马达才勾
```

> 启用后会创建 `isp_task` 任务：读取 ISP 硬件统计 → 调用 esp_ipa 算法 → 反写 ISP/sensor/马达参数。

### 2. 初始化（esp_video_init 默认带 ISP flag）

```c
/* MIPI-CSI 会自动 select ESP_VIDEO_ENABLE_ISP，
   esp_video_init() 默认 ESP_VIDEO_INIT_FLAGS_ALL 已包含 ISP */
static const esp_video_init_csi_config_t csi_config = { /* ... */ };
static const esp_video_init_config_t cam_config = { .csi = &csi_config };
esp_video_init(&cam_config);
/* ISP 设备节点：/dev/video20（ESP_VIDEO_ISP1_DEVICE_NAME） */
```

### 3. ISP 参数控制（BF/CCM/Gamma）

`v4l2_cmd` 示例提供专用命令控制 ISP 子模块：

| 命令 | 控制对象 | 说明 |
|---|---|---|
| `v4l2-bf` | Bayer Filter（BF） | ISP 去马赛克相关 |
| `v4l2-ccm` | Color Correction Matrix（CCM） | 颜色校正矩阵 |
| `v4l2-gamma` | Gamma 校正 | 亮度映射曲线 |

应用层等价于通过 `VIDIOC_S_EXT_CTRLS` 对 ISP 设备设置对应 control id。

### 4. AF（自动对焦，需马达）

```c
/* menuconfig:
   [*] Enable Camera Motor Controller
   [*] ISP Pipeline Control Camera Motor (ESP_VIDEO_ISP_PIPELINE_CONTROL_CAMERA_MOTOR)
   需 esp_ipa AF 算法 (ESP_IPA_AF_ALGORITHM) */

/* cam_motor 配置（esp_video_init_cam_motor_config_t） */
static const esp_video_init_cam_motor_config_t motor_config = {
    .sccb_config = { /* SCCB 配置 */ },
    .reset_pin = -1,
    .pwdn_pin  = -1,
    .signal_pin = -1,                /* 无使能信号填 -1 */
};
```

ISP Pipeline 会通过 esp_ipa 的 AF 算法自动驱动 VCM（音圈马达）对焦，应用层无需干预。

### 5. sensor 3A 直接控制（绕过 Pipeline）

若不启用 Pipeline Controller，也可手动通过 sensor 的 3A control id（`esp_cam_sensor_types.h`）：

```c
/* 通过 V4L2_CTRL_CLASS_ESP_CAM_IOCTL + ESP_CAM_SENSOR_IOC_* 或 ext_ctrls */
/* ESP_CAM_SENSOR_AE_CONTROL / ESP_CAM_SENSOR_AWB / ESP_CAM_SENSOR_AF_AUTO 等 */
```

> 这些 control id 定义在 `esp_cam_sensor_types.h`，应用一般优先用 ISP Pipeline 自动处理。

## 常见错误

| 错误 | 原因 | 解决方法 |
|---|---|---|
| RAW 画面偏紫/绿 | 未启用 ISP Pipeline | menuconfig 开 `ESP_VIDEO_ENABLE_ISP_PIPELINE_CONTROLLER` |
| 非 P4 芯片无 ISP | ISP 仅 P4 | 换 P4，或用带内置 ISP 的 sensor（YUV/RGB 输出） |
| AF 不工作 | 马达/算法未启用 | 开 `CAMERA_MOTOR_CONTROLLER` + `ESP_IPA_AF_ALGORITHM` + `ISPP_PIPELINE_CONTROL_CAMERA_MOTOR` |
| ISP 任务栈溢出 | 默认栈不够 | 启用 `ISP_PIPELINE_CONTROLLER_TASK_STACK_USE_PSRAM` 或加大栈 |
| 颜色仍不准 | CCM/BF 参数未调 | 用 `v4l2_cmd` 的 `v4l2-ccm`/`v4l2-bf` 在线调参 |

## 参考项目

- `esp_video/examples/v4l2_cmd/` — ISP BF/CCM/Gamma 命令行调试
- `esp_video/Kconfig` — ISP 相关 Kconfig 与依赖关系
- `esp_ipa/include/esp_ipa.h`、`esp_ipa_types.h` — 图像处理算法库
- `esp_video/include/esp_video_isp_ioctl.h` — ISP 私有 ioctl
