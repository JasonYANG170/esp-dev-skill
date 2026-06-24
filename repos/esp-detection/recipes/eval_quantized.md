# 在 PC 上评估量化后模型

> **适用摘要**: 用 `deploy/eval_quantized_model.py` 在 PC 上对量化后的 ppq 计算图做 mAP 评估，验证 INT8 量化带来的精度损失。

## 触发意图

- "评估量化模型 mAP"
- "PTQ 精度验证"
- "quantized model validation"
- "量化前后精度对比"

## 前置条件

| 条件 | 要求 |
|---|---|
| 量化产物 | 已通过 `quant_espdet()` 得到 ppq graph（或 `.native` 图） |
| 验证集 | `cfg/datasets/*.yaml` 指向的 val 集 |
| 依赖 | esp-ppq、ultralytics、torchvision |

## 分步说明

### 1. 脚本作用

`deploy/eval_quantized_model.py` 提供：
- `CaliDataset`：校准/评估数据加载（与 `deploy/quantize.py` 一致）。
- `quant(imgsz)`：返回 ppq graph（可调 Equalization 参数）。
- `QuantizedModelValidator`：继承 `BaseValidator`，把推理替换为 ppq graph 调用。
- `ppq_graph_init` / `ppq_graph_inference`：初始化 `TorchExecutor` 并做检测后处理。
- `make_quant_validator_class(executor)`：生成可传给 `model.val(validator=...)` 的 validator 类。

### 2. ppq graph 推理与后处理（核心，来自源码）

```python
def ppq_graph_inference(executor, task, inputs, device):
    NC = 1
    graph_outputs = executor(inputs)
    if task == "detect":
        boxes_ls = [graph_outputs[i] for i in range(0, 6, 2)]   # 3 个 box 分支
        bs = inputs.shape[0]
        boxes  = torch.cat([graph_outputs[2*i].view(bs, 4, -1)    for i in range(3)], dim=-1)
        scores = torch.cat([graph_outputs[2*i+1].view(bs, NC, -1) for i in range(3)], dim=-1)
        preds = dict(boxes=boxes, scores=scores, feats=boxes_ls)
        detect_model = Detect(nc=NC, reg_max=1, end2end=False, ch=[32, 64, 128])
        detect_model.stride = [8.0, 16.0, 32.0]
        detect_model.to(device)
        y = detect_model._inference(preds)
        return y
```

要点：
- 量化图输出顺序与导出一致：`box0,score0,box1,score1,box2,score2`。
- `Detect` 用 `reg_max=1`、`end2end=False`、`ch=[32,64,128]`、`stride=[8,16,32]`，与 `ESPDetect` 3 个检测层匹配。
- `end2end=False`（`QuantizedModelValidator` 内显式设置，针对 espdet_pico）。

### 3. 运行评估（来自 `__main__`）

```python
import ultralytics.nn.tasks as tasks
from nn.esp_tasks import custom_parse_model
tasks.parse_model = custom_parse_model
from ultralytics import YOLO
from deploy.eval_quantized_model import ppq_graph_init, make_quant_validator_class, quant

# 加载原 .pt（仅为进入 val 流程并获取数据加载器）
model = YOLO("path/to/best.pt")

# 初始化量化图执行器（case 1: PTQ 量化图；case 2: QAT 用 native_path 加载 .native）
executor = ppq_graph_init(quant, imgsz=224, device="cpu", native_path=None)

QuantDetectionValidator = make_quant_validator_class(executor)
results = model.val(
    data="cfg/datasets/coco_cat.yaml",
    split="val",
    imgsz=224,
    device="cpu",
    validator=QuantDetectionValidator,
    save_json=False,
    save=True,
)
print("quantized mAP@0.5:0.95:", results.box.map)
```

### 4. rect 模型评估

把 `imgsz` 改为 `[h, w]`：

```python
executor = ppq_graph_init(quant, imgsz=[160, 288], device="cpu", native_path=None)
results = model.val(
    data="cfg/datasets/coco_cat.yaml",
    split="val",
    imgsz=[160, 288],
    device="cpu",
    validator=QuantDetectionValidator,
)
```

> `quant(imgsz)` 内部 `INPUT_SHAPE = [3, *imgsz]`，故 rect 时 `imgsz` 必须是 `[h, w]`。

### 5. QAT 图评估（case 2）

QAT 训练时 `.native` 与 `.espdl` 一同保存，评估时用 native 图：

```python
executor = ppq_graph_init(quant, imgsz=224, device="cpu",
                          native_path="path/to/model.native")
```

### 6. 解读精度损失

对比浮点 `val.py` 的 mAP 与量化后 mAP，若下降明显（如 >2~3 mAP@0.5:0.95），参考 `recipes/quantize_espdl.md` 第 5 步调 Equalization 或扩充校准集。

## 常见错误

| 错误 | 原因 | 解决方法 |
|---|---|---|
| `boxes/scores reshape` 维度错 | `graph_outputs` 顺序或 `NC` 不符 | 保持导出 6 输出顺序；`NC` 与模型 `nc` 一致 |
| `Detect` 报 stride 不匹配 | 改了 `ch` 或 `stride` | 固定 `ch=[32,64,128]`、`stride=[8,16,32]`、`reg_max=1` |
| 评估 0 mAP | 用了 `.yaml` 未训练模型作 `model` | `YOLO("best.pt")` 而非 `.yaml` |
| native 图加载失败 | QAT 未生成 `.native` | PTQ 用 `native_path=None`；QAT 确认 `.native` 存在 |
| rect 评估形状错 | `imgsz` 传标量 | rect 传 `imgsz=[h, w]` |
| `task` 非支持 | 传了非 `detect` | 当前仅支持 `detect`（否则 `NotImplementedError`） |

## 参考

- 仓库 `deploy/eval_quantized_model.py`（`quant`、`QuantizedModelValidator`、`ppq_graph_init`、`ppq_graph_inference`、`make_quant_validator_class`）
- `recipes/quantize_espdl.md`（量化调参）
- `recipes/train_square.md` 第 5 步（浮点 mAP 对照）
