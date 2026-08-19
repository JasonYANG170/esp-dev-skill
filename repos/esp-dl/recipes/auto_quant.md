# 用 AutoQuant 自动搜索最优量化配置

> **适用摘要**: 当默认 8-bit 量化精度不足时，用 `espdl_auto_quantize_onnx` 在多种量化策略与参数间自动搜索，按评估指标保留 Top-K 候选 `.espdl`，减少人工调参。

> Version: selected repo/component version; align with the user project and dependency manifest.
> Evidence: `repos/esp-dl/resources/`, source/examples in `repos/esp-dl/`, and this recipe path `repos/esp-dl/recipes/auto_quant.md`.
> Validation: draft metadata added from repository routing; verify APIs, Kconfig symbols, and component dependencies against the selected version.

## 触发意图

- "量化精度不够怎么办"
- "自动搜索量化配置"
- "AutoQuant 用法"
- "mixed precision 量化"

## 前置条件

| 条件 | 要求 |
|---|---|
| ESP-PPQ | `pip install esp-ppq`（含 `esp_ppq.autoquant`） |
| 参考文档 | `docs/en/tutorials/auto_quantization/how_to_use_AutoQuant.rst` |
| 示例 | `python -m esp_ppq.samples.AutoQuant.mobilenetv2`、`...yolo11n` |

## 分步说明

### 1. 准备评估函数（可选但推荐）

```python
def evaluate_fn(graph):
    # 用你的验证逻辑返回 (score, extras)
    top1, top5 = ...   # 例如分类 top1/top5；检测可用 mAP
    return top1, {"top1": top1, "top5": top5}
```

约束：返回 `(score, extras)` 元组；`score` 必须是有限数（不能是 nan/inf/bool）；`extras` 是 dict，且不能使用保留字段 `score/hash/index/folder/files/strategy/params`。

不提供 `evaluate_fn` 时，AutoQuant 默认用 `graphwise_error_analyse` 计算的 SNR，取误差最大的 3 层均值作为评分（此时应设 `score_direction="minimize"`）。

### 2. 配置搜索设置 `AutoQuantSearchSetting`

```python
from esp_ppq.api import AutoQuantSearchSetting, espdl_auto_quantize_onnx

setting = AutoQuantSearchSetting(
    search_mode="fast",           # "exhaustive" 枚举全部；"fast" 先快筛再细搜
    num_of_candidates=5,          # 最终保留 Top-K
    score_direction="maximize",   # 精度/mAP 用 maximize；误差用 minimize
    run_dir="outputs/auto_quant_mobilenetv2",
)

# fast 模式可进一步调：
setting.top_strategy = 10         # 快筛后保留多少策略组
setting.early_stop_patience = 3   # 连续 N 次无提升则停止

# 可选：微调策略/参数空间
setting.strategy_space["mixed_precision"] = [True, False]
setting.param_space["calib_algorithm"]["method"] = ["kl", "mse"]
```

| 字段 | 默认 | 说明 |
|---|---|---|
| `search_mode` | `"exhaustive"` | `exhaustive` 枚举；`fast` 先筛后搜 |
| `num_of_candidates` | `5` | Top-K |
| `score_direction` | `"maximize"` | 排序方向 |
| `candidate_filter` | `low_latency_candidate_filter` | Top-K 过滤；纯按分排序设 `None` |
| `run_dir` | `"outputs/auto_quant"` | 结果目录 |
| `resume` | `False` | 从已有 `run_dir` 续搜 |

### 3. 启动搜索

```python
espdl_auto_quantize_onnx(
    onnx_import_file="model.onnx",
    espdl_export_file="outputs/model.espdl",
    calib_dataloader=calib_loader,
    calib_steps=32,
    input_shape=[3, 224, 224],     # 不含 batch
    evaluate_fn=evaluate_fn,
    target="esp32p4",
    setting=setting,
    device="cuda",
)
```

### 4. 读取结果

```
<run_dir>/
    summary.json          # 所有已完成实验
    candidates.json       # 当前 Top-K 候选
    0000/
        config.json
        model.espdl
        model.info
        model.json
        model.native
    0001/ ...
```

`resume=True` 会读 `summary.json` 跳过已完成的 `(strategy, params)` 组合，用于断点续搜或精细化搜索。

## 常见错误

| 错误 | 原因 | 解决方法 |
|---|---|---|
| 评分方向反了 | `score_direction` 与指标语义不符 | 精度/mAP 用 `maximize`，误差/loss 用 `minimize` |
| 无 `evaluate_fn` 时排序异常 | 默认按误差均值，方向应为 minimize | 不传评估函数时设 `score_direction="minimize"` |
| 搜索太久 | `exhaustive` 枚举量大 | 改 `fast` 模式并调 `top_strategy` / `early_stop_patience` |
| `extras` 报保留字段冲突 | 用了保留键名 | 避开 `score/hash/index/folder/files/strategy/params` |

## 参考

- `docs/en/tutorials/auto_quantization/how_to_use_AutoQuant.rst`
- `docs/en/tutorials/auto_quantization/how_to_use_espdl_quantize_skill.rst`
- ESP-PPQ 样例：`python -m esp_ppq.samples.AutoQuant.mobilenetv2`
