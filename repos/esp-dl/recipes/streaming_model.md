# 部署流式（流式/时序）模型

> **适用摘要**: 把长时序输入（如音频）切分成 chunk，逐 chunk 喂给流式 `.espdl` 模型；模型内部自动维护跨 chunk 的状态（StreamingCache）。适合音频/语音等实时场景。

## 触发意图

- "部署流式模型"
- "音频模型逐帧推理"
- "streaming model"
- "TCN 流式部署"

## 前置条件

| 条件 | 要求 |
|---|---|
| 参考示例 | `examples/tutorial/how_to_deploy_streaming_model` |
| 量化工具 | ESP-PPQ，开启 `auto_streaming=True` |

## 分步说明

### 1. 量化时导出流式模型

ESP-PPQ 提供 `auto_streaming=True` 自动插入 `StreamingCache` 节点；`streaming_input_shape` 是每个 chunk 的输入形状（时间维较小）：

```python
from esp_ppq.api import espdl_quantize_torch

# 离线（完整）模型
espdl_quantize_torch(
    model=model,
    espdl_export_file="model.espdl",
    calib_dataloader=dataloader,
    calib_steps=32,
    input_shape=[1, 16, 15],     # 离线完整输入
    target="esp32s3",
    num_of_bits=8,
    device="cpu",
    export_test_values=True,
)

# 流式模型（自动转换）
espdl_quantize_torch(
    model=model,
    espdl_export_file="model_streaming.espdl",
    calib_dataloader=dataloader,
    calib_steps=32,
    input_shape=[1, 16, 15],
    target="esp32s3",
    num_of_bits=8,
    device="cpu",
    auto_streaming=True,
    streaming_input_shape=[1, 16, 3],   # 每个 chunk 的形状
    streaming_table=None,
)
```

### 2. 手动指定无法自动处理的变量（可选）

对 Transpose/Reshape/Slice 等无法自动插入 cache 的算子，用 `insert_streaming_cache_on_var`：

```python
from esp_ppq.api import insert_streaming_cache_on_var

streaming_table = []
streaming_table.append(insert_streaming_cache_on_var("/out_conv/Conv_output_0", window_size=output_frame_size - 1))
streaming_table.append(insert_streaming_cache_on_var("PPQ_Variable_0", 1, "/Slice"))

espdl_quantize_torch(..., auto_streaming=True,
                     streaming_input_shape=[1, 16, 3],
                     streaming_table=streaming_table)
```

### 3. 部署：按 chunk 喂数据

来自仓库示例的部署骨架（每个 chunk 调一次 `run()`，内部状态自动延续）：

```cpp
#include "dl_model_base.hpp"

dl::TensorBase *run_streaming_model(dl::Model *model, dl::TensorBase *test_input)
{
    dl::TensorBase *model_input  = model->get_inputs().begin()->second;
    dl::TensorBase *model_output = model->get_outputs().begin()->second;

    int test_input_size  = test_input->get_bytes();
    uint8_t *test_input_ptr = (uint8_t *)test_input->data;
    int model_input_size = model_input->get_bytes();
    uint8_t *model_input_ptr = (uint8_t *)model_input->data;

    int chunks = test_input_size / model_input_size;
    for (int i = 0; i < chunks; i++) {
        memcpy(model_input_ptr, test_input_ptr + i * model_input_size, model_input_size);
        model->run(model_input);
    }
    return model_output;   // 与等价离线模型的末段输出对应
}
```

> chunk 数 = 完整输入字节数 / 流式模型输入字节数。每个 chunk 调一次 `run()`，跨 chunk 状态由 StreamingCache 自动维护。

## 常见错误

| 错误 | 原因 | 解决方法 |
|---|---|---|
| 流式输出与离线对不上 | chunk 切分或 cache window 配错 | 检查 `streaming_input_shape` 与 `streaming_table` 的 window_size |
| 自动转换漏掉了某算子 | 该算子不在自动支持列表 | 用 `insert_streaming_cache_on_var` 手动指定 |
| `test()` 失败 | 流式模型导出时未开 `export_test_values`，或 chunk 对齐错 | 用 `export_test_values=True` 重新导出并核对 chunk 数 |

## 参考

- `docs/en/tutorials/how_to_deploy_streaming_model.rst`
- `examples/tutorial/how_to_deploy_streaming_model/test_streaming_model/main/app_main.cpp`
- `examples/tutorial/how_to_deploy_streaming_model/quantize_streaming_model/quantize_torch_model.py`
