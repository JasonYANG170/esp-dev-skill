# AGENTS.md — Supplementary Agent Guide

> 核心规则、recipe 索引、陷阱清单、执行工作流均在 `SKILL.md`。
> 本文件仅补充 `SKILL.md` 未覆盖的工程约定与工具链指南，不重复内容。

## Project Context

**Language**: C · **Target**: ESP32-C5 / ESP32-C6（HP maincore + LP subcore）、ESP32-P4（HP maincore + HP subcore） · **Toolchain**: ESP-IDF（v5.3.1+ 用于 C6/P4，v5.5+ 用于 C5），RISC-V GCC，idf.py 构建系统 · **License**: Apache-2.0

## Code Generation Conventions

### File Naming
- 源文件 `*.c`，头文件 `*.h`
- maincore 入口：`maincore/main/app_main.c`（unified build 下 maincore 是被改名的 main 组件）
- subcore 入口：`subcore/main/main.c`（subcore **不支持** 改名 main 组件，必须保持为 `main`）
- 跨核共享头文件：放 `common/` 目录（如 `event.h`、`sys_info.h`、`rpc_service.h`），由两侧 `INCLUDE_DIRS "../common"` 引用
- subcore 自定义组件：加 `sub_` 前缀（如 `sub_greeting`、`sub_main_cfg`），避免与 maincore 同名组件冲突

### Include Pattern

```c
// maincore 与 subcore 通用（esp_amp.h 聚合所有组件头）
#include "esp_amp.h"
// esp_amp.h 等价于聚合：
//   esp_amp_sw_intr.h / esp_amp_sys_info.h / esp_amp_event.h
//   esp_amp_queue.h / esp_amp_rpmsg.h / esp_amp_rpc.h
//   esp_amp_env.h / esp_amp_platform.h
// maincore 额外包含 esp_amp_system.h（由 IS_MAIN_CORE 控制）

// maincore（FreeRTOS）典型
#include "freertos/FreeRTOS.h"
#include "freertos/task.h"
#include "freertos/event_groups.h"
#include "esp_log.h"
#include "esp_err.h"
#include "esp_partition.h"
#include "esp_amp.h"
#include "event.h"          // common/ 下跨核共享的事件/ID 定义

// subcore（bare-metal）典型
#include <stdint.h>
#include <stdio.h>
#include "esp_amp.h"
#include "esp_amp_platform.h"   // delay/intr 接口
#include "event.h"
```

### Standard Unified Build Project Structure

```
my_amp_project/
├── CMakeLists.txt                  # 顶层（maincore）工程 CMakeLists.txt
├── partitions.csv                  # 含 sub_core 分区条目
├── sdkconfig.defaults              # 顶层（maincore + subcore 共享）sdkconfig.defaults
├── sdkconfig.defaults.<target>     # 按目标覆盖
├── common/                         # 双核共享头（event.h / sys_info.h / rpc_service.h）
├── maincore/                       # 被改名的 main 组件
│   ├── CMakeLists.txt              # 调用 esp_amp_add_subcore_project()
│   └── main/
│       └── app_main.c
└── subcore/                        # subcore 工程（非 component）
    ├── CMakeLists.txt              # subcore 工程 CMakeLists.txt
    ├── subcore_config.cmake        # 定义 SUBCORE_APP_NAME / SUBCORE_PROJECT_DIR
    ├── main/                       # 必须名为 main
    │   ├── CMakeLists.txt
    │   └── main.c
    └── components/                 # subcore 自定义组件（sub_ 前缀）
        └── sub_main_cfg/
```

### Standard Separate Build Project Structure

```
my_amp_separate/
├── common/
├── maincore_project/
│   ├── CMakeLists.txt
│   ├── partitions.csv
│   ├── sdkconfig.defaults
│   └── main/
│       ├── CMakeLists.txt
│       └── app_main.c
└── subcore_project/
    ├── CMakeLists.txt
    ├── sdkconfig.defaults          # 注意：与 maincore 的 ESP-AMP 相关项必须一致
    └── main/
        ├── CMakeLists.txt
        └── main.c
```

### Canonical Entry / Init Pattern

**maincore（`app_main`，FreeRTOS，永不返回由 IDF 调度）：**
```c
void app_main(void)
{
    /* 1. 初始化 ESP-AMP（内部完成 SysInfo/SwIntr/Event 初始化） */
    assert(esp_amp_init() == 0);

    /* 2. 创建 IPC 资源（仅 maincore 能 create/alloc），例如 RPMsg */
    assert(esp_amp_rpmsg_main_init(&rpmsg_dev, 32, 128, true /*notify*/, false /*poll*/) == 0);
    esp_amp_rpmsg_intr_enable(&rpmsg_dev);   // poll=false 时必须调用
    esp_amp_rpmsg_create_endpoint(&rpmsg_dev, 0, ept_cb, arg, &rpmsg_ept);

    /* 3. 加载并启动 subcore */
    const esp_partition_t *p = esp_partition_find_first(ESP_PARTITION_TYPE_DATA, 0x40, NULL);
    ESP_ERROR_CHECK(esp_amp_load_sub_from_partition(p));
    ESP_ERROR_CHECK(esp_amp_start_subcore());

    /* 4. 握手：等待 subcore 就绪 */
    assert((esp_amp_event_wait(EVENT_SUBCORE_READY, true, true, 10000)
            & EVENT_SUBCORE_READY) == EVENT_SUBCORE_READY);

    /* 5. 创建业务任务，进入事件循环 */
    xTaskCreate(my_task, "t", 2048, NULL, tskIDLE_PRIORITY, NULL);
}
```

**subcore（`main`，bare-metal，主循环不退出）：**
```c
int main(void)
{
    printf("SUB: Hello!!\r\n");

    /* 1. 初始化 ESP-AMP（subcore 侧） */
    assert(esp_amp_init() == 0);

    /* 2. 初始化 IPC（subcore 侧），参数 notify/poll 与对端互补 */
    assert(esp_amp_rpmsg_sub_init(&rpmsg_dev, true /*notify*/, true /*poll*/) == 0);
    esp_amp_rpmsg_create_endpoint(&rpmsg_dev, 0, ept_cb, NULL, &rpmsg_ept);

    /* 3. 通知 maincore 就绪 */
    esp_amp_event_notify(EVENT_SUBCORE_READY);

    /* 4. 主循环（轮询模式下需主动 poll） */
    for (;;) {
        while (esp_amp_rpmsg_poll(&rpmsg_dev) == 0);   // poll 模式才需要
        esp_amp_platform_delay_us(1000000);
    }
}
```

### EventGroup / ISR-Safe Logging Conventions

```c
// maincore 中断处理函数（含 RPMsg endpoint 回调、sw_intr handler）
static const DRAM_ATTR char TAG[] = "xxx";          // TAG 放 DRAM
static IRAM_ATTR int handler(void *arg) {
    ESP_DRAM_LOGI(TAG, "called");                   // ISR 内必须用 ESP_DRAM_LOGx
    BaseType_t need_yield = pdFALSE;
    /* ... 从 ISR 唤醒任务：xQueueSendFromISR / vTaskNotifyGiveFromISR ... */
    return (need_yield == pdTRUE);                   // maincore 返回值含义：是否需要切换
}

// subcore 中断处理函数（bare-metal）
static int handler(void *arg) {                      // 无需 IRAM_ATTR（已在内部 RAM）
    printf("called\r\n");                            // printf 可用（经 esp_amp_putchar 路由）
    return 0;                                        // 返回值被忽略
}
```

Debug UART：subcore 默认经 LP UART（LP subcore）或 UART1（HP subcore）；启用 supplicant 后路由到 maincore 控制台。

## Build Workflow

### Unified Build（推荐入门）
```shell
source $IDF_PATH/export.sh
cd my_amp_project
idf.py set-target esp32c6          # 或 esp32c5 / esp32p4
idf.py build                        # 同时构建 maincore + subcore
idf.py flash monitor
```
- subcore 固件位于 `build/subcore/${SUBCORE_APP_NAME}.bin`
- 嵌入模式：固件链接进 maincore 的 `.rodata.embedded`；分区模式：自动随 `idf.py flash` 烧入分区

### Separate Build（subcore 独立开发）
```shell
# 1. maincore
cd maincore_project
idf.py set-target esp32c6 && idf.py build && idf.py flash

# 2. subcore
cd ../subcore_project
idf.py set-target esp32c6 && idf.py build
# 手动烧录到 partitions.csv 中 sub_core 分区的 offset（如 0x200000）
esptool.py write_flash 0x200000 build/subcore_xxx.bin
```
- 两侧 sdkconfig 的 ESP-AMP 相关项（共享内存大小、SysInfo/Event 设置）**必须手动保持一致**

### 关键 CMake 片段

顶层 `CMakeLists.txt`（unified build，必须 `include(subcore_config.cmake)` 在 `project()` 之前）：
```cmake
include(${CMAKE_CURRENT_LIST_DIR}/subcore/subcore_config.cmake)
set(EXTRA_COMPONENT_DIRS
    ${ESP_AMP_PATH}/components
    ${CMAKE_CURRENT_LIST_DIR}/maincore
    ${SUBCORE_COMPONENT_DIRS}
)
project(my_amp_project)
```

maincore `CMakeLists.txt`：
```cmake
idf_component_register(SRCS main/app_main.c INCLUDE_DIRS "../common" REQUIRES esp_amp)
esp_amp_add_subcore_project(${SUBCORE_APP_NAME} ${SUBCORE_PROJECT_DIR} PARTITION TYPE data SUBTYPE 0x40)
```

`subcore_config.cmake`：
```cmake
set(app_name subcore_my_app)
idf_build_set_property(SUBCORE_APP_NAME "${app_name}" APPEND)
get_filename_component(directory "${CMAKE_CURRENT_LIST_DIR}" ABSOLUTE DIRECTORY)
idf_build_set_property(SUBCORE_PROJECT_DIR "${directory}" APPEND)
list(APPEND SUBCORE_COMPONENT_DIRS "${CMAKE_CURRENT_LIST_DIR}/components")
```

subcore component（unified build 下的 sdkconfig 触发守卫）：
```cmake
if(NOT SUBCORE_BUILD)
    idf_component_register()   # 仅为生成 sdkconfig，不在 maincore 工程编译
    return()
endif()
idf_component_register(SRCS "greeting.c" INCLUDE_DIRS ".")
```

## Code Generation Checklist

### maincore
- [ ] `esp_amp_init()` 在所有 ESP-AMP API 之前调用
- [ ] 所有 SysInfo 分配 / RPMsg main_init / Event create 在 `esp_amp_start_subcore()` 之前完成
- [ ] `esp_amp_start_subcore()` 后 `esp_amp_event_wait(EVENT_SUBCORE_READY, true, true, 10000)` 握手
- [ ] FreeRTOS 等待的 ESP-AMP event 已 `esp_amp_event_bind_handle()` 绑定 EventGroup
- [ ] subcore 分区条目（`partitions.csv`）type/subtype 与 `esp_amp_add_subcore_project(... TYPE data SUBTYPE 0x40)` 一致
- [ ] 中断处理函数（sw_intr handler、RPMsg endpoint 回调）加 `IRAM_ATTR`，TAG 放 `DRAM_ATTR`，日志用 `ESP_DRAM_LOGx`
- [ ] 嵌入固件：确认 unified build + `CONFIG_*_EMBEDDED` + 符号 `_binary_${SUBCORE_APP_NAME}_bin_start`

### subcore
- [ ] `main` 组件名保持为 `main`（不可改名）
- [ ] `esp_amp_init()` → IPC 初始化 → `esp_amp_event_notify(EVENT_SUBCORE_READY)` 顺序
- [ ] notify/poll 参数与 maincore 互补（一端 poll=false 则对端 notify=true）
- [ ] RPMsg/Virtqueue 接收的缓冲使用后调用 `esp_amp_rpmsg_destroy()` / `esp_amp_queue_free_try()`
- [ ] 主循环不退出（bare-metal 永不返回）
- [ ] 避免浮点打印；LP subcore 避免浮点运算
- [ ] 需要 malloc 时启用 `CONFIG_ESP_AMP_SUBCORE_ENABLE_HEAP=y`
- [ ] light sleep 模式下：ISR 用 `RTC_IRAM_ATTR`，访问 HP RAM/外设用 `ESP_AMP_PM_SKIP_LIGHT_SLEEP_ENTER/EXIT` 成对包裹

## Do Not Modify

- `espressif-repos/esp-amp/components/esp_amp/` — ESP-AMP 组件源码（`include/`、`src/`、`system/`、`port/`、`Kconfig`、`cmake/`、`idf_stub/`）
- `espressif-repos/esp-amp/docs/` — 原始文档（本 skill 的引用来源）
- `SKILL.md` front matter — Skill 元数据（name / description / tags / license / compatibility / metadata.version）
- 用户既有工程的 `sdkconfig`（除非用户明确要求修改；修改 sdkconfig.defaults 后需 `idf.py reconfigure`）
