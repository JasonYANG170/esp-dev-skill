# AGENTS.md — Supplementary Agent Guide

> 核心规则、映射表、陷阱、Recipes 索引、执行工作流均在 `SKILL.md`。
> 本文件**只**补充 `SKILL.md` 未覆盖的工程约定与工具链指引，不重复内容。

## Project Context

**Language**: C (C11) · **Target**: ESP32-H2 / ESP32-C6（native 802.15.4），或 ESP32-C3/S3/P4 + ESP32-H2/C6 RCP · **Toolchain**: ESP-IDF v5.2+（示例对齐 v5.5.4），`idf.py`，RISC-V/Xtensa GCC · **Protocol stack**: 乐鑫自研 Zigbee（预编译库 `espressif/esp-zigbee-lib`，v2.x）

## API 分层（务必区分）

| 层 | 前缀 / 路径 | 状态 |
|---|---|---|
| v2.x 主线（活动） | `ezb_*`，头文件 `components/esp-zigbee-lib/include/ezbee/` | **推荐**，新代码用这层 |
| v1.x 兼容层 | `esp_zb_*`，头文件 `components/esp-zigbee-lib/include/compat/` | 仅迁移/老项目，参考 `docs/en/migration-guide/v2.x/` |

> 写新代码默认使用 `ezb_` API。仓库 `docs/en/developing.rst`、所有 `examples/` 主线代码均用 `ezb_`。

## Code Generation Conventions

### File Naming
- 应用主文件：`main/<device_name>.c` + `main/<device_name>.h`（如 `on_off_light.c/.h`）
- 示例私有配置宏集中在 `main/<device_name>.h`（`ESP_ZIGBEE_*_CONFIG()`、信道掩码、endpoint ID、厂商/型号字符串）
- 公共驱动位于 `examples/utils/<driver>/`（`light_driver`、`switch_driver`、`temp_sensor_driver`、`alarm_timer`、`example_common`、`delta_ota`）

### Include Pattern（v2.x）
```c
#include "esp_check.h"
#include "esp_err.h"
#include "esp_log.h"
#include "nvs_flash.h"

#include "esp_zigbee.h"     // 平台层：esp_zigbee_init/start/launch_mainloop/lock
#include "ezbee/zha.h"      // ZHA 设备创建：ezb_zha_create_on_off_light 等
// 按需引入更多 ezbee/ 头：bdb.h, af.h, zcl.h, zdo.h, secur.h, nwk.h, touchlink.h

#include "<device_name>.h"  // 本示例私有配置宏
```

### Standard Project Structure
```
my_zigbee_device/
├── CMakeLists.txt
├── partitions.csv              # 必须含 zb_storage(16K,nvs) 与 zb_fct(1K,fat)
├── sdkconfig.defaults          # 例：CONFIG_PM_ENABLE / GPIO 引脚 / RCP 选项
├── main/
│   ├── CMakeLists.txt
│   ├── idf_component.yml       # 声明 espressif/esp-zigbee-lib >=2.0.0, idf >=5.2
│   ├── <device>.h              # ESP_ZIGBEE_*_CONFIG()、信道、EP_ID、厂商字符串
│   └── <device>.c              # app_main + zigbee task + signal handler + ZCL handler
└── utils/                      # 按需软链或复制 examples/utils/<driver>
```

### Canonical Main / Zigbee Task Pattern（来自 on_off_light 示例）
```c
void app_main(void)
{
    ESP_ERROR_CHECK(nvs_flash_init());
    ESP_ERROR_CHECK(nvs_flash_init_partition(ESP_ZIGBEE_STORAGE_PARTITION_NAME));
    ESP_LOGI(TAG, "Start ESP Zigbee Stack");
    xTaskCreate(esp_zigbee_stack_main_task, "Zigbee_main", 4096, NULL, 5, NULL);
}

static void esp_zigbee_stack_main_task(void *pvParameters)
{
    esp_zigbee_config_t config = ESP_ZIGBEE_DEFAULT_CONFIG();
    ESP_ERROR_CHECK(esp_zigbee_init(&config));
    ESP_ERROR_CHECK(esp_zigbee_setup_commissioning());          // 信道 + signal handler
    ESP_ERROR_CHECK(esp_zigbee_create_zha_on_off_light_device()); // 数据模型
    ESP_ERROR_CHECK(esp_zigbee_start(false));                   // no-autostart
    esp_zigbee_launch_mainloop();                               // 阻塞
    esp_zigbee_deinit();
    vTaskDelete(NULL);
}
```

### Signal Handler 骨架（所有示例统一形态）
```c
static bool esp_zigbee_app_signal_handler(const ezb_app_signal_t *app_signal)
{
    ezb_app_signal_type_t signal_type = ezb_app_signal_get_type(app_signal);
    switch (signal_type) {
    case EZB_ZDO_SIGNAL_SKIP_STARTUP:
        ezb_bdb_start_top_level_commissioning(EZB_BDB_MODE_INITIALIZATION);
        break;
    case EZB_BDB_SIGNAL_DEVICE_FIRST_START:
    case EZB_BDB_SIGNAL_DEVICE_REBOOT: {
        ezb_bdb_comm_status_t status = *((ezb_bdb_comm_status_t *)ezb_app_signal_get_params(app_signal));
        if (status == EZB_BDB_STATUS_SUCCESS) { /* 启动 deferred_driver_init；按角色 formation/steering */ }
        else { alarm_timer_schedule(esp_zigbee_alarm_bdb_commissioning, EZB_BDB_MODE_INITIALIZATION, 1000); }
    } break;
    case EZB_BDB_SIGNAL_STEERING: { /* 入网结果 */ } break;
    /* EZB_BDB_SIGNAL_FORMATION / EZB_ZDO_SIGNAL_DEVICE_ANNCE / EZB_ZDO_SIGNAL_LEAVE / EZB_NWK_SIGNAL_PERMIT_JOIN_STATUS ... */
    default: break;
    }
    return true;
}
```

### 锁规约
```c
// Zigbee 主循环回调内 / esp_zigbee_task_queue_post 投递的回调内：无需加锁
// 其他任务、按钮 ISR 派发、定时器：必须加锁
esp_zigbee_lock_acquire(portMAX_DELAY);
ezb_zcl_on_off_toggle_cmd_req(&cmd_req);
esp_zigbee_lock_release();
```

### 调试输出约定
- 日志统一用 `ESP_LOGI/ESP_LOGW/ESP_LOGE`，每文件 `static const char *TAG = "DEVICE_NAME";`
- 串口波特率默认 115200（`idf.py monitor`）
- 抓包默认 TC link key：`ZigbeeAlliance09`（`5A:69:67:42:65:65:41:6C:6C:69:61:6E:63:65:30:39`）

## Build Workflow

1. 设置 IDF 环境：`source $IDF_PATH/export.sh`（示例对齐 ESP-IDF v5.5.4）
2. 设定目标：`idf.py set-target esp32h2`（或 `esp32c6`）
3. 添加依赖（若新建项目）：`idf.py add-dependency "espressif/esp-zigbee-lib^2.0.0"`
4. 配置（菜单）：`idf.py menuconfig`（GPIO、RCP 引脚、`ZB_DEBUG_MODE`、PM 等）
5. 编译：`idf.py build`
6. 烧录：`idf.py -p PORT erase_flash flash monitor`（首次务必 `erase_flash`）
7. 调试：串口日志 + Wireshark 抓包（Pyspinel + 另一块 H2/C6 烧 `ot_rcp`）

## Zigbee 应用代码生成 Checklist

- [ ] `main/<device>.h` 定义正确的 `ESP_ZIGBEE_ZC/ZR/ZED_CONFIG()` 与 `ESP_ZIGBEE_DEFAULT_CONFIG()`
- [ ] 角色与 BDB 模式匹配（ZC→FORMATION+STEERING；ZR/ZED→STEERING）
- [ ] `partitions.csv` 含 `zb_storage`(16K,nvs) 与 `zb_fct`(1K,fat)
- [ ] `app_main` 调用 `nvs_flash_init_partition(ESP_ZIGBEE_STORAGE_PARTITION_NAME)`
- [ ] 主任务栈 ≥ 4096，调用顺序 init→setup→create device→start(false)→launch_mainloop
- [ ] `esp_zigbee_setup_commissioning` 中设置主/次信道掩码、注册 signal handler、（终端）关 RxOnWhenIdle
- [ ] signal handler 处理 `EZB_ZDO_SIGNAL_SKIP_STARTUP` 与 `EZB_BDB_SIGNAL_DEVICE_FIRST_START`
- [ ] 数据模型：`create_device_desc` → `add_endpoint_desc` → `device_desc_register`
- [ ] 需要收属性写入：`ezb_zcl_core_action_handler_register` 处理 `EZB_ZCL_CORE_SET_ATTR_VALUE_CB_ID`
- [ ] 应用任务调用任何 Zigbee API 前 `esp_zigbee_lock_acquire/release`
- [ ] commissioning 失败用 `alarm_timer_schedule` 重试
- [ ] `idf_component.yml` 声明 `espressif/esp-zigbee-lib >=2.0.0` 与 IDF `>=5.2`

## Do Not Modify

- `espressif-repos/esp-zigbee-sdk/components/esp-zigbee-lib/` — 预编译库与公共头（`ezbee/`、`compat/`），只读引用
- `SKILL.md` frontmatter — Skill 元数据
- 仓库 `docs/en/` — 原始文档，仅作引用来源
