# 从 SD 卡加载模型

> **适用摘要**: 把 `.espdl` 放到 FAT32 SD 卡，用 `MODEL_LOCATION_IN_SDCARD` 加载。适合 Flash 紧张或需要频繁换模型的场景。

> Version: selected repo/component version; align with the user project and dependency manifest.
> Evidence: `repos/esp-dl/resources/`, source/examples in `repos/esp-dl/`, and this recipe path `repos/esp-dl/recipes/load_model_sdcard.md`.
> Validation: draft metadata added from repository routing; verify APIs, Kconfig symbols, and component dependencies against the selected version.

## 触发意图

- "模型放 SD 卡"
- "动态切换模型"
- "MODEL_LOCATION_IN_SDCARD"

## 前置条件

| 条件 | 要求 |
|---|---|
| SD 卡 | FAT32 格式（否则挂载时会自动格式化，数据丢失） |
| 参考示例 | `examples/tutorial/how_to_load_test_profile_model/model_in_sdcard`、`examples/yolo11_detect`（`CONFIG_*_MODEL_IN_SDCARD`） |

## 分步说明

### 1. 挂载 SD 卡（使用 BSP）

```cpp
#include "bsp/esp-bsp.h"

ESP_ERROR_CHECK(bsp_sdcard_mount());
```

menuconfig 中可启用 `CONFIG_BSP_SD_FORMAT_ON_MOUNT_FAIL` 以便挂载失败时自动格式化。

### 2. 或不使用 BSP 手动挂载

```cpp
#include "esp_vfs_fat.h"
#include "sdmmc_cmd.h"

esp_vfs_fat_sdmmc_mount_config_t mount_config = {
    .format_if_mount_failed = true,
    .max_files = 5,
    .allocation_unit_size = 16 * 1024,
};
// 之后按你的硬件调用 esp_vfs_fat_sdmmc_mount / sdspi ...
```

### 3. 把 `.espdl` 拷到 SD 卡根目录（例如 `/sdcard/model.espdl`）

### 4. 加载并推理

```cpp
#include "dl_model_base.hpp"

extern "C" void app_main(void)
{
    ESP_ERROR_CHECK(bsp_sdcard_mount());

    // 第一个参数是 VFS 路径
    dl::Model *model = new dl::Model("/sdcard/model.espdl", fbs::MODEL_LOCATION_IN_SDCARD);

    ESP_ERROR_CHECK(model->test());
    model->profile();
    delete model;

    ESP_ERROR_CHECK(bsp_sdcard_unmount());
}
```

> 从 SD 卡加��比从 FLASH 慢（模型数据需拷到 RAM），但能省 Flash 空间、方便换模型。

## 常见错误

| 错误 | 原因 | 解决方法 |
|---|---|---|
| 挂载失败 / 数据丢失 | 非 FAT32 | 提前备份并格式化为 FAT32，或开启自动格式化 |
| 路径找不到 | VFS 路径写错 | 确认挂载点前缀（BSP 默认 `/sdcard`）与文件名 |
| 加载慢 | SD 卡读取慢 | SD 卡加载本身较慢，属正常；可换更快卡或回到分区方式 |

## 参考

- `docs/en/tutorials/how_to_load_test_profile_model.rst`（Load model from sdcard 一节）
- `examples/yolo11_detect/main/app_main.cpp`（`CONFIG_COCO_DETECT_MODEL_IN_SDCARD` 分支）
