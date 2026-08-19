# 测试、剖析模型内存与延迟（test / profile）

> **适用摘要**: 用 `model->test()` 验证板端推理正确性，用 `profile_memory()` / `profile_module()` / `profile()` 打印内存占用和逐层延迟，定位精度与性能瓶颈。

> Version: selected repo/component version; align with the user project and dependency manifest.
> Evidence: `repos/esp-dl/resources/`, source/examples in `repos/esp-dl/`, and this recipe path `repos/esp-dl/recipes/profile_test.md`.

## 触发意图

- "验证模型推理对不对"
- "测模型延迟"
- "看模型用了多少内存"
- "profile espdl"

## 前置条件

| 条件 | 要求 |
|---|---|
| 模型 | 量化时开启 `export_test_values=True`（test 才有效） |
| 参考示例 | `examples/tutorial/how_to_load_test_profile_model/model_in_flash_partition` |

## 分步说明

### 1. `test()`：比对板端结果与模型内嵌的真值

```cpp
#include "dl_model_base.hpp"

extern "C" void app_main(void)
{
    dl::Model *model = new dl::Model("model", fbs::MODEL_LOCATION_IN_FLASH_PARTITION);

    esp_err_t ret = model->test();
    if (ret == ESP_OK) {
        ESP_LOGI("TAG", "Model test passed!");
    } else {
        ESP_LOGE("TAG", "Model test failed!");
    }
    // 或：ESP_ERROR_CHECK(model->test());

    delete model;
}
```

��作原理：加载模型内嵌的测试输入 → 跑完整推理 → 与内嵌真值逐输出比对（带量化误差容差）。

> INT16 模型由于量化舍入，`test()` 允许每个元素 ±1 的差异。

### 2. `profile_memory()`：内存分类占用

```cpp
model->profile_memory();
```

输出包含 `fbs_model`（含 parameter）、`parameter_copy`（仅 `param_copy=true` 时出现）、`variable`（输入/输出/中间张量）、`others`、`total`，并按 内部 RAM / PSRAM / FLASH 分别统计。

### 3. `profile_module()`：逐层延迟

```cpp
model->profile_module();        // 按 ONNX 拓扑序
model->profile_module(true);    // 按延迟降序
```

输出每层：name、type（算子类型）、延迟（微秒，或开启 `DL_LOG_LATENCY_UNIT` 时为 cycle），末尾给出总延迟。

### 4. `profile()`：内存 + 延迟合并

```cpp
model->profile();        // 拓扑序
model->profile(true);    // 按延迟降序
```

### 5. 编程式获取（不打印）

```cpp
std::map<std::string, mem_info_t>    mem = model->get_memory_info();
std::map<std::string, module_info>   mod = model->get_module_info();
model->print_module_info(mod, /*sort_module_by_latency=*/false);
```

## 常见错误

| 错误 | 原因 | 解决方法 |
|---|---|---|
| `test()` 返回 `ESP_FAIL` | 模型未带测试值，或 target/芯片不符 | 用 `export_test_values=True` 重新导出；确认 target 与芯片一致 |
| INT16 `test()` 偶发失败 | 量化舍入导致 ±1 | INT16 允许 ±1，属正常；勿手动改判定 |
| 延迟看不出热点 | 默认拓扑序 | 传 `true` 按延迟降序打印 |
| `profile_memory` 没有 parameter_copy 行 | `param_copy=false` | 想看该项需保持默认 `param_copy=true` |

## 参考

- `examples/tutorial/how_to_load_test_profile_model/model_in_flash_partition/main/app_main.cpp`
- `docs/en/tutorials/how_to_load_test_profile_model.rst`（Test / Profile 各节）
- `esp-dl/dl/model/include/dl_model_base.hpp`（`test` / `profile*` / `get_memory_info` / `get_module_info`）
