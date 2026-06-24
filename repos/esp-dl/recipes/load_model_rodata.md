# 从 .rodata 加载模型

> **适用摘要**: 把 `.espdl` 模型嵌入应用的 `.rodata` 段，用 `MODEL_LOCATION_IN_FLASH_RODATA` 加载。最简单的加载方式，缺点是改代码也会重新烧模型。

## 触发意图

- "把模型嵌入固件"
- "从 rodata 加载 espdl"
- "最简单的模型加载方式"

## 前置条件

| 条件 | 要求 |
|---|---|
| 参考示例 | `examples/tutorial/how_to_run_model`、`examples/tutorial/how_to_load_test_profile_model/model_in_flash_rodata` |

## 分步说明

### 1. `main/CMakeLists.txt` 嵌入模型

```cmake
idf_build_get_property(component_targets __COMPONENT_TARGETS)
if ("___idf_espressif__esp-dl" IN_LIST component_targets)
   idf_component_get_property(espdl_dir espressif__esp-dl COMPONENT_DIR)
elseif("___idf_esp-dl" IN_LIST component_targets)
   idf_component_get_property(espdl_dir esp-dl COMPONENT_DIR)
endif()
include(${espdl_dir}/fbs_loader/cmake/utilities.cmake)

set(embed_files models/s3/model.espdl)

idf_component_register(SRCS app_main.cpp REQUIRES esp-dl)
target_add_aligned_binary_data(${COMPONENT_LIB} ${embed_files} BINARY)
```

### 2. 在代码中加载

```cpp
#include "dl_model_base.hpp"

// 符号名 = "_binary_" + 文件名(点→下划线) + "_start"
extern const uint8_t model_espdl[] asm("_binary_model_espdl_start");

extern "C" void app_main(void)
{
    // 构造即完成加载、构建执行计划、内存规划；输入/输出内存已分配
    dl::Model *model = new dl::Model((const char *)model_espdl,
                                     fbs::MODEL_LOCATION_IN_FLASH_RODATA);

    // 高级用法（参数留在 FLASH、限内部 RAM 0 字节、贪心内存管理、不拷贝参数）：
    // dl::Model *model = new dl::Model((const char *)model_espdl,
    //                                  fbs::MODEL_LOCATION_IN_FLASH_RODATA,
    //                                  0,                            // max_internal_size
    //                                  dl::MEMORY_MANAGER_GREEDY,    // mm_type
    //                                  nullptr,                      // key (加密模型)
    //                                  false);                       // param_copy=false 参数留 FLASH

    model->test();      // 需量化时开启 export_test_values
    // model->profile(); // 打印内存与逐层延迟

    delete model;
}
```

### 3. 大模型需扩大 app 分区

若模型很大，把 `partitions.csv` 里 app 分区调大，否则链接/烧录失败。

## 常见错误

| 错误 | 原因 | 解决方法 |
|---|---|---|
| 链接报 `_binary_*_start` 未定义 | 符号名拼错 | 文件名里的 `.` 要替换为 `_` |
| 改一行代码也要重烧大模型 | 模型随 app 一起烧 | 改用分区或 SD 卡方式 |
| Flash 不够 | 模型+app 超分区 | 增大 app 分区或换分区加载 |
| 对齐异常 | 用了 `EMBED_FILES` | 改用 `target_add_aligned_binary_data` |

## 参考

- `examples/tutorial/how_to_run_model/main/app_main.cpp`
- `docs/en/tutorials/how_to_load_test_profile_model.rst`（Load model from rodata 一节）
