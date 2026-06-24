# 视觉姿态估计端到端（JPEG 解码 → 模型 → 关键点后处理）

> **适用摘要**: 使用 ESP-DL 跑一个 YOLO11n-pose/COCO 姿态模型：软件解码 JPEG 得到 `img_t`，构造 `COCOPose`（内部用 `yolo11posePostProcessor`），`run(img)` 得到每个实例的边界框 + 17 个 COCO 关键点坐标。

## 触发意图

- "跑 yolo11 姿态估计"
- "ESP-DL 关键点检测"
- "yolo11n-pose 部署"
- "coco_pose 怎么用"
- "keypoint 检测"

## 前置条件

| 条件 | 要求 |
|---|---|
| 依赖 | `idf_component.yml` 含 `espressif/coco_pose`（`override_path: ../../../models/coco_pose`）与对应板级 BSP |
| 参考示例 | `examples/yolo11_pose`（S3/P4） |
| 量化脚本 | `examples/tutorial/how_to_quantize_model/quantize_yolo11n-pose/quantize_onnx_model.py`（PTQ）、`yolo11n_pose_qat.py`（QAT） |

## 分步说明

### 1. 解码 JPEG 得到 `img_t`

与检测一致，姿态估计使用 `DL_IMAGE_PIX_TYPE_RGB888`：

```cpp
#include "dl_image_jpeg.hpp"

extern const uint8_t bus_jpg_start[] asm("_binary_bus_jpg_start");
extern const uint8_t bus_jpg_end[]   asm("_binary_bus_jpg_end");

dl::image::jpeg_img_t jpeg_img = {
    .data     = (void *)bus_jpg_start,
    .data_len = (size_t)(bus_jpg_end - bus_jpg_start),
};
auto img = dl::image::sw_decode_jpeg(jpeg_img, dl::image::DL_IMAGE_PIX_TYPE_RGB888);
```

### 2. 构造 `COCOPose` 并推理

`COCOPose` 继承自 `dl::detect::DetectWrapper`，内部封装模型加载、预处理、推理与 `yolo11posePostProcessor` 后处理（解析 score/box/kpt 三个输出，按 score 过滤后解码框与关键点，再 NMS）：

```cpp
#include "coco_pose.hpp"     // 提供 COCOPose
#include "esp_log.h"

const char *TAG = "yolo11n-pose";

COCOPose *pose = new COCOPose();        // 默认模型类型 + lazy_load
auto &pose_results = pose->run(img);    // 返回 std::list<dl::detect::result_t>&
```

`dl::detect::result_t`（来自 `dl_detect_define.hpp`）：

```cpp
typedef struct {
    int category;                 // 姿态模型一般为单类（人）
    float score;                  // 实例分数
    std::vector<int> box;         // [left_up_x, left_up_y, right_down_x, right_down_y]
    std::vector<int> keypoint;    // [x1,y1, x2,y2, ...] 按需成对出现
    std::vector<float> mask_coeff{};
    std::vector<uint8_t> mask{};
    void limit_box(int width, int height);
    void limit_keypoint(int width, int height);   // 关键点越界裁剪
    int box_area() const;
} result_t;
```

### 3. 读取 17 个 COCO 关键点

YOLO11n-pose 每个实例输出 17 个 COCO 关键点，`keypoint` 按 `[x0,y0, x1,y1, ...]` 成对存放：

```cpp
const char *kpt_names[17] = {"nose", "left eye", "right eye", "left ear", "right ear",
                             "left shoulder", "right shoulder", "left elbow", "right elbow",
                             "left wrist", "right wrist", "left hip", "right hip",
                             "left knee", "right knee", "left ankle", "right ankle"};

for (const auto &res : pose_results) {
    ESP_LOGI(TAG, "[score: %f, x1: %d, y1: %d, x2: %d, y2: %d]",
             res.score, res.box[0], res.box[1], res.box[2], res.box[3]);

    char log_buf[512];
    char *p = log_buf;
    for (int i = 0; i < 17; ++i) {
        p += sprintf(p, "%s: [%d, %d] ", kpt_names[i],
                     res.keypoint[2 * i], res.keypoint[2 * i + 1]);
    }
    ESP_LOGI(TAG, "%s", log_buf);
}
```

### 4. 释放内存

```cpp
delete pose;
heap_caps_free(img.data);   // sw_decode_jpeg 分配的图像数据需手动释放
```

### 5. 从 SD 卡加载模型（可选）

示例通过 Kconfig `CONFIG_COCO_POSE_MODEL_IN_SDCARD` 切换模型来源：

```cpp
#if CONFIG_COCO_POSE_MODEL_IN_SDCARD
#include "bsp/esp-bsp.h"
ESP_ERROR_CHECK(bsp_sdcard_mount());
#endif
// ... 解码 + 推理 ...
#if CONFIG_COCO_POSE_MODEL_IN_SDCARD
ESP_ERROR_CHECK(bsp_sdcard_unmount());
#endif
```

### 6. 模型类型与阈值

`COCOPose` 支持的模型类型（来自 `models/coco_pose/coco_pose.hpp`）：

```cpp
typedef enum { YOLO11N_POSE_S8_V1, YOLO11N_POSE_S8_V2 } model_type_t;
COCOPose(model_type_t model_type =
             static_cast<model_type_t>(CONFIG_DEFAULT_COCO_POSE_MODEL),
         bool lazy_load = true);
```

`coco_pose::Yolo11nPose` 默认阈值：`default_score_thr = 0.25`，`default_nms_thr = 0.7`。

### 7. 示例 `main/CMakeLists.txt` 关键片段

```cmake
set(src_dirs ./)
set(include_dirs ./)
set(requires coco_pose)

if (IDF_TARGET STREQUAL "esp32s3")
    list(APPEND requires esp32_s3_eye_noglib esp_lcd)
elseif (IDF_TARGET STREQUAL "esp32p4")
    list(APPEND requires esp32_p4_function_ev_board_noglib esp_lcd)
endif()

set(embed_files "bus.jpg")
idf_component_register(SRC_DIRS ${src_dirs} INCLUDE_DIRS ${include_dirs}
                       REQUIRES ${requires} EMBED_FILES ${embed_files})
```

### 8. 精度提升路径（量化）

`how_to_deploy_yolo11n-pose.rst` 给出三种量化策略的 Pose mAP50-95（COCO）对比：

| 策略 | Pose mAP50-95 | 说明 |
|---|---|---|
| float | 50.0% | 浮点基线 |
| 8-bit PTQ（默认） | 43.1% | `QuantizationSettingFactory.espdl_setting()`，cumulative error 较大 |
| 8-bit QAT | 44.9% | `yolo11n_pose_qat.py` + `trainer.py`，out 层累积误差显著降低 |

> 默认 PTQ 的误差主要集中在头部分支（`/model.23/cv3.*`、`/model.23/cv4.*`、`/model.22/m.0/cv2`）。若精度不足，可走 QAT（有标签）或参考 `quantize_tqt.md`（TQT，无需标签）。

## 常见错误

| 错误 | 原因 | 解决方法 |
|---|---|---|
| 关键点坐标全 0/越界 | 模型输出被当成纯检测 | 确认用的是 `COCOPose`/`coco_pose.hpp` 而非 `COCODetect` |
| 关键点数对不上 | 误用其它 pose 模型 | YOLO11n-pose 固定 17 个 COCO 关键点；`keypoint.size()==34` |
| 关键点漂到框外 | 未做坐标裁剪 | 调用 `res.limit_keypoint(width, height)` 或自行 clip |
| 内存不足 | 大图 + 模型占满 PSRAM | 用更小输入分辨率；SD 卡加载；`param_copy=false` |
| mAP 明显偏低 | 默认 PTQ 累积误差大 | 走 QAT 或 TQT；检查 head 分支量化误差 |
| 找不到 `coco_pose` 组件 | 未配置 `idf_component.yml` | 加 `espressif/coco_pose` 与 `override_path` |

## 参考

- `examples/yolo11_pose/main/app_main.cpp`
- `examples/yolo11_pose/main/CMakeLists.txt`
- `examples/tutorial/how_to_quantize_model/quantize_yolo11n-pose/quantize_onnx_model.py`（PTQ）
- `examples/tutorial/how_to_quantize_model/quantize_yolo11n-pose/yolo11n_pose_qat.py`、`trainer.py`（QAT）
- `docs/en/tutorials/how_to_deploy_yolo11n-pose.rst`
- `esp-dl/vision/detect/dl_pose_yolo11_postprocessor.hpp`（`yolo11posePostProcessor`）
- `esp-dl/vision/detect/dl_detect_define.hpp`（`result_t.keypoint`）
- `models/coco_pose/coco_pose.hpp`（`COCOPose`、`coco_pose::Yolo11nPose`、默认阈值）
