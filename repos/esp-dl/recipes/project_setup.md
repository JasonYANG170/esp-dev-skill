# 项目集成：把 esp-dl 作为托管组件加入工程

> **适用摘要**: 通过 ESP-IDF Component Registry 把 `esp-dl` 加入新工程，配置 `idf_component.yml` 与 `CMakeLists.txt`，准备加载 `.espdl` 模型。

## 触发意图

- "在工程里加入 esp-dl"
- "如何使用 esp-dl 组件"
- "esp-dl 依赖怎么配"
- "新建一个 ESP-DL 推理工程"

## 前置条件

| 条件 | 要求 |
|---|---|
| ESP-IDF | `release/v5.3` 及以上（ESP32-C5 需 `>=5.5`，ESP32-S31 需 `>=6.0`） |
| 目标芯片 | ESP32 / ESP32-S3 / ESP32-P4 / C / S 系列 |
| 参考示例 | `examples/tutorial/how_to_run_model` |

## 分步说明

### 1. 声明依赖（`main/idf_component.yml`）

```yaml
version: "0.0.1"
description: My ESP-DL inference app
dependencies:
  espressif/esp-dl:
    version: "^3.3"
```

`esp-dl` 组件名在 CMake target 里可能呈现为 `___idf_espressif__esp-dl` 或 `___idf_esp-dl`，因此后续 CMake 片段需同时兼容两者。

### 2. 在 `main/CMakeLists.txt` 中引用 fbs_loader 的 cmake 工具

```cmake
set(srcs app_main.cpp)
set(requires esp-dl)

idf_build_get_property(component_targets __COMPONENT_TARGETS)
if ("___idf_espressif__esp-dl" IN_LIST component_targets)
   idf_component_get_property(espdl_dir espressif__esp-dl COMPONENT_DIR)
elseif("___idf_esp-dl" IN_LIST component_targets)
   idf_component_get_property(espdl_dir esp-dl COMPONENT_DIR)
endif()
set(cmake_dir ${espdl_dir}/fbs_loader/cmake)
include(${cmake_dir}/utilities.cmake)

# 按目标芯片选择对应的模型二进制
if (IDF_TARGET STREQUAL "esp32s3")
    set(embed_files models/s3/model.espdl)
elseif (IDF_TARGET STREQUAL "esp32p4")
    set(embed_files models/p4/model.espdl)
endif()

idf_component_register(SRCS ${srcs} REQUIRES ${requires})

# 关键：用对齐方式把 .espdl 放进 .rodata，保证 16 字节对齐
target_add_aligned_binary_data(${COMPONENT_LIB} ${embed_files} BINARY)
```

> `EMBED_FILES` 不保证 16 字节对齐，`.espdl` 必须使用 `target_add_aligned_binary_data`。

### 3. 设置目标并编译

```bash
idf.py set-target esp32s3   # 必须与量化时的 target 一致
idf.py menuconfig           # 可选：调整 ESP-DL 像素转换 Kconfig
idf.py build
idf.py flash monitor
```

### 4. 选择模型存放位置

| 方式 | 构造参数 | 适用场景 |
|---|---|---|
| rodata 嵌入 | `fbs::MODEL_LOCATION_IN_FLASH_RODATA` + 符号地址 | 最简单，模型随 app 一起烧录 |
| 分区 | `fbs::MODEL_LOCATION_IN_FLASH_PARTITION` + 分区 label | 模型可独立更新，推荐大模型 |
| SD 卡 | `fbs::MODEL_LOCATION_IN_SDCARD` + VFS 路径 | 频繁换模型、Flash 紧张 |

详见 `recipes/load_model_rodata.md`、`recipes/load_model_partition.md`、`recipes/load_model_sdcard.md`。

## 常见错误

| 错误 | 原因 | 解决方法 |
|---|---|---|
| `_binary_model_espdl_start` 找不到 | 文件名点号未转下划线 | rodata 符号名为 `_binary_` + 文件名(点→下划线) + `_start` |
| `.espdl` 加载报对齐错误 | 用了 `EMBED_FILES` | 改用 `target_add_aligned_binary_data` |
| 组件 target 名未识别 | 组件来自 registry 或本地 | CMake 里同时判断 `espressif__esp-dl` 与 `esp-dl` 两种前缀 |
| `target` 与芯片不符导致结果错误 | 量化 target 和 `set-target` 不一致 | 两者必须对应（`c`/`esp32s3`/`esp32p4`） |

## 参考

- `examples/tutorial/how_to_run_model/main/CMakeLists.txt`
- `docs/en/getting_started/readme.rst`
- `esp-dl/idf_component.yml`
