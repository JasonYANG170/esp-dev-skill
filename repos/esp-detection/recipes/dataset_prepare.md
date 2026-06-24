# 准备 YOLO 格式数据集

> **适用摘要**: 按 Ultralytics YOLO 检测格式组织数据，编写 `cfg/datasets/*.yaml`，并按需启用 negative sampling（负样本）或 weighted sampling（类别均衡）。

## 触发意图

- "准备检测数据集"
- "写 dataset yaml"
- "负样本 / negative sampling"
- "类别不均衡 / weighted dataset"

## 前置条件

| 条件 | 要求 |
|---|---|
| 数据格式 | YOLO detect 格式（`images/` + `labels/*.txt`，每行 `cls cx cy w h`） |
| 参考配置 | `cfg/datasets/coco_cat.yaml`、`cfg/datasets/esp_cat.yaml` |

## 分步说明

### 1. 目录结构（YOLO 格式）

```
datasets/coco_cat/
├── images/
│   ├── train/  *.jpg
│   └── val/    *.jpg
└── labels/
    ├── train/  *.txt   # 与 images 同名，内容: <cls> <cx> <cy> <w> <h>（归一化）
    └── val/    *.txt
```

> 从 COCO 等格式转 YOLO 可用 Ultralytics 的 [converter.py](https://github.com/ultralytics/ultralytics/blob/main/ultralytics/data/converter.py)。

### 2. 基础 dataset YAML（来自 `cfg/datasets/coco_cat.yaml`）

```yaml
# cfg/datasets/coco_cat.yaml
path: datasets/coco_cat      # 数据集根（相对仓库根或绝对路径）
train: images/train          # 训练图（相对 path）
val: images/val              # 验证图
test:                        # 可选

names:
  0: cat                     # 类别 id → 名称；单类检测 nc=1
```

### 3. 单类检测的 nc 必须为 1

`cfg/models/espdet_pico.yaml` 顶部 `nc: 1` 与 dataset 的 `names` 数量必须一致：

```yaml
# espdet_pico.yaml
nc: 1
```

多类时同步改 `nc` 与 `names`，并准备对应数量类别的标注。

### 4. 启用负样本（negative sampling）

参考 `cfg/datasets/esp_cat.yaml` 与 `data/esp_dataset.py::YOLOPosNegDataset`：

```yaml
# cfg/datasets/esp_cat.yaml
path: esp_cat
train: images/train
val: images/val
test:

negative_setting:
  neg_ratio: 0.111                              # 负样本采样比例
  use_extra_neg: True
  extra_neg_sources: { "esp_cat/negative_images": 100014 }   # 额外负样本目录:期望数量
  fix_dataset_length: 101241                    # 固定一个 epoch 的迭代数；0 表示用原始长度

names:
  0: cat
```

启用方式：训练时把数据集类替换为 `YOLOPosNegDataset`（在 Ultralytics 配置或自定义训练脚本里指定 `dataset` 为该类的 yaml），`get_labels()` 会自动合并正/负样本索引并按 `neg_ratio` 在 `__getitem__` 中按概率采样负样本。

### 5. 启用类别均衡（weighted sampling）

参考 `data/esp_dataset.py::YOLOWeightedDataset`：按各类别实例数倒数加权，稀有类别被更频繁采样。类内部自动 `count_instances()` → `calculate_weights()` → `calculate_probabilities()`，训练时 `__getitem__` 用 `np.random.choice(len(labels), p=probabilities)` 选索引。

### 6. 验证数据集可被加载

```python
import ultralytics.nn.tasks as tasks
from nn.esp_tasks import custom_parse_model
tasks.parse_model = custom_parse_model
from ultralytics import YOLO

model = YOLO('cfg/models/espdet_pico.yaml')
metrics = model.val(data='cfg/datasets/coco_cat.yaml', imgsz=224)
print('map50-95:', metrics.box.map)
```

## 常见错误

| 错误 | 原因 | 解决方法 |
|---|---|---|
| `No labels found` / 0 mAP | `labels/*.txt` 与 `images/*` 不同名或路径错 | 检查 `path`/`train`/`val`；标签文件名与图片一致 |
| `nc` 与类别数不符 | 改了 `names` 没改 `nc` | 同步 `cfg/models/espdet_pico.yaml` 的 `nc` |
| 负样本不生效 | yaml 缺 `negative_setting` 或用了默认 `YOLODataset` | 按上面格式加 `negative_setting` 并指定 `YOLOPosNegDataset` |
| `.cache` 校验失败 | 改了图片/标签后旧 cache 未删 | 删除 `labels/*.cache` 重新生成 |
| `fix_dataset_length` 报错 | yaml 没配或值为非 int | 设为整数（0=用原始长度） |

## 参考

- `cfg/datasets/coco_cat.yaml`、`cfg/datasets/esp_cat.yaml`
- `data/esp_dataset.py`（`YOLOPosNegDataset`、`YOLOWeightedDataset`）
- 仓库 `README.md`（Step 1: Prepare dataset）
- Ultralytics [YOLO detect dataset format](https://docs.ultralytics.com/datasets/detect/)
