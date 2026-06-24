# 用 ESP-PPQ 把 ONNX 模型量化为 .espdl

> **适用摘要**: 使用 `espdl_quantize_onnx` 把 ONNX 模型做训练后量化（PTQ），导出可在 ESP 芯片部署的 `.espdl` 模型。PyTorch/TensorFlow/Paddle 需先转 ONNX。

## 触发意图

- "量化 onnx 模型"
- "导出 espdl"
- "espdl_quantize_onnx 怎么用"
- "PTQ 量化"

## 前置条件

| 条件 | 要求 |
|---|---|
| ESP-PPQ | `pip install esp-ppq`（先装 CPU 版 torch） |
| 算子支持 | 模型算子都在 `operator_support_state.md` 内（64 个算子，推荐 opset 18） |
| 参考示例 | `examples/tutorial/how_to_quantize_model/quantize_sin_model`、`examples/tutorial/how_to_quantize_model/quantize_mobilenetv2` |

## 分步说明

### 1. 安装 ESP-PPQ

```bash
pip install torch torchvision torchaudio --index-url https://download.pytorch.org/whl/cpu
pip install esp-ppq
```

### 2. 选择 `target`（决定量化策略与舍入模式）

| target | 芯片 | 策略 | 舍入 |
|---|---|---|---|
| `c` | ESP32 / C2 / C3 / C5 / C6 / S2 | per-tensor，C 实现 | round-half-up |
| `esp32s3` | ESP32-S3 | per-tensor，PIE V1 | round-half-up |
| `esp32p4` | ESP32-P4 | Conv/Gemm per-channel，其余 per-tensor，PIE V2 | round-half-even |

> 不同平台的 `.espdl` 不能混用，否则推理结果不准。

### 3. 量化脚本（ONNX）

```python
from torch.utils.data import DataLoader, TensorDataset
from esp_ppq.api import espdl_quantize_onnx

ONNX_MODEL_PATH  = "model.onnx"
ESPDL_MODEL_PATH = "model.espdl"
INPUT_SHAPE = [1, 1]          # 不含 batch；batch 固定为 1
TARGET      = "esp32s3"       # 'c' / 'esp32s3' / 'esp32p4'
NUM_OF_BITS = 8               # 8 或 16
DEVICE      = "cpu"           # 'cpu' 或 'cuda'

# 校准数据（dataloader 的 shuffle 必须为 False：多次遍历要求数据顺序稳定）
dataset = TensorDataset(x_calib)        # 只需要输入 x，不需要 label
dataloader = DataLoader(dataset, batch_size=32, shuffle=False)

def collate_fn(batch):
    return batch[0].to(DEVICE)

quant_ppq_graph = espdl_quantize_onnx(
    onnx_import_file=ONNX_MODEL_PATH,
    espdl_export_file=ESPDL_MODEL_PATH,
    calib_dataloader=dataloader,
    calib_steps=32,                     # 校准步数
    input_shape=INPUT_SHAPE,            # batch 为 1
    inputs=None,                        # None 表示用随机测试输入；或传具体输入
    target=TARGET,
    num_of_bits=NUM_OF_BITS,
    collate_fn=collate_fn,
    dispatching_override=None,
    device=DEVICE,
    error_report=True,                  # 打印量化误差报告
    skip_export=False,
    export_test_values=True,            # 想在板端用 model->test() 就要开
    verbose=1,
)
```

### 4. 导出产物

- `model.espdl`：部署二进制（可用 [netron](https://netron.app) 可视化）
- `model.info`：文本调试信息（模型结构、量化权重、测试输入/输出，均 16 字节对齐）
- `model.json`：量化信息文件，可保存/加载

### 5.（可选）在 PC 上评估量化精度

```python
from esp_ppq.api import TorchExecutor
executor = TorchExecutor(graph=quant_ppq_graph, device=DEVICE)
output = executor(input)
```

板端 `esp-dl` 结果与 `esp-ppq` 对齐，PC 侧指标可直接评估板端精度。

## 常见错误

| 错误 | 原因 | 解决方法 |
|---|---|---|
| 算子不支持 | 不在 64 个支持算子内 | 查 `operator_support_state.md`；提 issue 或用 operator skill 实现 |
| 校准误差统计错乱 | dataloader `shuffle=True` | 必须设 `shuffle=False` |
| 板端结果与 PC 不一致 | target 与芯片不符 | `target` 必须等于部署芯片类别 |
| `test()` 失败 | 未导出测试值 | 设 `export_test_values=True` |
| 动态 batch 报错 | ESP-DL 只支持 batch=1 | `input_shape` 的 batch 固定为 1 |

## 参考

- `examples/tutorial/how_to_quantize_model/quantize_sin_model/quantize_onnx_model.py`
- `examples/tutorial/how_to_quantize_model/quantize_mobilenetv2/quantize_onnx_model.py`
- `docs/en/tutorials/how_to_quantize_model.rst`
- `docs/en/getting_started/readme.rst`（ESP-PPQ 安装）
