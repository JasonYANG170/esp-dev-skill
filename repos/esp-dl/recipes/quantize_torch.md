# 用 ESP-PPQ 把 PyTorch 模型量化为 .espdl

> **适用摘要**: 使用 `espdl_quantize_torch` 直接对 `torch.nn.Module` 做量化并导出 `.espdl`，无需先导 ONNX。支持普通模型和流式模型（`auto_streaming`）。

> Version: selected repo/component version; align with the user project and dependency manifest.
> Evidence: `repos/esp-dl/resources/`, source/examples in `repos/esp-dl/`, and this recipe path `repos/esp-dl/recipes/quantize_torch.md`.
> Validation: draft metadata added from repository routing; verify APIs, Kconfig symbols, and component dependencies against the selected version.

## 触发意图

- "量化 pytorch 模型"
- "espdl_quantize_torch 用法"
- "直接从 torch 导出 espdl"

## 前置条件

| 条件 | 要求 |
|---|---|
| ESP-PPQ | `pip install esp-ppq` |
| 参考示例 | `examples/tutorial/how_to_quantize_model/quantize_sin_model/quantize_torch_model.py`、`examples/tutorial/how_to_deploy_streaming_model/quantize_streaming_model/quantize_torch_model.py` |

## 分步说明

### 1. 量化脚本（PyTorch）

```python
from torch.utils.data import DataLoader, TensorDataset
from esp_ppq.api import espdl_quantize_torch

ESPDL_MODEL_PATH = "model.espdl"
INPUT_SHAPE = [1, 1]          # 不含 batch；batch 固定为 1
TARGET      = "esp32s3"       # 'c' / 'esp32s3' / 'esp32p4'
NUM_OF_BITS = 8
DEVICE      = "cpu"

dataloader = DataLoader(dataset, batch_size=32, shuffle=False)   # shuffle 必须 False

quant_ppq_graph = espdl_quantize_torch(
    model=model,                        # torch.nn.Module 实例
    espdl_export_file=ESPDL_MODEL_PATH,
    calib_dataloader=dataloader,
    calib_steps=32,
    input_shape=INPUT_SHAPE,
    inputs=None,
    target=TARGET,
    num_of_bits=NUM_OF_BITS,
    dispatching_override=None,
    device=DEVICE,
    error_report=True,
    skip_export=False,
    export_test_values=True,
    verbose=1,
)
```

### 2. 导出流式模型（自动插入 StreamingCache）

```python
quant_ppq_graph = espdl_quantize_torch(
    model=model,
    espdl_export_file="model_streaming.espdl",
    calib_dataloader=dataloader,
    calib_steps=32,
    input_shape=INPUT_SHAPE,            # 离线完整输入形状
    target=TARGET,
    num_of_bits=NUM_OF_BITS,
    device=DEVICE,
    error_report=True,
    skip_export=False,
    export_test_values=False,
    verbose=1,
    auto_streaming=True,                # 开启自动流式转换
    streaming_input_shape=[1, 16, 3],   # 每个 chunk 的输入形状
    streaming_table=None,               # 或手动 cache 表（见 streaming_model 食谱）
)
```

### 3. 在 PC 上评估

```python
from esp_ppq.api import TorchExecutor
executor = TorchExecutor(graph=quant_ppq_graph, device=DEVICE)
output = executor(input)
```

## 常见错误

| 错误 | 原因 | 解决方法 |
|---|---|---|
| 模型无法被追踪 | 含动态控制流 | 改写为可追踪结构，或先导 ONNX 再用 `espdl_quantize_onnx` |
| 校准误差错乱 | `shuffle=True` | 设 `shuffle=False` |
| 流式输出与离线不符 | window/cache 配置错 | 用 `streaming_table` 手动指定 cache，核对 `streaming_input_shape` |
| 多 batch 报错 | ESP-DL 仅 batch=1 | 固定 batch=1 |

## 参考

- `examples/tutorial/how_to_quantize_model/quantize_sin_model/quantize_torch_model.py`
- `examples/tutorial/how_to_deploy_streaming_model/quantize_streaming_model/quantize_torch_model.py`
- `docs/en/tutorials/how_to_quantize_model.rst`
- `docs/en/tutorials/how_to_deploy_streaming_model.rst`
