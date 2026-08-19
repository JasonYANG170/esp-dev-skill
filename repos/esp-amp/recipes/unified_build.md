# Unified Build：单命令构建 maincore + subcore

> **适用摘要**: 使用 ESP-AMP unified build 模式，通过一条 `idf.py build` 同时构建 maincore 与 subcore 固件，支持将 subcore 固件嵌入 maincore 或烧入 flash 分区。适合 ESP-AMP 入门与协作开发场景。

> Version: selected repo/component version; align with the user project and dependency manifest.
> Evidence: `repos/esp-amp/resources/`, source/examples in `repos/esp-amp/`, and this recipe path `repos/esp-amp/recipes/unified_build.md`.
> Validation: draft metadata added from repository routing; verify APIs, Kconfig symbols, and component dependencies against the selected version.

## 触发意图

- "构建 ESP-AMP 工程"
- "unified build"
- "一条命令构建双核固件"
- "把 subcore 固件嵌入 maincore"
- "subcore 固件 OTA"

## 前置条件

| 条件 | 要求 |
|---|---|
| 参考工程 | `examples/rpmsg_send_recv/`、`examples/build_system/unified_build/` |
| ESP-IDF | v5.3.1+（C6/P4）或 v5.5+（C5），已 `export.sh` |
| 环境变量 | 可选 `ESP_AMP_PATH` 指向 esp-amp 仓库根；不设则默认 `CMAKE_CURRENT_LIST_DIR/../..` |

## 分步说明

### 1. 工程目录结构

```
my_amp_project/
├── CMakeLists.txt
├── partitions.csv
├── sdkconfig.defaults
├── common/event.h
├── maincore/
│   ├── CMakeLists.txt
│   └── main/app_main.c
└── subcore/
    ├── CMakeLists.txt
    ├── subcore_config.cmake
    ├── main/
    │   ├── CMakeLists.txt
    │   └── main.c
    └── components/sub_main_cfg/
```

### 2. 顶层 CMakeLists.txt（关键：include subcore_config.cmake 在 project() 之前）

```cmake
cmake_minimum_required(VERSION 3.16)
list(APPEND SDKCONFIG_DEFAULTS "sdkconfig.defaults")

if(DEFINED ENV{ESP_AMP_PATH})
  set(ESP_AMP_PATH $ENV{ESP_AMP_PATH})
else()
  set(ESP_AMP_PATH ${CMAKE_CURRENT_LIST_DIR}/../..)
endif()

include($ENV{IDF_PATH}/tools/cmake/project.cmake)

# 必须在 project() 之前 include，使 SUBCORE_APP_NAME / SUBCORE_PROJECT_DIR 全局可用
include(${CMAKE_CURRENT_LIST_DIR}/subcore/subcore_config.cmake)

set(EXTRA_COMPONENT_DIRS
    ${ESP_AMP_PATH}/components
    ${CMAKE_CURRENT_LIST_DIR}/maincore
    ${SUBCORE_COMPONENT_DIRS}
)

project(my_amp_project)
```

### 3. subcore_config.cmake（定义 subcore app 名与路径）

```cmake
# subcore app 名（决定生成的 bin 与嵌入符号名 _binary_<app_name>_bin_start）
set(app_name subcore_my_app)
idf_build_set_property(SUBCORE_APP_NAME "${app_name}" APPEND)

# subcore 工程目录
get_filename_component(directory "${CMAKE_CURRENT_LIST_DIR}" ABSOLUTE DIRECTORY)
idf_build_set_property(SUBCORE_PROJECT_DIR "${directory}" APPEND)

# subcore 自定义组件目录
list(APPEND SUBCORE_COMPONENT_DIRS "${CMAKE_CURRENT_LIST_DIR}/components")
```

### 4. maincore/CMakeLists.txt（调用 esp_amp_add_subcore_project）

```cmake
idf_component_register(
    SRCS main/app_main.c
    INCLUDE_DIRS "../common"
    REQUIRES esp_amp
)

# 通过 sdkconfig 切换嵌入 / 分区存储
if(CONFIG_SUBCORE_FIRMWARE_EMBEDDED)
    esp_amp_add_subcore_project(${SUBCORE_APP_NAME} ${SUBCORE_PROJECT_DIR} EMBED)
else()
    esp_amp_add_subcore_project(${SUBCORE_APP_NAME} ${SUBCORE_PROJECT_DIR} PARTITION TYPE data SUBTYPE 0x40)
endif()
```

### 5. partitions.csv（sub_core 分区条目，分区模式必需）

```csv
# Name,     Type,       SubType,    Offset,     Size,   Flags
nvs,        data,       nvs,        0x9000,     24K,
phy_init,   data,       phy,        0xf000,     4K,
factory,    app,        factory,    0x10000,    1M,
sub_core,   data,       0x40,       0x200000,   64K,
```
> type `data` subtype `0x40` 必须与 `esp_amp_add_subcore_project(... TYPE data SUBTYPE 0x40)` 一致。

### 6. 构建与烧录

```shell
idf.py set-target esp32c6      # 或 esp32c5 / esp32p4
idf.py build                    # 同时构建 maincore + subcore
idf.py flash monitor
```
- maincore 固件：`build/my_amp_project.bin`
- subcore 固件：`build/subcore/subcore_my_app.bin`

### 7. 嵌入模式的加载代码（maincore）

```c
extern const uint8_t subcore_my_app_bin_start[] asm("_binary_subcore_my_app_bin_start");
extern const uint8_t subcore_my_app_bin_end[]   asm("_binary_subcore_my_app_bin_end");
ESP_ERROR_CHECK(esp_amp_load_sub(subcore_my_app_bin_start));
```

### 8. 分区模式的加载代码（maincore）

```c
const esp_partition_t *p = esp_partition_find_first(ESP_PARTITION_TYPE_DATA, 0x40, NULL);
ESP_ERROR_CHECK(esp_amp_load_sub_from_partition(p));
```

## 常见错误

| 错误 | 原因 | 解决方法 |
|---|---|---|
| `_binary_xxx_bin_start not found` | 未用 unified build ��用 EMBED；或 app_name 与符号名不匹配 | 确认 unified build；符号名 = `_binary_${SUBCORE_APP_NAME}_bin_start`；查 `build/*.bin.S` 与 map 文件 `.rodata.embedded` |
| subcore 固件未烧入分区 | 分区模式下 type/subtype 与 `partitions.csv` 不一致 | 对齐 `TYPE data SUBTYPE 0x40` 与分区表 |
| 顶层 CMake 报 SUBCORE_APP_NAME 未定义 | `subcore_config.cmake` 在 `project()` 之后才 include | 必须在 `project()` 之前 include |
| subcore 组件覆盖 maincore 同名组件 | 未加 `sub_` 前缀 | subcore 组件统一加 `sub_` 前缀 |
| unified build 下 subcore 组件被 maincore 编译 | 缺守卫 | subcore 组件 `CMakeLists.txt` 加 `if(NOT SUBCORE_BUILD) idf_component_register() return() endif()` |

## 参考

- `examples/build_system/unified_build/` — unified build 完整示例
- `examples/rpmsg_send_recv/` — 含完整 unified build 结构与 `subcore_config.cmake`
- `espressif-repos/esp-amp/docs/build_system.md`
