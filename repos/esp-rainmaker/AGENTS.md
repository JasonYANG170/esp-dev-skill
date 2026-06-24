# AGENTS.md — Supplementary Agent Guide

> 核心规则、场景映射、陷阱、执行工作流均在 `SKILL.md`。
> 本文件仅补充 `SKILL.md` 未涉及的约定与工具链指引，不重复内容。

## Project Context

**Language**: C · **Target**: ESP32 / ESP32-S2 / S3 / C2 / C3 / C5 / C6 / H2 · **Toolchain/Build**: ESP-IDF v5.1+（`idf.py`，GCC Xtensa/RISC-V）· **Component**: `esp_rainmaker`（`idf_component.yml` version 1.15.0）

## Code Generation Conventions

### File Naming
- 应用入口：`main/app_main.c`（所有官方示例统一）
- 驱动私有头：`main/app_priv.h`，驱动实现：`main/app_driver.c`
- 示例共用组件位于 `examples/common/`（`rmaker_app_network`、`rmaker_app_insights`、`rmaker_app_reset`、`gpio_button`、`ledc_driver` 等）

### Include Pattern
```c
/* 系统 */
#include <string.h>
#include <freertos/FreeRTOS.h>
#include <freertos/task.h>
#include <esp_log.h>
#include <esp_event.h>
#include <nvs_flash.h>

/* RainMaker Core + 标准 类型/参数/设备 */
#include <esp_rmaker_core.h>
#include <esp_rmaker_standard_types.h>
#include <esp_rmaker_standard_params.h>
#include <esp_rmaker_standard_devices.h>
#include <esp_rmaker_standard_services.h>
#include <esp_rmaker_console.h>
#include <esp_rmaker_ota.h>

/* 按需启用的服务 */
#include <esp_rmaker_schedule.h>      // 调度
#include <esp_rmaker_scenes.h>        // 场景
#include <esp_rmaker_user_mapping.h>  // 用户-节点映射
#include <esp_rmaker_mqtt.h>          // MQTT 直发
#include <esp_rmaker_connectivity.h>  // Connectivity 服务
#include <esp_rmaker_groups.h>        // Groups 服务
#include <esp_rmaker_controller.h>    // Controller 角色
#include <esp_rmaker_thread_br.h>     // Thread BR

/* 事件 */
#include <esp_rmaker_common_events.h>

/* 示例共用组件 */
#include <app_network.h>
#include <app_insights.h>

#include "app_priv.h"
```

### Standard Project Structure
```
my_rmaker_project/
├── main/
│   ├── CMakeLists.txt
│   ├── idf_component.yml          # 依赖 esp_rainmaker 等
│   ├── Kconfig.projbuild          # 示例自定义选项（如 DEFAULT_POWER）
│   ├── app_main.c                 # app_main() 入口
│   ├── app_driver.c               # 硬件驱动（GPIO/PWM/LED）
│   └── app_priv.h                 # 驱动私有声明、MFG_DATA 宏、DEFAULT_* 宏
├── sdkconfig.defaults             # 默认 Kconfig（含 CONFIG_ESP_RMAKER_*）
├── sdkconfig.defaults.<chip>      # 芯片特定覆盖
├── partitions.csv                 # 分区表（需含 nvs/factory/ota 区）
├── CMakeLists.txt                 # 顶层 cmake
└── README.md
```

### Canonical app_main() Pattern
```c
void app_main(void)
{
    esp_rmaker_console_init();
    app_driver_init();

    /* NVS */
    esp_err_t err = nvs_flash_init();
    if (err == ESP_ERR_NVS_NO_FREE_PAGES || err == ESP_ERR_NVS_NEW_VERSION_FOUND) {
        ESP_ERROR_CHECK(nvs_flash_erase());
        err = nvs_flash_init();
    }
    ESP_ERROR_CHECK(err);

    /* 网络（Wi-Fi/Thread）初始化，必须在 node_init 之前 */
    app_network_init();

    /* 事件 handler 注册 */
    ESP_ERROR_CHECK(esp_event_handler_register(RMAKER_EVENT, ESP_EVENT_ANY_ID, &event_handler, NULL));
    ESP_ERROR_CHECK(esp_event_handler_register(RMAKER_COMMON_EVENT, ESP_EVENT_ANY_ID, &event_handler, NULL));
    ESP_ERROR_CHECK(esp_event_handler_register(RMAKER_OTA_EVENT, ESP_EVENT_ANY_ID, &event_handler, NULL));

    /* 初始化节点 */
    esp_rmaker_config_t rainmaker_cfg = { .enable_time_sync = false };
    esp_rmaker_node_t *node = esp_rmaker_node_init(&rainmaker_cfg, "ESP RainMaker Device", "Switch");
    if (!node) { abort(); }

    /* 创建设备、加参数、加回调、挂载 */
    esp_rmaker_device_t *dev = esp_rmaker_switch_device_create("Switch", NULL, DEFAULT_POWER);
    esp_rmaker_device_add_cb(dev, write_cb, NULL);
    esp_rmaker_node_add_device(node, dev);

    /* 启用服务（必须在 esp_rmaker_start 之前） */
    esp_rmaker_ota_enable_default();
    esp_rmaker_timezone_service_enable();
    esp_rmaker_schedule_enable();
    esp_rmaker_scenes_enable();
    app_insights_enable();

    /* 启动 Agent */
    esp_rmaker_start();

    /* 启动网络连接（在 esp_rmaker_start 之后） */
    app_network_start(POP_TYPE_RANDOM);
}
```

### write 回调模板
```c
static esp_err_t write_cb(const esp_rmaker_device_t *device, const esp_rmaker_param_t *param,
            const esp_rmaker_param_val_t val, void *priv_data, esp_rmaker_write_ctx_t *ctx)
{
    if (ctx) {
        ESP_LOGI(TAG, "Received write request via : %s", esp_rmaker_device_cb_src_to_str(ctx->src));
    }
    const char *param_name = esp_rmaker_param_get_name(param);
    if (strcmp(param_name, ESP_RMAKER_DEF_POWER_NAME) == 0) {
        app_driver_set_state(val.val.b);
        esp_rmaker_param_update(param, val);
    }
    return ESP_OK;
}
```

### 日志约定
```c
static const char *TAG = "app_main";
ESP_LOGI(TAG, "RainMaker Initialised.");
```
默认日志走 UART0（`CONFIG_ESP_RMAKER_CONSOLE_UART_NUM_0`），可通过 `CONFIG_ESP_RMAKER_CONSOLE_UART_NUM_1` 切换。

## Build Workflow

1. 设置目标芯片：`idf.py set-target esp32s3`（或 esp32/esp32c3/esp32c6 等）
2. 配置（可选）：`idf.py menuconfig` → “ESP RainMaker Config”
3. 编译：`idf.py build`
4. 烧录监控：`idf.py flash monitor`
5. 首次烧录后按 Claiming 流程：Self Claim 自动执行；Assisted Claim 由手机 App 配网触发
6. 串口控制台命令：`esp_rmaker_console_init()` 注册后，可用 `get-config`、`set-param` 等（依赖对应 Kconfig）

## RainMaker 代码生成 Checklist

- [ ] `nvs_flash_init()` 处理了 `ESP_ERR_NVS_NO_FREE_PAGES`
- [ ] `app_network_init()` 在 `esp_rmaker_node_init()` 之前
- [ ] `esp_rmaker_node_init()` 是第一个调用的 RainMaker API
- [ ] 设备已 `esp_rmaker_node_add_device()` 挂到节点
- [ ] write 回调内调用了 `esp_rmaker_param_update()`/`_and_report()`
- [ ] 参数值用 `esp_rmaker_bool()/int()/float()/str()` 构造
- [ ] 所有 `*_enable()` 在 `esp_rmaker_start()` 之前
- [ ] `app_network_start()` 在 `esp_rmaker_start()` 之后
- [ ] 事件 base 用宏 `RMAKER_EVENT` / `RMAKER_COMMON_EVENT` / `RMAKER_OTA_EVENT`
- [ ] `priv_data` 用静态/堆内存，生命周期覆盖设备
- [ ] Claiming 类型与目标芯片匹配（ESP32/ESP32-C2 不能 Self Claim）
- [ ] Local Control 与 on-network chal_resp 未同时启用
- [ ] `partitions.csv` 含 ota 分区（如启用 OTA）
- [ ] `sdkconfig.defaults` 中 `CONFIG_ESP_RMAKER_*` 与需求一致

## Do Not Modify

- `components/esp_rainmaker/` — 仓库组件源码与头文件
- `examples/common/` — 示例共用组件（作为依赖引用，不要内联修改）
- `SKILL.md` front matter — Skill 元数据
