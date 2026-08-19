# 创建与定制新产品

> **适用摘要**: 基于模板或最接近的现有产品创建一个新 LowCode 产品，正确组织目录结构、声明组件依赖。

> Version: selected repo/component version; align with the user project and dependency manifest.
> Evidence: `repos/esp-lowcode-matter/resources/`, source/examples in `repos/esp-lowcode-matter/`, and this recipe path `repos/esp-lowcode-matter/recipes/create_product.md`.
> Validation: draft metadata added from repository routing; verify APIs, Kconfig symbols, and component dependencies against the selected version.

## 触发意图

- "创建新 lowcode 产品"
- "新建 matter 产品"
- "LowCode: Create Product"
- "怎么加一个新设备类型"

## 前置条件

| 条件 | 要求 |
|---|---|
| 环境 | 已按 `recipes/getting_started.md` 完成环境搭建 |
| 参考产品 | `products/template/` 或最接近的现有 product |

## 分步说明

### 1. 选参考产品

按设备类型选最接近的（见 `resources/example_list.md`）：灯→`light_cw_pwm`/`light_rgbcw_ws2812`；插座→`socket`/`socket_2_channel`；传感器→`temperature_sensor`；占用→`occupancy_sensor`；温控→`thermostat`；无对应→`template`。

### 2. 复制目录

```sh
cd $LOW_CODE_PATH/products
cp -r template my_product
```

保留整体结构：`configuration/`、`main/`、`CMakeLists.txt`、`sdkconfig.defaults`、`README.md`。

### 3. 编辑 `main/CMakeLists.txt` 声明依赖

模板默认：

```cmake
idf_component_register(SRC_DIRS .
                        INCLUDE_DIRS .
                        REQUIRES low_code system)
```

用到更多组件时补齐（例如温度传感器产品）：

```cmake
idf_component_register(SRC_DIRS .
                        INCLUDE_DIRS .
                        REQUIRES low_code system button light temperature_sensor_sht30)
```

### 4. 编辑 `app_priv.h` 暴露驱动函数

```c
#pragma once
#include <stdint.h>
#include <low_code.h>

/* 驱动函数 */
int app_driver_init(void);
int app_driver_set_socket_state(bool state);   // 按需命名

/* 系统回调 */
int feature_update_from_system(low_code_feature_data_t *data);
int event_from_system(low_code_event_t *event);
int app_driver_event_handler(low_code_event_t *event);
```

### 5. 编辑 `configuration/product_info.json`

```json
{
    "config_version": 3,
    "vendor_id": 65521,
    "product_id": 32768,
    "origin_vendor_id": 65521,
    "origin_product_id": 32768,
    "device_type_id": 266,
    "vendor_name": "Espressif",
    "product_name": "Matter Product",
    "hw_ver": 1,
    "hw_ver_str": "1",
    "chip": "esp32c6",
    "connection_type": "wifi",
    "module": "ESP32-C6-MINI-1",
    "flash_size": "4MB",
    "secure_boot": "enabled",
    "product_type": "socket",
    "solution_type": "low_code"
}
```

字段含义见 `docs/product_configuration.md`。`chip` 当前必须为 `esp32c6`，`connection_type` 可为 `wifi` 或 `thread`（对应 `data_model_wifi.zap` / `data_model_thread.zap`）。

### 6. 定制数据模型（可选）

编辑 `configuration/data_model_wifi.zap`（用 [ZAP 编辑器](https://github.com/project-chip/zap)）：
- endpoint 0 必须为 root node，应用节点放其它 endpoint
- 启用所需 cluster/attribute 并更新 feature_map
- 改完务必重跑「Upload Configuration」重新生成 `data_model.bin`
- 新增内容会增加内存占用，需测试

### 7. 实现 `app_main.cpp` / `app_driver.cpp`

按 `recipes/setup_loop_model.md` 的骨架编写；新设备类型在 `feature_update_from_system` 里按 endpoint+feature_id 分发。

### 8. 走构建/烧录流程

见 `recipes/getting_started.md`（`export SELECTED_PRODUCT=my_product` 后执行 Prepare Device / Upload Configuration / Upload Code）。

## 常见错误

| 错误 | 原因 | 解决方法 |
|---|---|---|
| 新增组件符号未定义 | REQUIRES 未声明 | 在 `main/CMakeLists.txt` 的 REQUIRES 加组件名 |
| zap 改了但设备行为没变 | 未重跑 Upload Configuration | 重新生成并烧录 data_model.bin |
| 新增 endpoint 后接收不到 | app 未按新 endpoint 分发 | 在 `feature_update_from_system` 加 endpoint 判断 |
| 内存不足/OOM | 数据模型过大 | 精简 cluster/attribute，测试内存占用 |

## 参考

- `docs/create_product.md`
- `docs/product_configuration.md`
- `products/template/README.md`
- `recipes/setup_loop_model.md`
