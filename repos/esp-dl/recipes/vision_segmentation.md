# 实例分割端到端（JPEG 解码 → 模型 → mask 后处理）

> **适用摘要**: 使用 ESP-DL 跑一个 YOLO11n-seg/COCO 分割模型：软件解码 JPEG 得到 `img_t`，构造 `COCOSeg`（内部 `yolo11segPostProcessor` 解析 score/box/mask_coeff + 32 维 proto，合成逐实例二值 mask），`run(img)` 得到每个实例的框 + 与框对齐的 mask 栅格，可按 alpha 混合可视化。

> 注：分割在 `docs/en/tutorials/` 下没有独立 `.rst`，但仓库提供了完整可编译示例 `examples/yolo11_seg`、量化脚本 `quantize_yolo11n_seg`、后处理器头 `dl_seg_yolo11_postprocessor.hpp`，本 recipe 即基于这三者。

## 触发意图

- "跑 yolo11 实例分割"
- "ESP-DL 分割"
- "yolo11n-seg 部署"
- "coco_seg 怎么用"
- "mask 检测"

## 前置条件

| 条件 | 要求 |
|---|---|
| 依赖 | `idf_component.yml` 含 `espressif/coco_seg`（`override_path: ../../../models/coco_seg`）与对应板级 BSP |
| 参考示例 | `examples/yolo11_seg`（S3/P4） |
| 量化脚本 | `examples/tutorial/how_to_quantize_model/quantize_yolo11n_seg` |
| 可视化（可选） | SD 卡（`bsp_sdcard_mount`）用于 `write_bmp` 导出叠加图 |

## 分步说明

### 1. 解码 JPEG 得到 `img_t`

与检测/姿态一致，用 `DL_IMAGE_PIX_TYPE_RGB888`：

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

### 2. 构造 `COCOSeg` 并推理

`COCOSeg` 继承自 `dl::detect::DetectWrapper`，后处理器是 `yolo11segPostProcessor`（解析 score/box/mask_coeff 三路输出，再用 32 维 proto `synthesize_masks` 合成与框对齐的二值 mask）：

```cpp
#include "coco_seg.hpp"      // 提供 COCOSeg
#include "esp_log.h"

const char *TAG = "yolo11n-seg";

COCOSeg *segment = new COCOSeg();
auto &seg_results = segment->run(img);   // 返回 std::list<dl::detect::result_t>&
```

分割结果复用 `dl::detect::result_t`，但用到了 `mask_coeff` 与 `mask` 两个字段（来自 `dl_detect_define.hpp`）：

```cpp
typedef struct {
    int category;
    float score;
    std::vector<int> box;              // [x1,y1,x2,y2]
    std::vector<int> keypoint;         // 分割模型不产生关键点
    std::vector<float> mask_coeff{};   // YOLO-seg mask 系数（内部用）
    std::vector<uint8_t> mask{};       // 与 box 对齐的二值 mask（原图尺度）
    // ...
} result_t;
```

> `yolo11segPostProcessor` 内部 `m_nm = 32`（mask proto 维度），mask 已合成到 `result_t.mask`，用户一般直接读 `mask`，无需关心 `mask_coeff`/proto。

### 3. 读取每个实例的框与 mask 像素数

```cpp
for (const auto &res : seg_results) {
    int box_w = res.box[2] - res.box[0];
    int box_h = res.box[3] - res.box[1];
    int mask_pixels = 0;
    for (uint8_t v : res.mask) {
        mask_pixels += v > 0;
    }
    ESP_LOGI(TAG,
             "[category: %d, score: %f, x1: %d, y1: %d, x2: %d, y2: %d, mask_pixels: %d, box_area: %d]",
             res.category, res.score,
             res.box[0], res.box[1], res.box[2], res.box[3],
             mask_pixels, box_w * box_h);
}
```

### 4. 把 mask 叠加到原图（alpha 混合）

`result_t.mask` 与 `box` 对齐：`mask[oy * box_w + ox]` 非零表示该框内像素属于该实例。按类别取色，做 alpha 混合：

```cpp
static constexpr float k_mask_alpha = 0.5f;

static void blend_instance_mask(dl::image::img_t &img, const dl::detect::result_t &res, float alpha)
{
    if (res.mask.empty()) return;
    int x1 = res.box[0], y1 = res.box[1], x2 = res.box[2], y2 = res.box[3];
    int box_w = x2 - x1, box_h = y2 - y1;
    if (box_w <= 0 || box_h <= 0) return;

    auto color = get_class_color_rgb(res.category);   // 自行实现：按 category 取 RGB
    uint8_t *data = static_cast<uint8_t *>(img.data);
    int row_step = img.row_step();
    float inv_alpha = 1.f - alpha;

    for (int oy = 0; oy < box_h; oy++) {
        int y = y1 + oy;
        if (y < 0 || y >= (int)img.height) continue;
        for (int ox = 0; ox < box_w; ox++) {
            if (!res.mask[oy * box_w + ox]) continue;   // 非实例像素跳过
            int x = x1 + ox;
            if (x < 0 || x >= (int)img.width) continue;
            uint8_t *px = data + y * row_step + x * 3;
            px[0] = (uint8_t)(px[0] * inv_alpha + color[0] * alpha);
            px[1] = (uint8_t)(px[1] * inv_alpha + color[1] * alpha);
            px[2] = (uint8_t)(px[2] * inv_alpha + color[2] * alpha);
        }
    }
}
```

### 5. 画框 + 导出 BMP（需 SD 卡）

`draw_hollow_rectangle` / `write_bmp` 由 `dl_image.hpp` 聚合头提供（实现在 `dl_image_draw.cpp` / `dl_image_bmp.cpp`）：

```cpp
#include "dl_image.hpp"       // draw_hollow_rectangle, write_bmp
#include "bsp/esp-bsp.h"

static constexpr const char *k_result_bmp_path = "/sdcard/yolo11_seg_result.bmp";

ESP_ERROR_CHECK(bsp_sdcard_mount());
// ... run + blend ...
for (const auto &res : seg_results) {
    dl::image::draw_hollow_rectangle(img, res.box[0], res.box[1], res.box[2], res.box[3],
                                     get_class_color_rgb(res.category), 2);
}
if (dl::image::write_bmp(img, k_result_bmp_path) == ESP_OK) {
    ESP_LOGI(TAG, "Visualization saved to %s", k_result_bmp_path);
}
ESP_ERROR_CHECK(bsp_sdcard_unmount());
```

### 6. 释放内存

```cpp
delete segment;
heap_caps_free(img.data);
```

### 7. 模型类型与阈值

`COCOSeg` 支持的模型类型（来自 `models/coco_seg/coco_seg.hpp`）：

```cpp
typedef enum { YOLO11N_SEG_S8_V1 } model_type_t;
COCOSeg(model_type_t model_type =
            static_cast<model_type_t>(CONFIG_DEFAULT_COCO_SEG_MODEL),
        bool lazy_load = true);
```

`coco_seg::Yolo11nSeg` 默认阈值：`default_score_thr = 0.25`，`default_nms_thr = 0.7`。

### 8. 示例 `main/CMakeLists.txt` 关键片段

```cmake
set(src_dirs ./)
set(include_dirs ./)
set(requires coco_seg)

if (IDF_TARGET STREQUAL "esp32s3")
    list(APPEND requires esp32_s3_eye_noglib esp_lcd)
elseif (IDF_TARGET STREQUAL "esp32p4")
    list(APPEND requires esp32_p4_function_ev_board_noglib esp_lcd)
endif()

set(embed_files "bus.jpg")
idf_component_register(SRC_DIRS ${src_dirs} INCLUDE_DIRS ${include_dirs}
                       REQUIRES ${requires} EMBED_FILES ${embed_files})
```

## 常见错误

| 错误 | 原因 | 解决方法 |
|---|---|---|
| `mask` 为空 | 用了 `COCODetect` 而非 `COCOSeg` | 分割必须用 `COCOSeg`/`coco_seg.hpp` |
| mask 错位 | `mask` 与 `box` 对齐，按 `box_w*box_h` 索引 | 用 `mask[oy*box_w+ox]`，坐标加 `box` 左上角偏移 |
| mask 越界写显存 | 未做图像边界裁剪 | 混合时 clip `x`/`y` 到 `[0,width/height)` |
| BMP 写入失败 | 未挂载 SD 卡 | 先 `bsp_sdcard_mount()`；或仅内存叠加不导出 |
| 内存不足 | mask 按实例数 × 框面积分配 | 调高 `score_thr` 减少实例；SD 卡加载模型 |
| 想要逐像素类别图 | 这是**实例**分割（每实例一 mask） | 语义分割需换模型，当前 ESP-DL 模型库以实例分割为主 |

## 参考

- `examples/yolo11_seg/main/app_main.cpp`（含 mask 混合与 BMP 导出完整实现）
- `examples/yolo11_seg/main/CMakeLists.txt`
- `examples/tutorial/how_to_quantize_model/quantize_yolo11n_seg`
- `esp-dl/vision/detect/dl_seg_yolo11_postprocessor.hpp`（`yolo11segPostProcessor`，`m_nm = 32`）
- `esp-dl/vision/detect/dl_detect_define.hpp`（`result_t.mask` / `mask_coeff`）
- `esp-dl/vision/image/dl_image_draw.hpp`（`draw_hollow_rectangle`）
- `esp-dl/vision/image/dl_image_bmp.hpp`（`write_bmp`）
- `models/coco_seg/coco_seg.hpp`（`COCOSeg`、`coco_seg::Yolo11nSeg`、默认阈值）
