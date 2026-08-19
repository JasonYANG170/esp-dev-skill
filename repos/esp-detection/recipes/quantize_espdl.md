# 量化为 INT8 .espdl（esp-ppq PTQ）

> **适用摘要**: 用 `deploy/quantize.py::quant_espdet()` 基于 esp-ppq 把 ONNX 后训练量化（PTQ）为 INT8 ESP-DL `.espdl`，需提供校准数据集。

> Version: selected repo/component version; align with the user project and dependency manifest.
> Evidence: `repos/esp-detection/resources/`, source/examples in `repos/esp-detection/`, and this recipe path `repos/esp-detection/recipes/quantize_espdl.md`.
> Validation: draft metadata added from repository routing; verify APIs, Kconfig symbols, and component dependencies against the selected version.

## 触发意图

- "量化模型"
- "ONNX 转 espdl"
- "esp-ppq PTQ"
- "INT8 量化 / calibration"

## 前置条件

| 条件 | 要求 |
|---|---|
| 输入 | `best.onnx`（由 `deploy/export.py::Export()` 生成，opset 13） |
| 校准集 | `calib_dir` 目录含 `.jpg/.jpeg/.png/.bmp` |
| 依赖 | `esp-ppq`（git 安装）、`onnxsim==0.4.36` |
| 教程 | `docs/tutorials/how_to_train_and_deploy_model_with_rect_is_True.md`（Quantization 章节） |

## 分步说明

### 1. 调用 quant_espdet

来自 `deploy/quantize.py`：

```python
from deploy.quantize import quant_espdet

quant_espdet(
    onnx_path="espdet_pico_224_224_cat.onnx",
    target="esp32p4",          # 或 "esp32s3"，必须与最终部署芯片一致
    num_of_bits=8,
    device='cpu',
    batchsz=32,
    imgsz=224,                 # 方形标量；rect 模型用 [h, w]，如 [160, 288]
    calib_dir="deploy/cat_calib",
    espdl_model_path="espdet_pico_224_224_cat.espdl",
)
```

> `imgsz` 为标量时 `INPUT_SHAPE=[3, imgsz, imgsz]`；为 `[h,w]` 时 `[3, h, w]`（来自源码 `[3, *imgsz]`）。

### 2. 内部流程（来自源码）

`quant_espdet()` 依次：
1. `onnx.load` → `onnxsim.simplify` → `onnx.shape_inference.infer_shapes` 回写。
2. 构造 `CaliDataset(calib_dir, img_shape=imgsz)`：`ToTensor` + `Resize((h,w))` + `Normalize(mean=[0,0,0], std=[1,1,1])`（**不**做 ImageNet 归一化）。
3. `DataLoader(batch_size=batchsz, shuffle=False)`。
4. `QuantizationSettingFactory.espdl_setting()`，并开启 Equalization（默认 `iterations=4`、`value_threshold=0.4`、`opt_level=2`）。
5. `espdl_quantize_onnx(...)` 完成 PTQ 并写出 `.espdl`，`calib_steps=32`、`input_shape=[1]+INPUT_SHAPE`。

### 3. 校准数据集要求

`CaliDataset.__init__`（来自 `deploy/quantize.py`）：

```python
class CaliDataset(Dataset):
    def __init__(self, path, img_shape=640):
        height, width = img_shape if isinstance(img_shape, (list, tuple)) else (img_shape, img_shape)
        self.transform = transforms.Compose([
            transforms.ToTensor(),
            transforms.Resize((height, width)),
            transforms.Normalize(mean=[0, 0, 0], std=[1, 1, 1]),
        ])
        # 遍历 path 下 .jpg/.jpeg/.png/.bmp
```

要点：
- 校准图应**与训练/部署分布一致**（同类别、同场景），数量几十到几百张即可。
- 仓库自带的 `deploy/cat_calib/000000574810.jpg` 仅是占位样例，真实使用需补充。

### 4. rect 模型量化

非方形分辨率传 `[h, w]`：

```python
quant_espdet(
    onnx_path="espdet_pico_160_288_cat.onnx",
    target="esp32s3",
    num_of_bits=8,
    device='cpu',
    batchsz=32,
    imgsz=[160, 288],
    calib_dir="deploy/cat_calib",
    espdl_model_path="espdet_pico_160_288_cat.espdl",
)
```

### 5. 调优量化精度（可选）

来自 `deploy/eval_quantized_model.py::quant()` 的对照示例，可调 Equalization 参数：

```python
quant_setting = QuantizationSettingFactory.espdl_setting()
quant_setting.equalization = True
quant_setting.equalization_setting.iterations = 10       # 默认 4，精度不足可加大
quant_setting.equalization_setting.value_threshold = 0.3 # 默认 0.4
quant_setting.equalization_setting.opt_level = 2
quant_setting.equalization_setting.interested_layers = None
```

> `deploy/quantize.py::quant_espdet` 内部用的是 `iterations=4 / value_threshold=0.4`；`deploy/eval_quantized_model.py::quant` 示例用 `10 / 0.3`。按精度需求二选一或自行调参。

### 6. 评估量化精度（可选）

量化后用 `recipes/eval_quantized.md` 在 PC 上验证 mAP，确认精度损失在可接受范围。

## 常见错误

| 错误 | 原因 | 解决方法 |
|---|---|---|
| 量化后 mAP 大幅下降 | 校准集分布偏差 / Equalization 不够 | 补充校准图；加大 `equalization_setting.iterations` |
| `target` 与芯片不符 | 量化 P4 部署 S3（或反之） | `target` 与最终部署芯片严格一致 |
| `Simplified ONNX model could not be validated` | onnx/onnxsim 版本不匹配 | 用 requirements 锁定的 `onnx==1.17.0`、`onnxsim==0.4.36` |
| 归一化导致精度异常 | 自定义校准集加了 ImageNet 均值方差 | 用仓库默认 `mean=0, std=1` |
| rect 模型量化形状错 | `imgsz` 传了标量 | rect 模型传 `imgsz=[h, w]` |
| 校准集为空 | 目录无图片或后缀不支持 | 确保含 `.jpg/.jpeg/.png/.bmp` |

## 参考

- 仓库 `deploy/quantize.py`（`quant_espdet`、`CaliDataset`）
- 仓库 `deploy/eval_quantized_model.py`（`quant` 的 Equalization 调参对照）
- 仓库 `deploy/cat_calib/`（校准图样例）
- `docs/tutorials/how_to_train_and_deploy_model_with_rect_is_True.md`（Quantization 章节）
- `recipes/export_onnx.md`、`recipes/eval_quantized.md`
