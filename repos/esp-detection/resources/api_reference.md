# esp-detection API Quick Reference

> 所有签名均来自仓库源码（`D:/esp-skill/espressif-repos/esp-detection/`）。未在源码出现的 API 一律不列出。

## Python — 训练 / 导出 / 量化

### train.py

```python
def Train(pretrained_path=None, dataset="cfg/datasets/coco_cat.yaml", imgsz=224, **kwargs):
    """
    训练 espdet_pico。内部：
      - tasks.parse_model = custom_parse_model
      - pretrained_path 为 None/'None' 时从 'cfg/models/espdet_pico.yaml' 新建模型，否则 YOLO(pretrained_path)
      - 默认 train_setting: epochs=1200, imgsz=imgsz, batch=128, device='cpu',
        optimizer='auto', close_mosaic=30, mosaic=1.0, mixup=0.0, copy_paste=0.1, rect=False
      - kwargs 覆盖 train_setting
    返回: model.train(**train_setting) 的 results（含 .save_dir）
    """
```

### deploy/export.py

```python
class ESP_Attention(Attention):
    def forward(self, x): ...   # 自实现的自注意力 forward（适配 ESP-DL 导出）

class ESP_Detect_Exporter(Exporter):
    @try_export
    def export_onnx(self, prefix=colorstr("ONNX:")):
        # 固定 opset_version=13；用 onnxsim（非 onnxslim）
        # output_names = ["box0","score0","box1","score1","box2","score2"]
        # dynamic 仅 QAT 用；部署 batch=1 静态
        ...

class ESP_YOLO(YOLO):
    def export(self, **kwargs): ...   # 用 ESP_Detect_Exporter

def Export(model_path, input_size):
    """
    注入 custom_parse_model → ESP_YOLO(model_path)
    → 把 ESPDetect.forward 绑定为 ESPDetect.export_onnx_forward
    → 把 Attention.forward 绑定为 ESP_Attention.forward
    → model.export(format='onnx', simplify=True, opset=13, imgsz=input_size)
    input_size: 标量（方形）或 [h, w]（非方形）
    """
```

### deploy/quantize.py

```python
class CaliDataset(torch.utils.data.Dataset):
    def __init__(self, path, img_shape=640):
        # img_shape 标量→(s,s)；列表→(h,w)
        # transform: ToTensor + Resize((h,w)) + Normalize(mean=[0,0,0], std=[1,1,1])
        # 仅加载 .jpg/.jpeg/.png/.bmp

def quant_espdet(onnx_path, target, num_of_bits, device, batchsz, imgsz,
                 calib_dir, espdl_model_path):
    """
    INPUT_SHAPE = [3, *imgsz]（imgsz 标量或 [h,w]）
    流程: onnx.load → onnxsim.simplify → shape_inference 回写
         → CaliDataset(calib_dir, img_shape=imgsz) → DataLoader(batch_size=batchsz)
         → QuantizationSettingFactory.espdl_setting()
         → Equalization: iterations=4, value_threshold=0.4, opt_level=2
         → espdl_quantize_onnx(calib_steps=32, input_shape=[1]+INPUT_SHAPE,
                                target=target, num_of_bits=num_of_bits, ...)
    返回: quant_ppq_graph
    """
```

### deploy/eval_quantized_model.py

```python
class QuantizedModelValidator(BaseValidator):
    # self.end2end = False  # 针对 espdet_pico
    def __call__(self, trainer=None, model=None, executor=None): ...

def make_quant_validator_class(executor):
    # 返回 QuantDetectionValidator（继承 DetectionValidator）
    ...

def ppq_graph_init(quant_func, imgsz, device, native_path=None):
    """
    case 1 (PTQ): ppq_graph = quant_func(imgsz)
    case 2 (QAT): ppq_graph = load_native_graph(native_path)
    返回: TorchExecutor(graph=ppq_graph, device=device)
    """

def ppq_graph_inference(executor, task, inputs, device):
    """
    task=='detect' 时:
      boxes_ls  = [graph_outputs[i] for i in range(0,6,2)]
      boxes  = cat([graph_outputs[2*i].view(bs,4,-1)   for i in range(3)], dim=-1)
      scores = cat([graph_outputs[2*i+1].view(bs,NC,-1) for i in range(3)], dim=-1)
      preds  = dict(boxes=boxes, scores=scores, feats=boxes_ls)
      detect_model = Detect(nc=NC, reg_max=1, end2end=False, ch=[32,64,128])
      detect_model.stride = [8.0, 16.0, 32.0]
      return detect_model._inference(preds)
    否则: NotImplementedError
    """
```

### espdet_run.py

```python
def rename_project(root_dir: Path, replacements: dict):
    # 处理 .cpp/.hpp/.txt/.yml 及名为 Kconfig 的文件
    # 先用 __PLACEHOLDER_i__ 保护所有 add_custom_command 整行，再做字符串替换，最后还原

def run(class_name, pretrained_path, dataset, size, target, calib_data, espdl, img):
    # size 必须是 len=2 的 list；h,w=size
    # h!=w → Train(..., rect=True)；否则 Train(...)
    # → Export → quant_espdet → git clone esp-dl → 复制模板 → rename_project → 拷 espdl 与 img
```

## Python — 自定义网络模块（nn/modules）

### nn/modules/esp_conv.py

```python
class DSConv(nn.Module):
    """Depthwise Separable Convolution."""
    def __init__(self, c1, c2, k=3, s=1, p=1, dilation=1, groups=1, act=nn.ReLU(inplace=False)):
        # depthwise = Conv2d(c1,c1,k,s,p,groups=c1) + BN + act
        # pointwise = Conv2d(c1,c2,1,1,0) + BN + act
    def forward(self, x): ...
```

### nn/modules/esp_block.py

```python
class C3k(C3):
    def __init__(self, c1, c2, n=1, shortcut=True, g=1, e=0.5, k=3): ...

class DSBottleneck(nn.Module):
    """用 DSConv 替换标准 Bottleneck 的 Conv。"""
    def __init__(self, c1, c2, shortcut=True, g=1, k=(3,3), e=0.5): ...

class DSC3(C3):
    def __init__(self, c1, c2, n=1, shortcut=True, g=1, e=0.5, k=3): ...

class DSC3k2(C2f):
    """C3k2 的 bottleneck 换成 DSBottleneck。"""
    def __init__(self, c1, c2, n=1, c3k=False, e=0.5, g=1, shortcut=True): ...

class ESPSerial(nn.Module):
    def __init__(self, c1, c2, n=1, shortcut=True, g=1, e=0.5): ...

class ESPSerialLite(nn.Module):
    def __init__(self, c1, c2, n=1, shortcut=False, g=1, e=0.5): ...

class ESPBlock(ESPSerial):
    def __init__(self, c1, c2, n=1, c3k=False, e=0.5, g=1, shortcut=True): ...

class ESPBlockLite(ESPSerialLite):
    def __init__(self, c1, c2, n=1, c3k=False, e=0.5, g=1, shortcut=False): ...
```

### nn/modules/esp_head.py

```python
class ESPDetect(Detect):
    def __init__(self, nc=1, ch=()):
        # super().__init__(nc=nc, reg_max=1, ch=ch)
        # reg_max=1; no = nc + reg_max*4
        # cv2: 3×[DSConv, DSConv, Conv2d(→4*reg_max)]   (CIoU 分支)
        # cv3: 3×[DWConv+Conv, DWConv+Conv, Conv2d(→nc)] (CLS 分支)
        # dfl = DFL(reg_max) if reg_max>1 else Identity
    def export_onnx_forward(self, x):
        # 返回 box0,score0,box1,score1,box2,score2（3 个尺度）
```

### nn/modules/__init__.py 导出

```python
__all__ = ("DSConv","DSBottleneck","DSC3k2","ESPSerial","ESPSerialLite","ESPBlock","ESPBlockLite","ESPDetect")
```

### nn/esp_tasks.py

```python
def custom_parse_model(d, ch, verbose=True):
    """
    替换 ultralytics.nn.tasks.parse_model：把 DSConv/DSBottleneck/DSC3k2/ESPBlock/ESPBlockLite/
    ESPSerial/ESPSerialLite 加入 base_modules 与 repeat_modules；把 ESPDetect 加入检测头集合。
    返回: (torch.nn.Sequential(*layers), sorted(save))
    """
```

### data/esp_dataset.py

```python
class YOLOPosNegDataset(YOLODataset):
    """支持正/负样本混合采样。依赖 dataset yaml 的 negative_setting:
       neg_ratio, use_extra_neg, extra_neg_sources{dir:count}, fix_dataset_length"""

class YOLOWeightedDataset(YOLODataset):
    """按类别实例数倒数加权采样，缓解类别不均衡。"""
```

## C++ — 芯片端（来自 deploy/espdet_*_template）

### espdet_detect.hpp

```cpp
namespace espdet_detect {
class ESPDet : public dl::detect::DetectImpl {
public:
    static inline constexpr float default_score_thr = 0.25;
    static inline constexpr float default_nms_thr   = 0.7;
    ESPDet(const char *model_name, float score_thr, float nms_thr);
};
} // namespace espdet_detect

class ESPDetDetect : public dl::detect::DetectWrapper {
public:
    typedef enum {
        ESPDET_PICO_imgH_imgW_CUSTOM,   // 由 rename_project 替换 imgH/imgW/CUSTOM
    } model_type_t;
    ESPDetDetect(model_type_t model_type = static_cast<model_type_t>(CONFIG_DEFAULT_ESPDET_DETECT_MODEL),
                 bool lazy_load = true);
private:
    void load_model() override;
    model_type_t m_model_type;
};
```

### espdet_detect.cpp（构造逻辑要点）

```cpp
ESPDet::ESPDet(const char *model_name, float score_thr, float nms_thr)
{
    // 非 sdcard: dl::Model(path, model_name, fbs::model_location_type_t(CONFIG_ESPDET_DETECT_MODEL_LOCATION))
    // sdcard:    dl::Model(sd_path.c_str(), fbs::MODEL_LOCATION_IN_SDCARD)
    m_model->minimize();

#if CONFIG_IDF_TARGET_ESP32P4
    m_image_preprocessor = new dl::image::ImagePreprocessor(m_model, {0,0,0}, {255,255,255});
#else
    m_image_preprocessor = new dl::image::ImagePreprocessor(
        m_model, {0,0,0}, {255,255,255}, dl::image::DL_IMAGE_CAP_RGB565_BIG_ENDIAN);
#endif
    m_image_preprocessor->enable_letterbox({114,114,114});

    m_postprocessor = new dl::detect::ESPDetPostProcessor(
        m_model, m_image_preprocessor, score_thr, nms_thr, 10,
        {{8,8,4,4},{16,16,8,8},{32,32,16,16}});
}
```

### app_main.cpp（关键调用）

```cpp
dl::image::jpeg_img_t jpeg_img = {.data=..., .data_len=...};
auto img = dl::image::sw_decode_jpeg(jpeg_img, dl::image::DL_IMAGE_PIX_TYPE_RGB888);

ESPDetDetect *detect = new ESPDetDetect();
auto &results = detect->run(img);
// results 中每个元素: .category(int), .score(float), .box[4](int: x1,y1,x2,y2)
```

## 参考（源文件）

- `train.py`, `val.py`, `espdet_run.py`
- `deploy/export.py`, `deploy/quantize.py`, `deploy/eval_quantized_model.py`
- `nn/esp_tasks.py`, `nn/modules/{esp_conv,esp_block,esp_head}.py`, `nn/modules/__init__.py`
- `data/esp_dataset.py`
- `deploy/espdet_model_template/espdet_detect.{hpp,cpp}`
- `deploy/espdet_example_template/main/app_main.cpp`
