# AGENTS.md — Supplementary Agent Guide

> 核心规则、芯片支持矩阵、调用链、陷阱、recipe 索引与执行流程均见 `SKILL.md`。
> 本文件只补充 `SKILL.md` 未覆盖的约定与工具用法，不重复内容。

## Project Context

**语言**: C（含可选 C++，组件 `cxx`） · **目标**: Espressif ESP32 / ESP32-S2/S3 / ESP32-C2/C3/C5/C6/C61 / ESP32-H2/H4/H21 / ESP32-P4 SoC · **工具链/构建**: ESP-IDF 工具链（GCC for Xtensa / RISC-V），CMake + Ninja，统一入口 `idf.py`。运行于 Windows / Linux / macOS。

## 工程目录结构（标准）

```
my_project/
├── CMakeLists.txt          # 顶层：cmake_minimum_required + include(project.cmake) + project(name)
├── sdkconfig               # 由 menuconfig 生成，勿手改（改用 sdkconfig.defaults）
├── sdkconfig.defaults      # 可选：默认配置覆盖
├── partitions.csv          # 可选：自定义分区表（配合 CONFIG_PARTITION_TABLE_CUSTOM=y）
├── idf_component.yml       # 可选：外部组件依赖（从 ESP Component Registry 拉取）
├── main/
│   ├── CMakeLists.txt      # idf_component_register(...)
│   ├── idf_component.yml   # 可选：main 组件依赖
│   ├── Kconfig.projbuild   # 可选：在 menuconfig 顶层加菜单项
│   └── <app>_main.c        # void app_main(void)
└── components/             # 可选：本地私有组件，每个子目录一个组件
    └── my_driver/
        ├── CMakeLists.txt
        ├── include/my_driver.h
        └── my_driver.c
```

## 顶层 CMakeLists 固定顺序

```cmake
# 必须严格按此顺序
cmake_minimum_required(VERSION 3.22)

include($ENV{IDF_PATH}/tools/cmake/project.cmake)
project(my_project)
```

## 组件注册（main/CMakeLists.txt）

```cmake
idf_component_register(SRCS "app_main.c" "net.c"
                       INCLUDE_DIRS "."
                       REQUIRES nvs_flash          # 公共依赖（被其他组件间接看到）
                       PRIV_REQUIRES esp_wifi       # 仅本组件私有依赖
                                  esp_event
                                  esp_driver_gpio
                                  esp_driver_uart)
```

> 关键词：`REQUIRES`（公共传递）、`PRIV_REQUIRES`（私有，不传递）、`INCLUDE_DIRS`（对外暴露的头文件目录）。

## Include 模式

```c
/* 系统与日志 */
#include <stdio.h>
#include "sdkconfig.h"
#include "esp_log.h"
#include "esp_err.h"
#include "esp_system.h"

/* FreeRTOS */
#include "freertos/FreeRTOS.h"
#include "freertos/task.h"
#include "freertos/queue.h"
#include "freertos/event_groups.h"

/* 网络 */
#include "esp_wifi.h"
#include "esp_event.h"
#include "esp_netif.h"
#include "nvs_flash.h"

/* 外设（v5.x 新 HAL，句柄式） */
#include "driver/gpio.h"
#include "driver/uart.h"
#include "driver/i2c_master.h"
#include "driver/spi_master.h"
#include "driver/ledc.h"
#include "driver/gptimer.h"
#include "esp_adc/adc_oneshot.h"

/* 存储 / OTA */
#include "nvs.h"
#include "esp_partition.h"
#include "esp_ota_ops.h"
#include "esp_https_ota.h"
#include "esp_flash.h"
#include "spi_flash_mmap.h"
```

## 标准入口（app_main）模板

```c
static const char *TAG = "app";

void app_main(void)
{
    /* 1. NVS（绝大多数应用都需要） */
    esp_err_t ret = nvs_flash_init();
    if (ret == ESP_ERR_NVS_NO_FREE_PAGES || ret == ESP_ERR_NVS_NEW_VERSION_FOUND) {
        ESP_ERROR_CHECK(nvs_flash_erase());
        ret = nvs_flash_init();
    }
    ESP_ERROR_CHECK(ret);

    ESP_LOGI(TAG, "app start, free heap=%" PRIu32, esp_get_free_heap_size());

    /* 2. 网络：先 netif + 事件循环，再 wifi */
    ESP_ERROR_CHECK(esp_netif_init());
    ESP_ERROR_CHECK(esp_event_loop_create_default());

    /* 3. 业务任务 */
    xTaskCreate(main_task, "main_task", 4096, NULL, 5, NULL);

    /* app_main 可返回；如需主线程常驻用 while(1)+vTaskDelay */
}
```

## 日志约定

```c
static const char *TAG = "module";
ESP_LOGE(TAG, "error %s", esp_err_to_name(err));   // ERROR，红色
ESP_LOGW(TAG, "warn %d", n);
ESP_LOGI(TAG, "info");
ESP_LOGD(TAG, "debug");   // 默认被裁剪，需提级
ESP_LOGV(TAG, "verbose");
```
- 等级受 `CONFIG_LOG_DEFAULT_LEVEL`（编译期）与 `esp_log_level_set(tag, level)`（运行期）控制。
- tag 建议每个 `.c` 一个 `static const char *TAG`，便于过滤。

## ISR 中断处理模板（GPIO）

```c
static void IRAM_ATTR gpio_isr_handler(void *arg)
{
    BaseType_t hpw = pdFALSE;
    uint32_t gpio_num = (uint32_t)arg;
    xQueueSendFromISR(gpio_evt_queue, &gpio_num, &hpw);
    portYIELD_FROM_ISR(hpw);
}
```
- ISR 函数须标记 `IRAM_ATTR`；只允许调用 `...FromISR` 系列 API；禁止任何阻塞。

## 构建工作流

1. 激活 ESP-IDF 环境：Windows `export.bat`（或 EIM），Linux/macOS `source export.sh`。
2. `cd` 到工程根目录（**路径不得含空格**）。
3. `idf.py set-target <chip>`（首次或换芯片）。
4. `idf.py menuconfig`（按需，可跳过）。
5. `idf.py build`（生成 `build/`，含 bootloader、分区表、镜像）。
6. `idf.py -p PORT flash monitor`（PORT 形如 `COM3`/`/dev/ttyUSB0`/`/dev/cu.usbserial-*`）。
7. `idf.py erase-flash`（改分区表/OTA 后整片擦除）。
8. `idf.py fullclean`（彻底清构建）。

## 代码生成 Checklist

- [ ] 顶层 `CMakeLists.txt` 三行顺序正确（`cmake_minimum_required` → `include(project.cmake)` → `project()`）
- [ ] `main/CMakeLists.txt` 用 `idf_component_register`，`SRCS`/`INCLUDE_DIRS`/`REQUIRES`/`PRIV_REQUIRES` 完整
- [ ] 外设依赖加入对应 `esp_driver_*` 组件
- [ ] 已用 `idf.py set-target <chip>` 设定目标，`sdkconfig` 与芯片一致
- [ ] 入口为 `void app_main(void)`；长驻逻辑用任务或 `while(1)`
- [ ] NVS 初始化处理 `ESP_ERR_NVS_NO_FREE_PAGES` / `ESP_ERR_NVS_NEW_VERSION_FOUND`
- [ ] Wi-Fi 前：`esp_netif_init()` + `esp_event_loop_create_default()` + `esp_netif_create_default_wifi_sta/ap()`
- [ ] 每个 `esp_err_t` 返回值都被检查（`ESP_ERROR_CHECK` 或显式判断 + `esp_err_to_name`）
- [ ] ISR 标记 `IRAM_ATTR`，内部仅用 `...FromISR` API
- [ ] OTA 工程含 factory + ota_0 + ota_1 + otadata 分区，且 `CONFIG_PARTITION_TABLE_*` 指向自定义 CSV（如需）
- [ ] 日志用 `ESP_LOG*` + 静态 `TAG`；无裸 `printf`（除非重定向）
- [ ] 引脚选择参考目标 SoC datasheet，避免 strapping/Flash 引脚冲突

## Do Not Modify

- ESP-IDF 仓库源码 `D:/esp-skill/espressif-repos/esp-idf/components/`、`tools/` —— 仅作参考与复制样例来源，不要就地修改。
- 本 skill 的 `resources/` —— API 文档来源，保持与仓库一致。
- `SKILL.md` front matter —— 技能元数据，勿擅改。
