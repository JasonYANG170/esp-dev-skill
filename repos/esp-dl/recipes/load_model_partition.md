# 从 FLASH 分区加载模型

> **适用摘要**: 把 `.espdl` 存到独立的 SPIFFS 数据分区，用 `MODEL_LOCATION_IN_FLASH_PARTITION` 加载。模型可独立于 app 更新，开发期可用 `idf.py app-flash` 跳过模型分区。

> Version: selected repo/component version; align with the user project and dependency manifest.
> Evidence: `repos/esp-dl/resources/`, source/examples in `repos/esp-dl/`, and this recipe path `repos/esp-dl/recipes/load_model_partition.md`.
> Validation: draft metadata added from repository routing; verify APIs, Kconfig symbols, and component dependencies against the selected version.

## 触发意图

- "模型放分区"
- "独立更新模型不重新烧 app"
- "MODEL_LOCATION_IN_FLASH_PARTITION"

## 前置条件

| 条件 | 要求 |
|---|---|
| 参考示例 | `examples/tutorial/how_to_load_test_profile_model/model_in_flash_partition` |

## 分步说明

### 1. `partitions.csv` 添加模型分区

```
# Name,  Type, SubType, Offset,  Size,   Flags
factory, app,  factory, 0x010000,4000K,
model,   data, spiffs,  ,        4000K,
```

- Name：任意，但 ≤ 16 字符（含结尾 `\0`），需与代码/CMake 中一致
- Type：`data`
- SubType：必须是 `spiffs`
- Size：必须大于模型文件大小

### 2. `main/CMakeLists.txt` 声明烧写到分区

```cmake
set(srcs app_main.cpp)
set(requires esp-dl)

idf_component_register(SRCS ${srcs} REQUIRES ${requires})

if (IDF_TARGET STREQUAL "esp32s3")
    set(image_file ${COMPONENT_DIR}/models/s3/model.espdl)
elseif (IDF_TARGET STREQUAL "esp32p4")
    set(image_file ${COMPONENT_DIR}/models/p4/model.espdl)
endif()

# 第二个参数必须等于 partition.csv 的 Name
esptool_py_flash_to_partition(flash "model" "${image_file}")
```

### 3. 代码加载

```cpp
#include "dl_model_base.hpp"

extern "C" void app_main(void)
{
    // 第一个参数 = 分区 label，必须等于 partition.csv 的 Name
    // 以下示例：参数留 FLASH（param_copy=false），节省 PSRAM/内部 RAM，代价是推理变慢
    dl::Model *model = new dl::Model("model",
                                     fbs::MODEL_LOCATION_IN_FLASH_PARTITION,
                                     0,                          // max_internal_size
                                     dl::MEMORY_MANAGER_GREEDY,  // mm_type
                                     nullptr,                    // key
                                     false);                     // param_copy

    ESP_ERROR_CHECK(model->test());
    model->profile();
    delete model;
}
```

### 4. 开发期只烧 app

```bash
idf.py app-flash monitor    # 只更新 app 分区，不重烧 model 分区
```

## 常见错误

| 错误 | 原因 | 解决方法 |
|---|---|---|
| 找不到分区 | 分区 label 与代码/CMake 不一致 | 三处 Name 必须完全相同 |
| 分区类型错误 | SubType 不是 `spiffs` | SubType 必须为 `spiffs` |
| 分区过小 | Size 小于模型文件 | 增大分区 Size |
| 加密模型读不出 | 未传 `key` | 构造时第 5 个参数传 16+ 字节密钥指针 |

## 参考

- `examples/tutorial/how_to_load_test_profile_model/model_in_flash_partition/main/app_main.cpp`
- `examples/tutorial/how_to_load_test_profile_model/model_in_flash_partition/main/CMakeLists.txt`
- `docs/en/tutorials/how_to_load_test_profile_model.rst`（Load model from partition 一节）
