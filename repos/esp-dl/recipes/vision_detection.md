# 视觉目标检测端到端（JPEG 解码 → 模型 → 后处理）

> **适用摘要**: 使用 ESP-DL 跑一个 YOLO11/COCO 检测模型：软件解码 JPEG 得到 `img_t`，构造检测器（`DetectWrapper` 子类），`run(img)` 得到检测结果（类别、分数、框），按需释放内存。

## 触发意图

- "跑 yolo 检测"
- "ESP-DL 目标检测"
- "JPEG 解码后送模型"
- "coco_detect 怎么用"

## 前置条件

| 条件 | 要求 |
|---|---|
| 依赖 | `idf_component.yml` 含 `espressif/esp-dl` 与对应模型组件（如 `coco_detect`）及板级 BSP |
| 参考示例 | `examples/yolo11_detect`、`examples/cat_detect`、`examples/pedestrian_detect` |

## 分步说明

### 1. 解码 JPEG 得到 `img_t`

`dl::image::sw_decode_jpeg` 把 JPEG 解码为指定像素格式（检测常用 `DL_IMAGE_PIX_TYPE_RGB888`）：

```cpp
#include "dl_image_jpeg.hpp"

// bus.jpg 通过 EMBED_FILES 嵌入（见示例 CMakeLists）
extern const uint8_t bus_jpg_start[] asm("_binary_bus_jpg_start");
extern const uint8_t bus_jpg_end[]   asm("_binary_bus_jpg_end");

dl::image::jpeg_img_t jpeg_img = {
    .data     = (void *)bus_jpg_start,
    .data_len = (size_t)(bus_jpg_end - bus_jpg_start),
};
auto img = dl::image::sw_decode_jpeg(jpeg_img, dl::image::DL_IMAGE_PIX_TYPE_RGB888);
```

带硬件 JPEG 的芯片（如 ESP32-P4，需 `CONFIG_SOC_JPEG_CODEC_SUPPORTED`）可用 `hw_decode_jpeg`：

```cpp
auto img = dl::image::hw_decode_jpeg(jpeg_img, dl::image::DL_IMAGE_PIX_TYPE_RGB888);
```

`img_t` 结构（来自 `dl_image_define.hpp`）：

```cpp
typedef struct img_s {
    void *data;
    uint16_t width;
    uint16_t height;
    pix_type_t pix_type;
    // 便捷方法：channel(), bytes(), pix_quant(), col_step(), row_step()
} img_t;
```

### 2. 构造检测器并推理

`COCODetect` 继承自 `dl::detect::DetectWrapper`，内部封装模型加载、预处理、推理与后处理（YOLO11 postprocessor + NMS）：

```cpp
#include "coco_detect.hpp"
#include "esp_log.h"

const char *TAG = "yolo11n";

COCODetect *detect = new COCODetect();        // 默认模型类型 + lazy_load
auto &detect_results = detect->run(img);      // 返回检测结果引用

for (const auto &res : detect_results) {
    ESP_LOGI(TAG,
             "[category: %d, score: %f, x1: %d, y1: %d, x2: %d, y2: %d]",
             res.category, res.score,
             res.box[0], res.box[1], res.box[2], res.box[3]);
}
```

`coco_detect::Yolo11n` 默认阈值（来自 `models/coco_detect/coco_detect.hpp`）：`default_score_thr = 0.25`，`default_nms_thr = 0.7`。

### 3. 从 SD 卡加载模型（可选）

示例通过 Kconfig `CONFIG_COCO_DETECT_MODEL_IN_SDCARD` 选择模型来源，启用时需先挂载 SD 卡：

```cpp
#if CONFIG_COCO_DETECT_MODEL_IN_SDCARD
#include "bsp/esp-bsp.h"
ESP_ERROR_CHECK(bsp_sdcard_mount());
#endif
// ... 解码 + 推理 ...
#if CONFIG_COCO_DETECT_MODEL_IN_SDCARD
ESP_ERROR_CHECK(bsp_sdcard_unmount());
#endif
```

### 4. 释放内存

```cpp
delete detect;
heap_caps_free(img.data);   // sw_decode_jpeg 分配的图像数据需手动释放
```

### 5. 示例 `main/CMakeLists.txt` 关键片段

```cmake
set(src_dirs ./)
set(include_dirs ./)
set(requires coco_detect)

if (IDF_TARGET STREQUAL "esp32s3")
    list(APPEND requires esp32_s3_eye_noglib esp_lcd)
elseif (IDF_TARGET STREQUAL "esp32p4")
    list(APPEND requires esp32_p4_function_ev_board_noglib esp_lcd)
endif()

set(embed_files "bus.jpg")
idf_component_register(SRC_DIRS ${src_dirs} INCLUDE_DIRS ${include_dirs}
                       REQUIRES ${requires} EMBED_FILES ${embed_files})
```

> 其它检测示例同理：`examples/cat_detect`（`cat_detect` 组件）、`examples/pedestrian_detect`、`examples/hand_detect` 等，只是模型组件与后处理类不同。

## 常见错误

| 错误 | 原因 | 解决方法 |
|---|---|---|
| 检测全错/空 | 图像像素格式与模型预处理不符 | 检测用 `DL_IMAGE_PIX_TYPE_RGB888`，确认模型组件期望一致 |
| 内存不足 | 大图 + 模型占满 PSRAM | 用更小输入分辨率；分区/SD 卡加载；`param_copy=false` |
| `hw_decode_jpeg` 未定义 | 芯片无硬件 JPEG | 改用 `sw_decode_jpeg` |
| 嵌入图片找不到符号 | 文件名点号未转下划线 | 符号名 `_binary_<name>_start/_end`，点→下划线 |
| 框/分数异常 | NMS/score 阈值不合适 | 构造时调 `score_thr`/`nms_thr` |

## 参考

- `examples/yolo11_detect/main/app_main.cpp`
- `examples/yolo11_detect/main/CMakeLists.txt`
- `esp-dl/vision/image/dl_image_jpeg.hpp`（`sw_decode_jpeg` / `hw_decode_jpeg`）
- `esp-dl/vision/image/dl_image_define.hpp`（`img_t` / `pix_type_t`）
- `models/coco_detect/coco_detect.hpp`（`COCODetect`、默认阈值）
