# 图像分类端到端（JPEG 解码 → 模型 → Top-K 后处理）

> **适用摘要**: 使用 ESP-DL 跑一个 MobileNetV2/ImageNet 分类模型：软件解码 JPEG 得到 `img_t`，构造 `ImageNetCls`（内部 `ImageNetClsPostprocessor` 输出 Top-K 类别，可选 softmax），`run(img)` 得到 `(类别名, 分数)` 列表。**分类不输出框，与检测后处理完全不同**。

## 触发意图

- "跑图像分类"
- "MobileNetV2 部署"
- "ESP-DL imagenet 分类"
- "imagenet_cls 怎么用"
- "top-1 / top-5 分类"

## 前置条件

| 条件 | 要求 |
|---|---|
| 依赖 | `idf_component.yml` 含 `espressif/imagenet_cls`（`override_path: ../../../models/imagenet_cls`）与对应板级 BSP |
| 参考示例 | `examples/mobilenetv2_cls`（S3/P4） |
| 量化脚本 | `examples/tutorial/how_to_quantize_model/quantize_mobilenetv2/quantize_onnx_model.py` |

## 分步说明

### 1. 解码 JPEG 得到 `img_t`

分类输入同样用 `DL_IMAGE_PIX_TYPE_RGB888`，预处理（resize/normalize/quantize）由 `ImagePreprocessor` 在 `ClsImpl` 内部完成，与训练时的 transform 对齐：

```cpp
#include "dl_image_jpeg.hpp"

extern const uint8_t cat_jpg_start[] asm("_binary_cat_jpg_start");
extern const uint8_t cat_jpg_end[]   asm("_binary_cat_jpg_end");

dl::image::jpeg_img_t jpeg_img = {
    .data     = (void *)cat_jpg_start,
    .data_len = (size_t)(cat_jpg_end - cat_jpg_start),
};
auto img = dl::image::sw_decode_jpeg(jpeg_img, dl::image::DL_IMAGE_PIX_TYPE_RGB888);
```

### 2. 构造 `ImageNetCls` 并推理

`ImageNetCls` 继承自 `dl::cls::ClsWrapper`，封装模型加载、预处理与 `ImageNetClsPostprocessor` 后处理：

```cpp
#include "imagenet_cls.hpp"   // 提供 ImageNetCls
#include "esp_log.h"

const char *TAG = "mobilenetv2_cls";

ImageNetCls *cls = new ImageNetCls();          // 默认模型类型 + lazy_load
auto &results = cls->run(img);                 // 返回 std::vector<dl::cls::result_t>&

for (const auto &res : results) {
    ESP_LOGI(TAG, "category: %s, score: %f", res.cat_name, res.score);
}
```

分类结果结构（来自 `dl_cls_define.hpp`）——**没有 box/keypoint**：

```cpp
namespace dl::cls {
typedef struct {
    const char *cat_name;   // ImageNet 类别名（字符串）
    float score;            // softmax 或原始 logit 分数
} result_t;
}
```

### 3. 调节 Top-K 与分数阈值（链式 setter）

`Cls` 基类提供链式设置（来自 `dl_cls_base.hpp`）：

```cpp
cls->set_topk(5)->set_score_thr(0.01f);
// 或在构造前后单独调用：
cls->set_topk(1);            // 只要 top-1
```

`imagenet_cls::MobileNetV2` 默认值（来自 `models/imagenet_cls/imagenet_cls.hpp`）：

```cpp
static constexpr int   default_topk      = 5;
static constexpr float default_score_thr = std::numeric_limits<float>::lowest();   // 不过滤
```

`ImageNetClsPostprocessor` 构造（来自 `imagenet_cls_postprocessor.hpp`）：

```cpp
ImageNetClsPostprocessor(Model *model, const int top_k, const float score_thr,
                         bool need_softmax, const std::string &output_name = "");
```

### 4. 释放内存

```cpp
delete cls;
heap_caps_free(img.data);   // sw_decode_jpeg 分配的图像数据需手动释放
```

### 5. 从 SD 卡加载模型（可选）

示例通过 Kconfig `CONFIG_IMAGENET_CLS_MODEL_IN_SDCARD` 切换模型来源：

```cpp
#if CONFIG_IMAGENET_CLS_MODEL_IN_SDCARD
#include "bsp/esp-bsp.h"
ESP_ERROR_CHECK(bsp_sdcard_mount());
#endif
// ... 解码 + 推理 ...
#if CONFIG_IMAGENET_CLS_MODEL_IN_SDCARD
ESP_ERROR_CHECK(bsp_sdcard_unmount());
#endif
```

### 6. 模型类型

`ImageNetCls` 支持的模型类型（来自 `models/imagenet_cls/imagenet_cls.hpp`）：

```cpp
typedef enum { MOBILENETV2_S8_V1 } model_type_t;
ImageNetCls(model_type_t model_type =
               static_cast<model_type_t>(CONFIG_DEFAULT_IMAGENET_CLS_MODEL),
            bool lazy_load = true);
```

### 7. 示例 `main/CMakeLists.txt` 关键片段

```cmake
set(src_dirs ./)
set(include_dirs ./)
set(requires imagenet_cls)

if (IDF_TARGET STREQUAL "esp32s3")
    list(APPEND requires esp32_s3_eye_noglib esp_lcd)
elseif (IDF_TARGET STREQUAL "esp32p4")
    list(APPEND requires esp32_p4_function_ev_board_noglib esp_lcd)
endif()

set(embed_files "cat.jpg")
idf_component_register(SRC_DIRS ${src_dirs} INCLUDE_DIRS ${include_dirs}
                       REQUIRES ${requires} EMBED_FILES ${embed_files})
```

### 8. 量化精度参考（MobileNetV2 / ImageNet）

`how_to_deploy_mobilenetv2.rst` 实测 Top-1 精度（float 基线 71.878%）：

| 量化方式 | target | Top-1 | Top-5 | 说明 |
|---|---|---|---|---|
| 8-bit per-channel | `esp32p4` | **71.150%** | 89.350% | 接近 float；P4 默认 Conv/Gemm per-channel |
| 8-bit per-tensor | `esp32s3` | 60.325% | 83.100% | S3 受 ISA 限制全部 per-tensor，掉点明显 |
| S3 + mixed precision（坏层 int16） | `esp32s3` | 69.225% | 88.700% | 把 `/features/features.1/conv/conv.0/...` 派发到 int16 |
| S3 + layerwise equalization | `esp32s3` | **69.800%** | 88.400% | 配合 ReLU6→ReLU，优于混合精度 |
| S3 + TQT | — | — | — | 见 `quantize_tqt.md`，MobileNetV2 TQT 后 Top-1 达 71.775% |

> **关键结论**：S3（per-tensor）默认 PTQ 掉点严重（60.5%），务必走混合精度 / 逐层均衡 / TQT 之一，见 `quantize_advanced_ptq.md` 与 `quantize_tqt.md`。P4 的 per-channel 已接近 float。

## 常见错误

| 错误 | 原因 | 解决方法 |
|---|---|---|
| 用了 `COCODetect`/检测 wrapper | 误把分类当检测 | 分类必须用 `ImageNetCls`/`ClsWrapper`，结果是 `(cat_name, score)` 无框 |
| `cat_name` 乱码或空 | 未启用 softmax 或 top_k 过滤掉 | 检查 `need_softmax`；调小 `score_thr` |
| Top-1 全错 | 预处理与训练不符 | ImageNet 标准预处理：Resize(256)→CenterCrop(224)→Normalize(mean/std) |
| S3 精度暴跌（~60%） | per-tensor 默认 PTQ | 走 mixed precision / equalization / TQT |
| ReLU6 模型做 equalization 无效 | ReLU6 不满足均衡条件 | 先 `convert_relu6_to_relu` 再量化 |
| 内存不足 | 大图 + 模型占满 PSRAM | 用 224×224 输入；SD 卡加载；`param_copy=false` |

## 参考

- `examples/mobilenetv2_cls/main/app_main.cpp`
- `examples/mobilenetv2_cls/main/CMakeLists.txt`
- `examples/tutorial/how_to_quantize_model/quantize_mobilenetv2/quantize_onnx_model.py`
- `docs/en/tutorials/how_to_deploy_mobilenetv2.rst`
- `esp-dl/vision/classification/dl_cls_base.hpp`（`Cls` / `ClsWrapper` / `ClsImpl`）
- `esp-dl/vision/classification/dl_cls_postprocessor.hpp`（`ClsPostprocessor`）
- `esp-dl/vision/classification/imagenet_cls_postprocessor.hpp`（`ImageNetClsPostprocessor`）
- `esp-dl/vision/classification/dl_cls_define.hpp`（`result_t`）
- `models/imagenet_cls/imagenet_cls.hpp`（`ImageNetCls`、`imagenet_cls::MobileNetV2`、默认 topk/score_thr）
