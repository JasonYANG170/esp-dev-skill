# 高级 PTQ：混合精度（坏层 int16）与逐层权重均衡

> **适用摘要**: 当默认 8-bit per-tensor PTQ（尤其 ESP32-S3）掉点严重，又不想上 TQT/QAT 时，用两种**确定性、可定向**的 PTQ 技巧：① 把量化误差最大的几层派发到 int16（混合精度）；② 对带 ReLU/ReLU6 的模型做逐层权重均衡（layerwise equalization）。两者都不需要标签、不需要训练。

## 触发意图

- "S3 上 PTQ 精度太低（~60%）"
- "混合精度量化"
- "把某些层量化成 int16"
- "layerwise equalization / 权重均衡"
- "MobileNetV2 S3 精度提升"
- "不想做 QAT 但要更高精度"

## 前置条件

| 条件 | 要求 |
|---|---|
| ESP-PPQ | `pip install esp-ppq` |
| 参考文档 | `docs/en/tutorials/how_to_deploy_mobilenetv2.rst`（Mixed precision / Layerwise equalization 两节） |
| 参考脚本 | `examples/tutorial/how_to_quantize_model/quantize_mobilenetv2/quantize_onnx_model.py`（`quant_setting_mobilenet_v2`） |

## 分步说明

### 1. 先定位最坏的层（误差报告）

跑默认 PTQ 并看 `error_report=True` 输出的 **Layerwise error**（单层量化误差，而非累积）。MobileNetV2 在 S3 上典型的坏层是 `/features/features.1/conv/conv.0/conv.0.0/Conv`（layerwise 14.3%）和它后面的 `Clip`（ReLU6）。大多数层 < 1%，**只需把少数坏层单独处理**。

```python
quant_setting = QuantizationSettingFactory.espdl_setting()
# 默认 PTQ，先跑出误差报告
quant_ppq_graph = espdl_quantize_onnx(..., setting=quant_setting, error_report=True, ...)
```

### 2. 混合精度：把坏层派发到 int16

用 `dispatching_table.append(layer_name, get_target_platform(target, 16))` 把指定层量化为 16-bit，其余仍 8-bit。`get_target_platform` 返回该 target 下指定位宽的平台标识：

```python
from esp_ppq.api import espdl_quantize_onnx, get_target_platform
from esp_ppq import QuantizationSettingFactory

TARGET = "esp32s3"
quant_setting = QuantizationSettingFactory.espdl_setting()

# 把误差最大的层（及其后 ReLU6/Clip）派发到 int16
quant_setting.dispatching_table.append(
    "/features/features.1/conv/conv.0/conv.0.0/Conv",
    get_target_platform(TARGET, 16),
)
quant_setting.dispatching_table.append(
    "/features/features.1/conv/conv.0/conv.0.2/Clip",   # ReLU6 在 ONNX 里是 Clip
    get_target_platform(TARGET, 16),
)
# 可继续 append 其它坏层
```

> 层名必须与 ONNX 图里的算子名完全一致（看 `error_report` 或 netron）。`Clip` 是 ONNX 对 ReLU6 的表示。

### 3. 逐层权重均衡（Layerwise Equalization）

原理见论文 *Data-Free Quantization Through Weight Equalization and Bias Correction*（ICCV 2019）：利用相邻 Conv 之间 ReLU 的正齐次性，把权重幅度在层间重新分配，让各层量化范围更均衡。**要求激活是 ReLU**（ReLU6 不满足条件，需先替换）。

```python
import torch.nn as nn

def convert_relu6_to_relu(model):
    for child_name, child in model.named_children():
        if isinstance(child, nn.ReLU6):
            setattr(model, child_name, nn.ReLU())
        else:
            convert_relu6_to_relu(child)
    return model

model = convert_relu6_to_relu(model)   # 仅在均衡时需要；均衡对 ONNX 用同名 _relu.onnx

quant_setting = QuantizationSettingFactory.espdl_setting()
quant_setting.equalization = True
quant_setting.equalization_setting.iterations = 6        # 均衡迭代次数（示例用 4~6）
quant_setting.equalization_setting.value_threshold = 0.5 # 触发均衡的阈值
quant_setting.equalization_setting.opt_level = 2         # 优化级别
quant_setting.equalization_setting.interested_layers = None  # None=全部；也可指定子集
```

### 4. MobileNetV2 / S3 实测精度对比

来自 `how_to_deploy_mobilenetv2.rst`（float 基线 Top-1 **71.878%**）：

| 策略 | target | Top-1 | Top-5 | 末层累积误差 |
|---|---|---|---|---|
| 8-bit 默认 PTQ | `esp32s3` | 60.325% | 83.100% | 27.097%（偏大） |
| **+ 混合精度**（坏层 int16） | `esp32s3` | **69.225%** | 88.700% | 10.289% |
| **+ 逐层均衡**（ReLU6→ReLU） | `esp32s3` | **69.800%** | 88.400% | 10.808% |
| 8-bit per-channel（参照） | `esp32p4` | 71.150% | 89.350% | — |

> 经验：末层累积误差降到 **10% 以下**时，量化带来的精度损失通常可接受。两种技巧把 S3 的 Top-1 从 60.5% 拉回 ~69.8%，接近 P4 per-channel。

### 5. 两种技巧的选择

| 情况 | 推荐 |
|---|---|
| 只有少数层 layerwise 误差大 | **混合精度**（最小改动，定向把坏层 int16） |
| 模型含 ReLU/ReLU6，整体 per-tensor 掉点多 | **逐层均衡**（全图重分配权重幅度） |
| 两者都用 | 可叠加：先均衡再混合精度；若仍不够上 TQT（见 `quantize_tqt.md`） |
| P4（per-channel） | 通常已接近 float，**无需**这两种技巧 |

### 6. 与 AutoQuant 的关系

`auto_quant.md` 的 AutoQuant 会在多种策略间自动搜索（含 mixed_precision）。当你**已经知道是哪几层坏**、想要确定性结果快速复现时，手动混合精度/均衡比跑一遍 AutoQuant 搜索更直接。不确定时用 AutoQuant 探索，确定后用手动配置固化。

## 常见错误

| 错误 | 原因 | 解决方法 |
|---|---|---|
| 派发 int16 后精度没变 | 层名写错或拼错 | 层名逐字符对照 `error_report` / netron；注意 ONNX 里 ReLU6 是 `Clip` |
| 均衡后精度反而降 | 模型含 ReLU6 未替换 | 先 `convert_relu6_to_relu` 再均衡 |
| 均衡报错“无可均衡层” | 相邻层间无 ReLU | 均衡只作用于 Conv–ReLU–Conv 结构；该模型可能不适合 |
| `equalization_setting` 不存在 | ESP-PPQ 版本旧 | 升级 `pip install -U esp-ppq` |
| 选错 target 平台 | `get_target_platform` 用了错的 target | 必须与 `espdl_quantize_onnx` 的 `target` 一致 |
| S3 上仍想再提升 | 两种 PTQ 技巧到顶（~69.8%） | 上 TQT（`quantize_tqt.md`）可达 71.775% |

## 参考

- `docs/en/tutorials/how_to_deploy_mobilenetv2.rst`（Mixed precision / Layerwise equalization 章节）
- `examples/tutorial/how_to_quantize_model/quantize_mobilenetv2/quantize_onnx_model.py`（`quant_setting_mobilenet_v2` 辅助函数）
- 论文：*Data-Free Quantization Through Weight Equalization and Bias Correction*（ICCV 2019）
- 相关 recipe：`recipes/quantize_onnx.md`（默认 PTQ）、`recipes/auto_quant.md`（自动搜索）、`recipes/quantize_tqt.md`（TQT）
