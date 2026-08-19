# 方形分辨率训练 espdet_pico

> **适用摘要**: 用 `train.py::Train()` 在方形输入分辨率（如 224×224、416×416）下从零训练 espdet_pico 检测模型，并验证 mAP。

> Version: selected repo/component version; align with the user project and dependency manifest.
> Evidence: `repos/esp-detection/resources/`, source/examples in `repos/esp-detection/`, and this recipe path `repos/esp-detection/recipes/train_square.md`.

## 触发意图

- "训练 espdet_pico"
- "方形输入训练"
- "224 训练 / 416 训练"
- "从头训练（不用预训练权重）"

## 前置条件

| 条件 | 要求 |
|---|---|
| 环境 | 已按 `recipes/env_setup.md` 搭建 |
| 数据 | 已按 `recipes/dataset_prepare.md` 准备 `cfg/datasets/*.yaml` |
| 设备 | `cpu` / `mps` / 单 GPU `0` / 多 GPU `[0,1,2,3]` 均可 |

## 分步说明

### 1. 注入自定义模块解析

任何训练入口的第一步（来自 `train.py`）：

```python
import ultralytics.nn.tasks as tasks
from nn.esp_tasks import custom_parse_model
tasks.parse_model = custom_parse_model   # 注册 DSConv/ESPBlock/ESPDetect 等
```

### 2. 调用 Train（从零训练）

`train.py::Train()` 的默认训练设置（来自仓库）：

```python
train_setting = dict(
    data=dataset,
    epochs=1200,          # 设为合理值
    imgsz=imgsz,          # 方形：标量或 [s, s]
    batch=128,
    device="cpu",         # "0" 单卡, [0,1,2,3] 多卡, "mps"
    optimizer='auto',
    close_mosaic=30,
    mosaic=1.0,
    mixup=0.0,
    copy_paste=0.1,
    rect=False,           # 方形训练保持 False
)
```

从零训练（`pretrained_path=None` 时从 `cfg/models/espdet_pico.yaml` 构建新模型）：

```python
from train import Train

results = Train(
    pretrained_path=None,                    # None → 从 YAML 新建模型
    dataset="cfg/datasets/coco_cat.yaml",
    imgsz=224,                               # 方形 224×224
    epochs=1200,
    batch=128,
    device="cpu",                            # 换成 "0" 用 GPU
)
# best.pt 路径
model_path = f"{results.save_dir}/weights/best.pt"
print("best:", model_path)
```

### 3. 加载已有权重继续训练

```python
from train import Train
results = Train(
    pretrained_path="path/to/best.pt",       # 加载已有权重微调
    dataset="cfg/datasets/coco_cat.yaml",
    imgsz=224,
    epochs=100,
    device="0",
)
```

### 4. 训练设置可被 kwargs 覆盖

`Train(..., **kwargs)` 会 `train_setting.update(kwargs)`，故可覆盖任意 Ultralytics 训练参数（参考 [Train Settings](https://docs.ultralytics.com/modes/train/#train-settings)）：

```python
results = Train(
    pretrained_path=None,
    dataset="cfg/datasets/coco_cat.yaml",
    imgsz=416,                # 416×416，精度更高、速度更慢
    epochs=900,
    batch=64,
    device="0",
    lr0=0.01,                 # 覆盖默认学习率
    patience=100,             # 早停
)
```

### 5. 验证浮点模型 mAP（来自 `val.py`）

```python
import ultralytics.nn.tasks as tasks
from nn.esp_tasks import custom_parse_model
tasks.parse_model = custom_parse_model
from ultralytics import YOLO

model = YOLO("path/to/best.pt")
metrics = model.val(data='cfg/datasets/coco_cat.yaml', save_json=True)
print("mAP@0.5:0.95:", metrics.box.map)   # 224 模型在 cat 子集约 69.9
print("mAP@0.5     :", metrics.box.map50)
print("per-class   :", metrics.box.maps)
```

## 常见错误

| 错误 | 原因 | 解决方法 |
|---|---|---|
| `KeyError: 'ESPBlock'` 等 | 未注入 `custom_parse_model` | 训练脚本入口加 `tasks.parse_model = custom_parse_model` |
| OOM（显存不足） | batch=128 太大 | 调小 `batch` 或换多卡 `device=[0,1,2,3]` |
| `imgsz` 维度歧义 | 方形却传列表 | 方形用标量 `224` 或 `[224,224]`；列表+`rect=True` 才走非方形路径 |
| 训练极慢 | 用了 CPU 且 batch 大 | 改 `device="0"`（GPU）或减小 batch/imgsz |
| mAP 异常低 | 数据 `nc` 与模型 `nc` 不符 | 同步 `cfg/models/espdet_pico.yaml` 的 `nc` 与 dataset `names` |

## 参考

- 仓库 `train.py`（`Train` 默认设置）
- 仓库 `val.py`（浮点验证示例）
- 仓库 `cfg/models/espdet_pico.yaml`
- `recipes/export_onnx.md`（训练后导出）
- Ultralytics [Train Settings](https://docs.ultralytics.com/modes/train/#train-settings)
