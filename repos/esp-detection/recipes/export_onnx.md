# 导出 ONNX（opset 13 + onnxsim）

> **适用摘要**: 用 `deploy/export.py::Export()` 把训练好的 `.pt` 导出为 ONNX（opset 13、onnxsim 简化），输出固定 6 个张量，为后续 esp-ppq 量化做准备。

> Version: selected repo/component version; align with the user project and dependency manifest.
> Evidence: `repos/esp-detection/resources/`, source/examples in `repos/esp-detection/`, and this recipe path `repos/esp-detection/recipes/export_onnx.md`.
> Validation: draft metadata added from repository routing; verify APIs, Kconfig symbols, and component dependencies against the selected version.

## 触发意图

- "导出 ONNX"
- "pt 转 onnx"
- "ESPDetect 导出"
- "opset 13 / onnxsim"

## 前置条件

| 条件 | 要求 |
|---|---|
| 训练产物 | `best.pt`（由 `train.py` 或 `espdet_run.py` 生成） |
| 依赖 | `onnx==1.17.0`、`onnxsim==0.4.36`、`onnxruntime>=1.19.0` |
| 入口 | 在 esp-detection 仓库根运行（能 import `nn.*` / `deploy.*`） |

## 分步说明

### 1. 为什么必须用 Export() 而非原生 ultralytics 导出

`deploy/export.py` 针对 ESP-DL 部署做了三处关键改造（来自源码）：
- **固定 `opset_version=13`**（esp-ppq 默认 18，但 YOLOv11 的 `arange` patch 与 18 不兼容）。
- **用 `onnxsim` 简化**（而非 ultralytics 默认的 onnxslim；onnxslim 会把 YOLOv11 的 `NCHW` 改成 `1(N*C)HW`，破坏后处理）。
- **绑定 `ESPDetect.export_onnx_forward` 与 `ESP_Attention.forward`**，使 ONNX 输出为 6 个分离张量 `box0,score0,box1,score1,box2,score2`（3 个检测尺度，每个尺度一对 box+score），与量化后处理期望严格对应。

### 2. 调用 Export

```python
import ultralytics.nn.tasks as tasks
from nn.esp_tasks import custom_parse_model
from deploy.export import Export

tasks.parse_model = custom_parse_model

# 方形
Export("path/to/best.pt", 224)          # 生成 best.onnx，输入 1x3x224x224

# 非方形（rect 训练后的模型）
Export("path/to/best.pt", [160, 288])   # 输入 1x3x160x288
```

`Export(model_path, input_size)` 内部（来自源码）：
```python
tasks.parse_model = custom_parse_model
model = ESP_YOLO(model_path)
for m in model.modules():
    if isinstance(m, Attention):
        m.forward = ESP_Attention.forward.__get__(m)
    if isinstance(m, ESPDetect):
        m.forward = ESPDetect.export_onnx_forward.__get__(m)
model.export(format="onnx", simplify=True, opset=13, imgsz=input_size)
```

### 3. 输出张量语义

`export_onnx_forward` 显式返回 6 个张量���来自 `nn/modules/esp_head.py`）：

```python
def export_onnx_forward(self, x):
    box0, score0 = self.cv2[0](x[0]), self.cv3[0](x[0])   # P3/8 尺度
    box1, score1 = self.cv2[1](x[1]), self.cv3[1](x[1])   # P4/16 尺度
    box2, score2 = self.cv2[2](x[2]), self.cv3[2](x[2])   # P5/32 尺度
    return box0, score0, box1, score1, box2, score2
```

ONNX `output_names`（来自 `export.py`）：
```python
output_names = ["box0", "score0", "box1", "score1", "box2", "score2"]
```

### 4. 动态 batch 仅用于 QAT，部署保持静态

```python
# 部署模型：dynamic=False（Export 默认 self.args.dynamic，部署时为 False）
# 输入固定 batch=1，形状 [1, 3, H, W]

# 仅 QAT gt onnx 推理才用：
dynamic = {"images": {0: "batch"}}
for name in output_names:
    dynamic[name] = {0: "batch"}
```

### 5. 验证 ONNX

```python
import onnx
m = onnx.load("best.onnx")
onnx.checker.check_model(m)
print("inputs :", [(i.name, [d.dim_value for d in i.type.tensor_type.shape.dim]) for i in m.graph.input])
print("outputs:", [o.name for o in m.graph.output])
# 预期输出: ['box0','score0','box1','score1','box2','score2']
```

## 常见错误

| 错误 | 原因 | 解决方法 |
|---|---|---|
| `KeyError: 'ESPDetect'` | 未注入 `custom_parse_model` | 入口加 `tasks.parse_model = custom_parse_model` |
| ONNX 输出只有一个张量 | 用了原生 `model.export()`，未绑定 export_onnx_forward | 必须用 `deploy/export.py::Export()` |
| onnxslim 把 NCHW 改成 1(N*C)HW | 用了 ultralytics 默认简化 | 用 `Export()`（内部用 onnxsim），不要手动开 onnxslim |
| opset 不兼容 | 手动设了 opset 18 | 固定 `opset=13`（Export 内部已固定） |
| `do_constant_folding` 警告 | torch≥1.12 DNN 推理 | 部署无影响；如需可设 `do_constant_folding=False` |
| 形状与量化不一致 | Export 的 `input_size` 与训练 imgsz 不同 | 两者保持同一 `[h, w]` |

## 参考

- 仓库 `deploy/export.py`（`Export`、`ESP_Detect_Exporter.export_onnx`、`ESP_YOLO`、`ESP_Attention`）
- 仓库 `nn/modules/esp_head.py`（`ESPDetect.export_onnx_forward`）
- `recipes/quantize_espdl.md`（导出后量化）
- `docs/tutorials/how_to_train_and_deploy_model_with_rect_is_True.md`（Export 章节��
