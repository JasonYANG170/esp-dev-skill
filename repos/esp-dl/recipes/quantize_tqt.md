# 用 TQT（Trained Quantization Thresholds）训练量化阈值

> **适用摘要**: 当默认 8-bit PTQ 精度不足时，启用 ESP-PPQ 的 TQT：在 log 域优化 `scale = 2^k` 并联合微调权重（**无需标签**，loss 是浮点输出与量化输出的 MSE），导出仍满足 ESP-DL 的 Power-of-2 约束。介于 PTQ 与 QAT 之间，是精度不足时最直接的下一步。

> Version: selected repo/component version; align with the user project and dependency manifest.
> Evidence: `repos/esp-dl/resources/`, source/examples in `repos/esp-dl/`, and this recipe path `repos/esp-dl/recipes/quantize_tqt.md`.
> Validation: draft metadata added from repository routing; verify APIs, Kconfig symbols, and component dependencies against the selected version.

## 触发意图

- "PTQ 精度不够，又不想做 QAT"
- "训练量化阈值"
- "TQT 量化"
- "learnable scale 量化"
- "scale = 2^k 微调"

## 前置条件

| 条件 | 要求 |
|---|---|
| ESP-PPQ | `pip install esp-ppq` **>= 1.2.7** |
| 校准数据 | 一批代表性输入（**无需标签**），与 `quantize_onnx.md` 相同 |
| 参考文档 | `docs/en/tutorials/quantize_model_with_TQT.rst` |
| 参考示例 | `examples/tutorial/how_to_quantize_model/quantize_yolo26`、`.../quantize_mobilenetv2` |

## 分步说明

### 1. 为什么用 TQT（而非 PTQ / QAT）

| 方法 | 是否需训练 | 是否需标签 | scale 来源 | 适用场景 |
|---|---|---|---|---|
| PTQ | 否 | 否 | 统计（可能次优） | 首选，速度快 |
| **TQT** | **是（少量步）** | **否**（MSE loss） | **反向传播学习 `log2(scale)`** | PTQ 精度不足、又不想做 QAT |
| QAT | 是（长训练） | 是 | fake-quant 联合训练 | 精度要求最高 |

ESP32 系列芯片只支持 **Per-Tensor + Symmetric + Power-of-2** 量化；TQT 在该约束下优化 `alpha = log2(scale)`，导出的 `.espdl` 里 exponent 来自 `int(log2(scale))`，与芯片侧一致。

### 2. 启用 TQT（最小改动）

在 `espdl_quantize_onnx` 用的 `quant_setting` 上打开 TQT 即可，其余参数与普通 PTQ 一致：

```python
from esp_ppq import QuantizationSettingFactory
from esp_ppq.api import espdl_quantize_onnx

quant_setting = QuantizationSettingFactory.espdl_setting()
quant_setting.tqt_optimization = True
quant_setting.tqt_optimization_setting.steps = 500
quant_setting.tqt_optimization_setting.lr = 1e-5
quant_setting.tqt_optimization_setting.block_size = 4
quant_setting.tqt_optimization_setting.collecting_device = "cpu"   # 有 GPU 设 "cuda"
# 可选：让学到的 alpha 更靠近整数，便于导出
quant_setting.tqt_optimization_setting.int_lambda = 0.25
```

### 3. `TQTSetting` 参数表

| 参数 | 类型 | 默认 | 说明 |
|---|---|---|---|
| `interested_layers` | `List[str]` | `[]` | 指定微调的算子；为空则对所有可训 Conv/Gemm 生效 |
| `steps` | `int` | `500` | 每个 block 的微调步数 |
| `lr` | `float` | `1e-5` | 学习率（YOLO26n 用 `1e-5`，MobileNetV2 用 `1e-4`） |
| `block_size` | `int` | `4` | 图分块的 block 大小；影响稳定性与速度（先试 4，不稳降到 2） |
| `is_scale_trainable` | `bool` | `True` | 是否训练 scale；`False` 则只微调权重 |
| `gamma` | `float` | `0.0` | 正则：MSE(浮点输出, 量化输出) |
| `int_lambda` | `float` | `0.0` | 正则：把 `alpha` 拉向 `round(alpha)`，范围 `[0.0, 1.0]` |
| `collecting_device` | `str` | `"cpu"` | 校准数据缓存设备；有 GPU 设 `"cuda"` 加速 |

### 4. 完整量化脚本（带 GPU 加速）

```python
from esp_ppq import QuantizationSettingFactory
from esp_ppq.api import espdl_quantize_onnx, ENABLE_CUDA_KERNEL

ONNX_PATH  = "model.onnx"
ESPDL_PATH = "model_tqt.espdl"
INPUT_SHAPE = [3, 224, 224]      # 不含 batch
TARGET      = "esp32p4"          # 'c' / 'esp32s3' / 'esp32p4'
NUM_OF_BITS = 8
DEVICE      = "cuda"             # 或 "cpu"

quant_setting = QuantizationSettingFactory.espdl_setting()
quant_setting.tqt_optimization = True
quant_setting.tqt_optimization_setting.collecting_device = "cuda"
quant_setting.tqt_optimization_setting.steps = 500
quant_setting.tqt_optimization_setting.block_size = 4
quant_setting.tqt_optimization_setting.lr = 1e-4

with ENABLE_CUDA_KERNEL():       # GPU 加速内核；CPU 模式去掉此 with
    quant_ppq_graph = espdl_quantize_onnx(
        onnx_import_file=ONNX_PATH,
        espdl_export_file=ESPDL_PATH,
        calib_dataloader=dataloader,
        calib_steps=32,
        input_shape=[1] + INPUT_SHAPE,
        target=TARGET,
        num_of_bits=NUM_OF_BITS,
        collate_fn=collate_fn,
        setting=quant_setting,
        device=DEVICE,
        error_report=True,
        skip_export=False,
        export_test_values=False,
        verbose=1,
    )
```

### 5.（可选）配合聚焦 logit 量化（检测模型）

YOLO26n 这类检测模型，喂给 Sigmoid 的 head logit 对量化极敏感。`quantize_yolo26` 示例在 TQT 之外额外给三个 head 卷积分配置更细的 scale（`0.0625 = 2^-4`），扩大 Sigmoid 有效区间内的量化层数：

```python
quant_setting.quantize_activation_setting.calib_algorithm = "percentile"  # kl -> percentile 提升 mAP
quant_setting.quant_config_modify = True
quant_setting.quant_config_modify_setting.custom_config = {
    "/model.23/one2one_cv3.0/one2one_cv3.0.2/Conv": 0.0625,
    "/model.23/one2one_cv3.1/one2one_cv3.1.2/Conv": 0.0625,
    "/model.23/one2one_cv3.2/one2one_cv3.2.2/Conv": 0.0625,
}
# 然后再开 TQT（见步骤 2）
```

> 非 end2end（one2many）模型把路径里的 `one2one_cv3.*` 换成 `cv3.*` 即可。

### 6. 实测精度提升

| 模型 | 配置 | 指标 | 结果 |
|---|---|---|---|
| YOLO26n (640, o2m) | PTQ 无 TQT | mAP50-95 | 0.342 |
| YOLO26n (640, o2m) | **PTQ + TQT** | mAP50-95 | **0.371**（+2.9） |
| YOLO26n (640, e2e) | PTQ 无 TQT | mAP50-95 | 0.332 |
| YOLO26n (640, e2e) | **PTQ + TQT** | mAP50-95 | **0.363**（+3.1） |
| YOLO26n (512, e2e) | PTQ 无 TQT | mAP50-95 | 0.315 |
| YOLO26n (512, e2e) | PTQ + TQT | mAP50-95 | 0.341（+2.6） |
| MobileNetV2 | per-channel PTQ（P4） | Top-1 | 70.850% |
| MobileNetV2 | **TQT**（8-bit） | Top-1 | **71.775%**（+1.3，逼近 float 71.878%） |

### 7. FAQ 速查

- **加速**：有 GPU 时设 `collecting_device="cuda"` 并包在 `with ENABLE_CUDA_KERNEL():` 里；`block_size` 调大到 2~4 减少分块数（过大不稳）。
- **只微调权重不动 scale**：`is_scale_trainable = False`。
- **与逐层均衡联用**：可以。`tqt_optimization_setting.equalization = True` + `tqt_optimization = True`（如 `espdet_pico` 这类模型组合后更好）。

## 常见错误

| 错误 | 原因 | 解决方法 |
|---|---|---|
| `tqt_optimization_setting` 不存在 | ESP-PPQ < 1.2.7 | 升级 `pip install -U esp-ppq` |
| TQT 很慢 | CPU + 校准数据在 CPU | 设 `collecting_device="cuda"` + `ENABLE_CUDA_KERNEL` |
| 训练发散/NaN | `lr` 过大或 `block_size` 过大 | 降 `lr`（如 `1e-5`）；`block_size` 降到 2 |
| 导出 scale 不是整数次幂 | 未做整数化 | 设 `int_lambda`（如 0.25）拉向整数 |
| 检测 mAP 没提升 | head logit 量化过粗 | 加聚焦 logit 量化（步骤 5） |
| 想完全不动 scale | 默认会训 scale | `is_scale_trainable = False` |

## 参考

- `docs/en/tutorials/quantize_model_with_TQT.rst`
- `examples/tutorial/how_to_quantize_model/quantize_yolo26/`（YOLO26n TQT + 聚焦 logit）
- `examples/tutorial/how_to_quantize_model/quantize_mobilenetv2/`（MobileNetV2 TQT）
- 论文：*Trained Quantization Thresholds for Accurate and Efficient Fixed-Point Inference of Deep Neural Networks*（MLSys 2020，arXiv:1903.08066）
