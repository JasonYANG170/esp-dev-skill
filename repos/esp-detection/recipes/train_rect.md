# 非方形分辨率 rect=True 训练

> **适用摘要**: 对非方形输入（如 160×288）启用 `rect=True` 训练，含可选的方形预训练阶段，以在不增加模型复杂度/推理时间的前提下提升精度与速度。

## 触发意图

- "rect 训练"
- "非方形输入训练"
- "160×288 训练"
- "预训练 + 微调两阶段"

## 前置条件

| 条件 | 要求 |
|---|---|
| 环境 | 已按 `recipes/env_setup.md` 搭建 |
| 数据 | 已按 `recipes/dataset_prepare.md` 准备 |
| 教程 | `docs/tutorials/how_to_train_and_deploy_model_with_rect_is_True.md` |

## 分步说明

`rect=True` 的价值（来自官方教程）：非方形场景下减少冗余计算、降低延迟、增强特征学习，从而在模型复杂度与运行时间不变的情况下提升精度。

### 阶段 1（可选）：方形预训练 rect=False

预训练用较大方形尺寸（`max(h,w)`），关闭 rect（来自教程）：

```python
import os
import ultralytics.nn.tasks as tasks
from nn.esp_tasks import custom_parse_model
tasks.parse_model = custom_parse_model
from train import Train

# 以 espdet_pico_160_288_cat 为例：最终 [h,w]=[160,288]，预训练用 288 方形
results = Train(
    dataset="cfg/datasets/coco_cat.yaml",
    imgsz=288,              # 方形预训练，用 max(h,w)
    epochs=900,
    rect=False,
    device="cpu",           # 或 "0"
)
model_path = os.path.join(str(results.save_dir), "weights/best.pt")
```

> 不想预训练可直接跳到阶段 2，并把 `pretrained_path=None`。

### 阶段 2：rect=True 微调

`imgsz=[h, w]`（h≠w），`rect=True`，微调轮数 30~50（来自教程与 `espdet_run.py`）：

```python
rect_results = Train(
    pretrained_path=model_path,      # 用阶段 1 的权重；不预训练则 None
    dataset="cfg/datasets/coco_cat.yaml",
    imgsz=[160, 288],                # 必须是 [h, w] 列表
    epochs=30,                       # 30~50
    rect=True,
    device="cpu",
)
rect_model_path = os.path.join(str(rect_results.save_dir), "weights/best.pt")
```

### 使用 espdet_run.py 自动判断

`espdet_run.py` 的 `run()` 会根据 `size` 自动选路径（来自源码）：

```python
h, w = size
if h != w:
    print("Adopt rect=True training strategy")
    results = Train(pretrained_path, dataset, size, rect=True)   # 内部 imgsz=[h,w], rect=True
else:
    results = Train(pretrained_path, dataset, size)              # 方形，rect=False
```

故非方形直接：

```bash
python espdet_run.py --class_name mycat --pretrained_path None \
  --dataset "cfg/datasets/coco_cat.yaml" --size 160 288 \
  --target esp32p4 --calib_data "deploy/cat_calib" \
  --espdl "espdet_pico_160_288_mycat.espdl" --img "espdet.jpg"
```

### rect 模型的后续阶段

rect 模型的导出与量化 `imgsz` 同样传 `[h, w]`（见 `recipes/export_onnx.md`、`recipes/quantize_espdl.md`）：

```python
Export(rect_model_path, [160, 288])
quant_espdet(onnx_path=ONNX, target="esp32p4", imgsz=[160, 288], ...)
```

## 常见错误

| 错误 | 原因 | 解决方法 |
|---|---|---|
| `rect=True` 却传标量 imgsz | imgsz 必须是 `[h, w]` | 改为 `imgsz=[160, 288]` |
| 形状不匹配报错 | 预训练方形与微调非方形 stride 不兼容 | 预训练用 `imgsz=max(h,w)` 且 `rect=False`；微调才 `rect=True` |
| 微调过拟合 / 精度不升 | epochs ���多 | 微调 30~50 轮即可，过多反而退化 |
| `size` 顺序写反 | 高宽颠倒导致变形 | `[h, w]` = `[高度, 宽度]`，如 `[160, 288]` 是宽图 |
| 量化后 rect 模型推理错位 | 导出/量化的 imgsz 与训练不一致 | 导出、量化、芯片端预处理三处的 `[h, w]` 完全一致 |

## 参考

- `docs/tutorials/how_to_train_and_deploy_model_with_rect_is_True.md`
- 仓库 `espdet_run.py`（`run()` 的 rect 分支）
- 仓库 `train.py`（`Train` 支持 `rect` / `imgsz=[h,w]`）
- `recipes/export_onnx.md`、`recipes/quantize_espdl.md`
