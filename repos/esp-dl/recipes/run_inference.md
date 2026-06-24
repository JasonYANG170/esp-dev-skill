# 执行模型推理（输入量化、运行、输出反量化）

> **适用摘要**: 加载 `.espdl` 后，获取输入/输出 TensorBase，对 float 输入做量化，调用 `run()`，再把 int 输出反量化为 float。这是 ESP-DL 最核心的推理流程。

## 触发意图

- "怎么跑模型推理"
- "输入怎么量化"
- "输出怎么反量化"
- "dl::Model run 用法"

## 前置条件

| 条件 | 要求 |
|---|---|
| 参考示例 | `examples/tutorial/how_to_run_model/main/app_main.cpp` |

## 分步说明

### 1. 量化和反量化的数学关系

ESP-DL 使用对称量化 + 2 的幂次 scale：

```
Q   = Clip(Round(R / Scale), MIN, MAX),  Scale = 2^exp
R'  = Q * Scale

8 bit:  MIN=-128, MAX=127
16 bit: MIN=-32768, MAX=32767
```

宏定义（来自 `dl_define.hpp`）：

```cpp
#define DL_SCALE(exponent)    (((exponent) > 0) ? (1 << (exponent))  : ((float)1.0 / (1 << -(exponent))))
#define DL_RESCALE(exponent)  (((exponent) > 0) ? ((float)1.0 / (1 << (exponent))) : (1 << -(exponent)))
```

注意：`dl::quantize` 的第二个参数是 **inverse scale**（用 `DL_RESCALE`），`dl::dequantize` 的第二个参数是 **scale**（用 `DL_SCALE`）。

### 2. 方式 A：单值量化 + 直接写 input 指针

```cpp
#include "dl_model_base.hpp"
#include <cmath>

extern const uint8_t model_espdl[] asm("_binary_model_espdl_start");

extern "C" void app_main(void)
{
    dl::Model *model = new dl::Model((const char *)model_espdl,
                                     fbs::MODEL_LOCATION_IN_FLASH_RODATA);

    dl::TensorBase *model_input  = model->get_inputs().begin()->second;
    dl::TensorBase *model_output = model->get_outputs().begin()->second;

    float input_v = 90 * M_PI / 180;
    int8_t q_in = dl::quantize<int8_t>(input_v, DL_RESCALE(model_input->exponent));
    *((int8_t *)model_input->data) = q_in;

    model->run();   // 默认 RUNTIME_MODE_SINGLE_CORE

    int8_t q_out = *((int8_t *)model_output->data);   // 必须在再次 run 前读出
    float output_v = dl::dequantize(q_out, DL_SCALE(model_output->exponent));
    printf("result = %f\n", output_v);

    delete model;
}
```

### 3. 方式 B：用 `TensorBase::assign` 整体量化一个 float 张量

```cpp
// 构造一个 shape=[1,1] 的 float 张量
dl::TensorBase *input_tensor = new dl::TensorBase({1, 1}, nullptr, 0, dl::DATA_TYPE_FLOAT);
*((float *)input_tensor->data) = input_v;

// assign 内部按 model_input 的 exponent/dtype 完成量化
model_input->assign(input_tensor);

model->run();

// 用一个 float 张量接收反量化后的输出
dl::TensorBase *output_tensor = new dl::TensorBase({1, 1}, nullptr, 0, dl::DATA_TYPE_FLOAT);
output_tensor->assign(model_output);
float output_v = *((float *)output_tensor->data);

delete input_tensor;
delete output_tensor;
```

### 4. 方式 C：用 `run(user_inputs, mode, user_outputs)` 直接传入/传出张量

```cpp
dl::TensorBase *input_tensor = new dl::TensorBase({1, 1}, nullptr, 0, dl::DATA_TYPE_FLOAT);
*((float *)input_tensor->data) = input_v;

std::string in_name  = model->get_inputs().begin()->first;
std::string out_name = model->get_outputs().begin()->first;
std::map<std::string, dl::TensorBase *> in_map  = {{in_name,  input_tensor}};

dl::TensorBase *output_tensor = new dl::TensorBase({1, 1}, nullptr, 0, dl::DATA_TYPE_FLOAT);
std::map<std::string, dl::TensorBase *> out_map = {{out_name, output_tensor}};

model->run(in_map, dl::RUNTIME_MODE_SINGLE_CORE, out_map);

float output_v = *((float *)output_tensor->data);
```

### 5. 多核加速

`run()` 默认单核。要让 Conv2D / DepthwiseConv2D 用上第二核：

```cpp
model->run(dl::RUNTIME_MODE_MULTI_CORE);
// 或交��框架自动选择
model->run(dl::RUNTIME_MODE_AUTO);
```

## 常见错误

| 错误 | 原因 | 解决方法 |
|---|---|---|
| 推理结果全错 | float 直接写进 `data`，没量化 | 用 `dl::quantize` + `DL_RESCALE`，或 `assign` |
| 第二次读输出得到脏数据 | 内存规划器复用 buffer | 每次 `run()` 后立即读取/拷贝所需输出 |
| 量化/反量化数值不对 | 宏用反 | 量化传 `DL_RESCALE`，反量化传 `DL_SCALE` |
| 第二核没被用上 | 默认 `RUNTIME_MODE_SINGLE_CORE` | 显式传 `RUNTIME_MODE_MULTI_CORE` |
| INT16 模型按 int8 取值 | dtype 不匹配 | 16 bit 用 `int16_t`，`DATA_TYPE_INT16` |

## 参考

- `examples/tutorial/how_to_run_model/main/app_main.cpp`（run1 / run2 / run3 三种用法）
- `docs/en/tutorials/how_to_run_model.rst`
- `esp-dl/dl/tensor/include/dl_tensor_base.hpp`（quantize / dequantize / assign）
