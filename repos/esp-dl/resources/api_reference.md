# ESP-DL C++ API Quick Reference

> 所有签名取自仓库 `esp-dl/dl/` 头文件与 `docs/en/api_reference/`。命名空间 `dl`（视觉为 `dl::image` / `dl::detect`）。

## 头文件一览

| 头文件 | 提供 |
|---|---|
| `dl_model_base.hpp` | `dl::Model` |
| `dl_tensor_base.hpp` | `dl::TensorBase`, `dl::ExponentInfo`, `dtype_t`, `quantize/dequantize` |
| `dl_memory_manager.hpp` | `dl::MemoryManagerBase`, `dl::TensorInfo`, `dl::MemoryChunk`, `MEMORY_MANAGER_GREEDY` |
| `dl_model_context.hpp` | `dl::ModelContext`（执行上下文） |
| `dl_define.hpp` | `quant_type_t`, `activation_type_t`, `runtime_mode_t`, `DL_SCALE`, `DL_RESCALE` |
| `fbs_model.hpp` (`fbs` 命名空间) | `model_location_type_t`, `FbsModel`, `FbsLoader` |
| `dl_image_jpeg.hpp` | JPEG 编解码 |
| `dl_image_define.hpp` | `img_t`, `jpeg_img_t`, `pix_type_t` |

## dl::Model（`dl_model_base.hpp`）

```cpp
namespace dl {
typedef enum { MEMORY_MANAGER_GREEDY = 0, LINEAR_MEMORY_MANAGER = 1 } memory_manager_t;

class Model {
public:
    // 构造（任选其一；构造即完成 load + build + 内存规划）
    Model(const char *rodata_or_partition_label_or_path,
          fbs::model_location_type_t location = fbs::MODEL_LOCATION_IN_FLASH_RODATA,
          int max_internal_size = 0,
          memory_manager_t mm_type = MEMORY_MANAGER_GREEDY,
          const uint8_t *key = nullptr,
          bool param_copy = true,
          const std::map<std::string, std::vector<int>> &input_shapes = {});

    // 多模型打包：按 index 或 name 选择
    Model(const char *..., int model_index, fbs::model_location_type_t location = ..., ...);
    Model(const char *..., const char *model_name, fbs::model_location_type_t location = ..., ...);

    // 由已加载的 FbsModel 构造
    Model(fbs::FbsModel *fbs_model,
          int internal_size = 0,
          memory_manager_t mm_type = MEMORY_MANAGER_GREEDY,
          const std::map<std::string, std::vector<int>> &input_shapes = {});

    virtual ~Model();

    // 显式 load / build（构造时已自动调用，通常无需手动）
    virtual esp_err_t load(const char *..., fbs::model_location_type_t location = ...,
                           const uint8_t *key = nullptr, bool param_copy = true);
    virtual esp_err_t load(const char *..., fbs::model_location_type_t location = ...,
                           int model_index = 0, const uint8_t *key = nullptr, bool param_copy = true);
    virtual esp_err_t load(const char *..., fbs::model_location_type_t location = ...,
                           const char *model_name = nullptr, const uint8_t *key = nullptr, bool param_copy = true);
    virtual esp_err_t load(fbs::FbsModel *fbs_model);

    virtual void build(size_t max_internal_size,
                       memory_manager_t mm_type = MEMORY_MANAGER_GREEDY,
                       bool preload = false,
                       const std::map<std::string, std::vector<int>> &input_shapes = {});

    // 运行
    virtual void run(runtime_mode_t mode = RUNTIME_MODE_SINGLE_CORE);
    virtual void run(TensorBase *input, runtime_mode_t mode = RUNTIME_MODE_SINGLE_CORE);
    virtual void run(std::map<std::string, TensorBase *> &user_inputs,
                     runtime_mode_t mode = RUNTIME_MODE_SINGLE_CORE,
                     std::map<std::string, TensorBase *> user_outputs = {});
    // 分段执行（调用方管理 stage 进度）
    virtual void run(int stage_num, int stage_index, runtime_mode_t mode = RUNTIME_MODE_SINGLE_CORE);

    void minimize();

    // 验证与剖析
    esp_err_t test();                                       // 需模型含 test 值
    std::map<std::string, mem_info_t> get_memory_info();
    std::map<std::string, module_info> get_module_info();
    void print_module_info(const std::map<std::string, module_info> &info,
                           bool sort_module_by_latency = false);
    void profile_memory();
    void profile_module(bool sort_module_by_latency = false);
    void profile(bool sort_module_by_latency = false);     // = profile_memory + profile_module

    // 输入/输出/中间张量
    virtual std::map<std::string, TensorBase *> &get_inputs();
    virtual TensorBase *get_input();
    virtual TensorBase *get_input(const std::string &name);
    virtual TensorBase *get_intermediate(const std::string &name);
    virtual std::map<std::string, TensorBase *> &get_outputs();
    virtual TensorBase *get_output();
    virtual TensorBase *get_output(const std::string &name);

    std::string get_metadata_prop(const std::string &key);
    virtual void print();
    virtual void reset();                                   // 重置 context 与所有 module，不重新加载
    virtual fbs::FbsModel *get_fbs_model();
};
} // namespace dl
```

## 模型位置枚举（`fbs::model_location_type_t`，来自 `fbs_model.hpp`）

```cpp
namespace fbs {
typedef enum {
    MODEL_LOCATION_IN_FLASH_RODATA     = 0,  // 嵌入 .rodata
    MODEL_LOCATION_IN_FLASH_PARTITION  = 1,  // SPIFFS 数据分区
    MODEL_LOCATION_IN_SDCARD           = 2,  // SD 卡文件
    MODEL_LOCATION_MAX = MODEL_LOCATION_IN_SDCARD,
} model_location_type_t;
}
```

## dl::TensorBase（`dl_tensor_base.hpp`）

```cpp
namespace dl {

// 量化指数信息（per-tensor 或 per-channel）
class ExponentInfo {
public:
    ExponentInfo(int exponent = 0);
    ExponentInfo(const std::vector<int> &exponents);   // >1 个时为 per-channel
    int get(int ch = -1) const;                        // ch<0 返回 per-tensor 值
    operator int() const;                              // 隐式转 per-tensor 值
    bool is_per_channel() const;
    int channel_size() const;
    const int *data() const;
};

// 数据类型（与 FlatBuffers dtype 对齐）
typedef enum {
    DATA_TYPE_UNDEFINED = 0, DATA_TYPE_FLOAT = 1, DATA_TYPE_UINT8 = 2, DATA_TYPE_INT8 = 3,
    DATA_TYPE_UINT16 = 4, DATA_TYPE_INT16 = 5, DATA_TYPE_INT32 = 6, DATA_TYPE_INT64 = 7,
    DATA_TYPE_STRING = 8, DATA_TYPE_BOOL = 9, DATA_TYPE_FLOAT16 = 10, DATA_TYPE_DOUBLE = 11,
    DATA_TYPE_UINT32 = 12, DATA_TYPE_UINT64 = 13,
} dtype_t;

// 全局辅助
template <typename RT, typename T = float> RT quantize(T input, float inv_scale);   // 注意是 inverse scale
template <typename T, typename RT = float> RT dequantize(T input, float scale);      // 注意是 scale
size_t dtype_sizeof(dtype_t dtype);
const char *dtype_to_string(dtype_t dtype);

class TensorBase {
public:
    int size;                     // 含 padding 的元素数
    std::vector<int> shape;
    dtype_t dtype;
    ExponentInfo exponent;        // 量化指数（per-tensor 或 per-channel）
    bool auto_free;
    std::vector<int> axis_offset;
    void *data;
    void *cache;                  // preload 缓存指针
    uint32_t caps;                // MALLOC_CAP_* 标志

    TensorBase(std::vector<int> shape, const void *element,
               int exponent = 0, dtype_t dtype = DATA_TYPE_FLOAT,
               bool deep = true, uint32_t caps = MALLOC_CAP_DEFAULT);
    TensorBase(std::vector<int> shape, const void *element,
               const std::vector<int> &exponents,
               dtype_t dtype = DATA_TYPE_FLOAT,
               bool deep = true, uint32_t caps = MALLOC_CAP_DEFAULT);
    virtual ~TensorBase();

    bool assign(TensorBase *tensor);                          // 整体量化/反量化赋值
    bool assign(std::vector<int> shape, const void *element, int exponent, dtype_t dtype);

    int get_size();
    int get_aligned_size();                                   // 按 16 字节对齐后的元素数
    size_t get_dtype_bytes();
    int get_bytes();                                          // = size * dtype_bytes
    int get_aligned_bytes();
    const char *get_dtype_string();
    virtual void *get_element_ptr();
    template <typename T> T *get_element_ptr();
    TensorBase &set_element_ptr(void *data);
    std::vector<int> get_shape();
    TensorBase &set_shape(const std::vector<int> shape);
    int get_exponent();
    dtype_t get_dtype();
    uint32_t get_caps();
    TensorBase *reshape(std::vector<int> shape);
    template <typename T> TensorBase *flip(const std::vector<int> &axes);
    TensorBase *transpose(TensorBase *input, std::vector<int> perm = {});
    bool is_same_shape(TensorBase *tensor);
    bool equal(TensorBase *tensor, float epsilon = 1e-6, bool verbose = false);
    TensorBase *slice(const std::vector<int> &start, const std::vector<int> &end,
                      const std::vector<int> &axes = {}, const std::vector<int> &step = {});
    static void slice(TensorBase *input, TensorBase *output,
                      const std::vector<int> &start, const std::vector<int> &end,
                      const std::vector<int> &axes = {}, const std::vector<int> &step = {});
    TensorBase *pad(TensorBase *input, const std::vector<int> &pads,
                    const padding_mode_t mode, TensorBase *const_value = nullptr);
    int get_element_index(const std::vector<int> &axis_index);
    std::vector<int> get_element_coordinates(int index);
    template <typename T> T get_element(int index);
    template <typename T> T get_element(const std::vector<int> &axis_index);
    void memset(int value);
    void rand();                                              // 用硬件 RNG 填充
    size_t set_preload_addr(void *addr, size_t size);
    virtual void preload();
    void reset_bias_layout(quant_type_t op_quant_type, bool is_depthwise);  // 仅 Conv
    void push(TensorBase *new_tensor, int dim);              // 流式堆叠
    virtual void print(bool print_data = false);
};
} // namespace dl
```

## 量化 / 激活 / 运行模式（`dl_define.hpp`）

```cpp
namespace dl {
typedef enum {
    QUANT_TYPE_NONE,
    QUANT_TYPE_FLOAT32,
    QUANT_TYPE_SYMM_8BIT,    // 对称 8bit（per tensor）
    QUANT_TYPE_SYMM_16BIT,   // 对称 16bit（per tensor）
    QUANT_TYPE_SYMM_32BIT,
} quant_type_t;

typedef enum { Linear, ReLU, LeakyReLU, PReLU } activation_type_t;

typedef enum {
    RUNTIME_MODE_AUTO = 0,          // 自动单/多核
    RUNTIME_MODE_SINGLE_CORE = 1,   // 单核
    RUNTIME_MODE_MULTI_CORE = 2,    // 双核（ESP32-S3 / ESP32-P4）
} runtime_mode_t;
}

#define DL_SCALE(exp)    (((exp) > 0) ? (1 << (exp))            : ((float)1.0 / (1 << -(exp))))   // 2^exp
#define DL_RESCALE(exp)  (((exp) > 0) ? ((float)1.0 / (1 << (exp))) : (1 << -(exp)))              // 1/2^exp
```

## 内存管理（`dl_memory_manager.hpp`）

```cpp
namespace dl {
class MemoryManagerBase {
public:
    int alignment;   // 默认 16
    MemoryManagerBase(int alignment = 16);
    virtual bool alloc(fbs::FbsModel *fbs_model,
                       std::vector<dl::module::Module *> &execution_plan,
                       ModelContext *context,
                       const std::map<std::string, std::vector<int>> &input_shapes = {}) = 0;
};
// 当前唯一实现：MEMORY_MANAGER_GREEDY（见 dl_memory_manager_greedy.hpp）

class TensorInfo { /* 规划用的张量元信息：name, shape, dtype, size, lifetime, offset ... */ };
class MemoryChunk { /* 模拟内存块：size, is_free, offset, alignment, tensor */ };

void print_memory_list(const char *tag, std::list<MemoryChunk *> &memory_list);
void sort_memory_list(std::list<MemoryChunk *> &memory_list);
}
```

## 视觉：图像（`dl_image_*`）

```cpp
namespace dl::image {

typedef enum {
    DL_IMAGE_PIX_TYPE_RGB888 = 0, DL_IMAGE_PIX_TYPE_RGB888_QINT8, DL_IMAGE_PIX_TYPE_RGB888_QINT16,
    DL_IMAGE_PIX_TYPE_BGR888, DL_IMAGE_PIX_TYPE_BGR888_QINT8, DL_IMAGE_PIX_TYPE_BGR888_QINT16,
    DL_IMAGE_PIX_TYPE_GRAY, DL_IMAGE_PIX_TYPE_GRAY_QINT8, DL_IMAGE_PIX_TYPE_GRAY_QINT16,
    DL_IMAGE_PIX_TYPE_RGB565LE, DL_IMAGE_PIX_TYPE_RGB565BE,
    DL_IMAGE_PIX_TYPE_BGR565LE, DL_IMAGE_PIX_TYPE_BGR565BE,
    DL_IMAGE_PIX_TYPE_HSV, DL_IMAGE_PIX_TYPE_HSV_MASK,
    DL_IMAGE_PIX_TYPE_YUYV, DL_IMAGE_PIX_TYPE_UYVY,
} pix_type_t;

typedef struct img_s {
    void *data;
    uint16_t width;
    uint16_t height;
    pix_type_t pix_type;
    int channel() const;
    size_t bytes() const;
    bool pix_quant() const;
    int col_step() const;
    int row_step() const;
} img_t;

typedef struct { void *data; size_t data_len; } jpeg_img_t;

// JPEG（dl_image_jpeg.hpp）
img_t       sw_decode_jpeg(const jpeg_img_t &jpeg_img, pix_type_t pix_type);
jpeg_img_t  sw_encode_jpeg_base(const img_t &img, uint8_t quality = 80);
jpeg_img_t  sw_encode_jpeg(const img_t &img, uint8_t quality = 80);
#if CONFIG_SOC_JPEG_CODEC_SUPPORTED
img_t       hw_decode_jpeg(const jpeg_img_t &jpeg_img, pix_type_t pix_type,
                           int timeout_ms = 60,
                           jpeg_yuv_rgb_conv_std_t yuv_rgb_conv_std = JPEG_YUV_RGB_CONV_STD_BT601);
jpeg_img_t  hw_encode_jpeg_base(const img_t &img, uint8_t quality = 80,
                                int timeout_ms = 70,
                                jpeg_down_sampling_type_t rgb_sub_sample_method = JPEG_DOWN_SAMPLING_YUV420);
jpeg_img_t  hw_encode_jpeg(const img_t &img, uint8_t quality = 80,
                           int timeout_ms = 70,
                           jpeg_down_sampling_type_t rgb_sub_sample_method = JPEG_DOWN_SAMPLING_YUV420);
#endif
esp_err_t   write_jpeg(const jpeg_img_t &img, const char *file_name);
jpeg_img_t  read_jpeg(const char *file_name);

} // namespace dl::image
```

## 检测后处理（模型组件，示例：`models/coco_detect/coco_detect.hpp`）

```cpp
namespace coco_detect {
class Yolo11n : public dl::detect::DetectImpl {
public:
    static constexpr float default_score_thr = 0.25f;
    static constexpr float default_nms_thr   = 0.7f;
    Yolo11n(const char *model_name, float score_thr, float nms_thr);
};
}

class COCODetect : public dl::detect::DetectWrapper {
public:
    typedef enum { YOLO11N_S8_V1, YOLO11N_320_S8_V1 } model_type_t;
    COCODetect(model_type_t model_type = static_cast<model_type_t>(CONFIG_DEFAULT_COCO_DETECT_MODEL),
               bool lazy_load = true);
    // run(img) 返回 std::vector<result_t>&，元素含 category / score / box[4]
};
```

> `dl::detect::DetectWrapper` / `DetectImpl` 及检测结果结构定义在 `esp-dl/vision/detect/` 与各模型组件中。其它模型组件（`human_face_detect`、`pedestrian_detect`、`cat_detect`、`mobilenetv2_cls` 等）提供各自的 Wrapper 类，用法一致。

## 视觉：检测结果结构（`dl_detect_define.hpp`，pose/seg 复用同一 struct）

```cpp
namespace dl::detect {
typedef struct {
    int category;                    // 类别索引
    float score;                     // 分数
    std::vector<int> box;            // [left_up_x, left_up_y, right_down_x, right_down_y]
    std::vector<int> keypoint;       // pose: [x0,y0, x1,y1, ...] 成对；detect/seg 不产生
    std::vector<float> mask_coeff{}; // seg: YOLO-seg mask 系数（内部用）
    std::vector<uint8_t> mask{};     // seg: 与 box 对齐的二值 mask（原图尺度）
    void limit_box(int width, int height);
    void limit_keypoint(int width, int height);
    int box_area() const;
} result_t;

typedef struct { int stride_y, stride_x, offset_y, offset_x; } anchor_point_stage_t;
} // namespace dl::detect
```

## 视觉：姿态 / 分割后处理器（`dl_pose_yolo11_postprocessor.hpp` / `dl_seg_yolo11_postprocessor.hpp`）

```cpp
namespace dl::detect {
// YOLO11n-pose：解析 score/box/kpt 三路，按 score 过滤后解码 17 关键点 + NMS
class yolo11posePostProcessor : public AnchorPointDetectPostprocessor {
public:
    void postprocess() override;
    using AnchorPointDetectPostprocessor::AnchorPointDetectPostprocessor;
};

// YOLO11n-seg：解析 score/box/mask_coeff，用 32 维 proto 合成逐实例 mask
class yolo11segPostProcessor : public AnchorPointDetectPostprocessor {
private:
    static constexpr int m_nm = 32;   // mask proto 维度
public:
    void postprocess() override;
    using AnchorPointDetectPostprocessor::AnchorPointDetectPostprocessor;
};
} // namespace dl::detect
```

## 视觉：模型组件（pose / seg，`models/coco_pose/coco_pose.hpp`、`models/coco_seg/coco_seg.hpp`）

```cpp
namespace coco_pose {
class Yolo11nPose : public dl::detect::DetectImpl {
public:
    static constexpr float default_score_thr = 0.25;
    static constexpr float default_nms_thr   = 0.7;
    Yolo11nPose(const char *model_name, float score_thr, float nms_thr);
};
} // namespace coco_pose
class COCOPose : public dl::detect::DetectWrapper {
public:
    typedef enum { YOLO11N_POSE_S8_V1, YOLO11N_POSE_S8_V2 } model_type_t;
    COCOPose(model_type_t model_type = static_cast<model_type_t>(CONFIG_DEFAULT_COCO_POSE_MODEL),
             bool lazy_load = true);
    // run(img) 返回 std::list<dl::detect::result_t>&，元素含 box + keypoint(34)
};

namespace coco_seg {
class Yolo11nSeg : public dl::detect::DetectImpl {
public:
    static constexpr float default_score_thr = 0.25;
    static constexpr float default_nms_thr   = 0.7;
    Yolo11nSeg(const char *model_name, float score_thr, float nms_thr);
};
} // namespace coco_seg
class COCOSeg : public dl::detect::DetectWrapper {
public:
    typedef enum { YOLO11N_SEG_S8_V1 } model_type_t;
    COCOSeg(model_type_t model_type = static_cast<model_type_t>(CONFIG_DEFAULT_COCO_SEG_MODEL),
            bool lazy_load = true);
    // run(img) 返回 std::list<dl::detect::result_t>&，元素含 box + mask（与 box 对齐）
};
```

## 视觉：分类（`dl_cls_base.hpp` / `dl_cls_postprocessor.hpp` / `imagenet_cls_postprocessor.hpp` / `dl_cls_define.hpp`）

```cpp
namespace dl::cls {
typedef struct {
    const char *cat_name;   // ImageNet 类别名
    float score;            // softmax 或原始 logit
} result_t;

class Cls {   // 抽象基类
public:
    virtual ~Cls() {};
    virtual std::vector<result_t> &run(const dl::image::img_t &img) = 0;
    virtual Cls &set_topk(int topk) = 0;
    virtual Cls &set_score_thr(float score_thr) = 0;
    virtual dl::Model *get_raw_model() = 0;
};

class ClsWrapper : public Cls { /* 持有 Cls*，转发 run/set_topk/set_score_thr */ };
class ClsImpl   : public Cls { /* 持有 Model* + ImagePreprocessor* + ClsPostprocessor* */ };

class ClsPostprocessor {
public:
    ClsPostprocessor(Model *model, const int topk, const float score_thr,
                     bool need_softmax, const std::string &output_name);
    virtual std::vector<result_t> &postprocess();
    void set_topk(int topk);
    void set_score_thr(float score_thr);
};

class ImageNetClsPostprocessor : public ClsPostprocessor {
public:
    ImageNetClsPostprocessor(Model *model, const int top_k, const float score_thr,
                             bool need_softmax, const std::string &output_name = "");
};
} // namespace dl::cls
```

## 视觉：分类模型组件（`models/imagenet_cls/imagenet_cls.hpp`）

```cpp
namespace imagenet_cls {
class MobileNetV2 : public dl::cls::ClsImpl {
public:
    static constexpr int   default_topk      = 5;
    static constexpr float default_score_thr = std::numeric_limits<float>::lowest();   // 不过滤
    MobileNetV2(const char *model_name, int topk, float score_thr);
};
} // namespace imagenet_cls
class ImageNetCls : public dl::cls::ClsWrapper {
public:
    typedef enum { MOBILENETV2_S8_V1 } model_type_t;
    ImageNetCls(model_type_t model_type = static_cast<model_type_t>(CONFIG_DEFAULT_IMAGENET_CLS_MODEL),
                bool lazy_load = true);
    // run(img) 返回 std::vector<dl::cls::result_t>&，元素为 {cat_name, score}，无 box
};
```

## 视觉：图像绘制 / BMP（`dl_image_draw.hpp` / `dl_image_bmp.hpp`，经 `dl_image.hpp` 聚合）

```cpp
namespace dl::image {
void      draw_hollow_rectangle(const img_t &img, int x1, int y1, int x2, int y2,
                                const std::vector<uint8_t> &color, uint8_t line_width);
void      draw_point(const img_t &img, int x, int y, const std::vector<uint8_t> &color, uint8_t radius);
esp_err_t write_bmp(const img_t &img, const char *file_name);   // 需先挂载 SD 卡等 VFS
esp_err_t write_bmp_base(const img_t &img, const char *file_name);
} // namespace dl::image
```

## ESP-PPQ 高级量化设置（TQT / 混合精度 / 逐层均衡）

```python
from esp_ppq import QuantizationSettingFactory
from esp_ppq.api import espdl_quantize_onnx, get_target_platform, ENABLE_CUDA_KERNEL

quant_setting = QuantizationSettingFactory.espdl_setting()

# —— TQT（Trained Quantization Thresholds，无需标签）——
quant_setting.tqt_optimization = True
quant_setting.tqt_optimization_setting.steps              = 500      # 每 block 步数
quant_setting.tqt_optimization_setting.lr                  = 1e-5     # 学习率
quant_setting.tqt_optimization_setting.block_size          = 4        # 图分块大小
quant_setting.tqt_optimization_setting.is_scale_trainable  = True     # False=只微调权重
quant_setting.tqt_optimization_setting.int_lambda          = 0.0      # [0,1]，拉 alpha 向整数
quant_setting.tqt_optimization_setting.gamma               = 0.0      # MSE(浮点,量化) 正则
quant_setting.tqt_optimization_setting.collecting_device   = "cpu"    # 或 "cuda"
quant_setting.tqt_optimization_setting.interested_layers   = []       # 空=全部可训算子

# —— 混合精度：把坏层派发到 int16 ——
quant_setting.dispatching_table.append("<onnx_layer_name>", get_target_platform(TARGET, 16))

# —— 逐层权重均衡（需 ReLU；ReLU6 要先转 ReLU）——
quant_setting.equalization = True
quant_setting.equalization_setting.iterations       = 6
quant_setting.equalization_setting.value_threshold  = 0.5
quant_setting.equalization_setting.opt_level        = 2
quant_setting.equalization_setting.interested_layers = None

# GPU 加速内核（TQT 用）
with ENABLE_CUDA_KERNEL():
    quant_ppq_graph = espdl_quantize_onnx(..., setting=quant_setting, ...)
```
