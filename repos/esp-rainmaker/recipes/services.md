# 标准服务（时区 / 系统 / 连接性 / 分组）

> **适用摘要**: 启用 RainMaker 标准服务：时区（timezone）、系统（reboot/factory-reset/wifi-reset）、连接性（Connectivity，含 MQTT LWT）、分组（Groups）。均须在 `esp_rmaker_start()` 之前调用。

> Version: selected repo/component version; align with the user project and dependency manifest.
> Evidence: `repos/esp-rainmaker/resources/`, source/examples in `repos/esp-rainmaker/`, and this recipe path `repos/esp-rainmaker/recipes/services.md`.
> Validation: draft metadata added from repository routing; verify APIs, Kconfig symbols, and component dependencies against the selected version.

## 触发意图

- "时区服务 timezone"
- "系统服务 reboot/factory reset"
- "Connectivity 服务 / MQTT LWT"
- "Groups 分组服务"
- "esp_rmaker_system_service_enable"

## 前置条件

| 条件 | 要求 |
|---|---|
| 节点 | 已 `esp_rmaker_node_init()` |
| 参考示例 | `examples/led_light/main/app_main.c`（system service）、`examples/switch/main/app_main.c`（timezone） |

## 分步说明

### 1. 时区服务（timezone）

```c
#include <esp_rmaker_core.h>

esp_rmaker_timezone_service_enable();
```

启用后 App 可写入 POSIX 时区（如 `CST-8`）或地点字符串（如 `Asia/Shanghai`）。调度依赖正确时区。

可选参数 helper：
```c
esp_rmaker_timezone_param_create(ESP_RMAKER_DEF_TIMEZONE_NAME, "Asia/Shanghai");
esp_rmaker_timezone_posix_param_create(ESP_RMAKER_DEF_TIMEZONE_POSIX_NAME, "CST-8");
```

时间服务相关 Kconfig：
- `CONFIG_ESP_RMAKER_TIME_CURRENT_TIME_PARAM`（默认 n）：加 `Current-Time` 参数，App 写 epoch 给无网设备设本地 RTC。

### 2. 系统服务（reboot / factory-reset / wifi-reset）

```c
#include <esp_rmaker_core.h>

esp_rmaker_system_serv_config_t cfg = {
    .flags               = SYSTEM_SERV_FLAGS_ALL,  /* 或按需 OR 三个 FLAG */
    .reboot_seconds      = 2,
    .reset_seconds       = 2,
    .reset_reboot_seconds= 2,
};
esp_rmaker_system_service_enable(&cfg);
```

标志宏：
```c
SYSTEM_SERV_FLAG_REBOOT        (1 << 0)
SYSTEM_SERV_FLAG_FACTORY_RESET (1 << 1)
SYSTEM_SERV_FLAG_WIFI_RESET    (1 << 2)
SYSTEM_SERV_FLAGS_ALL          /* 三者或 */
```

触发后会产生 `RMAKER_COMMON_EVENT` 事件：
- `RMAKER_EVENT_REBOOT`（`event_data` 为秒数 `uint8_t *`）
- `RMAKER_EVENT_WIFI_RESET`
- `RMAKER_EVENT_FACTORY_RESET`

标准参数 helper：
```c
esp_rmaker_reboot_param_create(ESP_RMAKER_DEF_REBOOT_NAME);
esp_rmaker_factory_reset_param_create(ESP_RMAKER_DEF_FACTORY_RESET_NAME);
esp_rmaker_wifi_reset_param_create(ESP_RMAKER_DEF_WIFI_RESET_NAME);
```

### 3. Connectivity 服务（MQTT 在线状态 + LWT）

```c
#include <esp_rmaker_connectivity.h>

esp_rmaker_connectivity_enable();
```

行为（见头文件注释）：
- MQTT 连上时上报 `{"Connectivity":{"Connected":true}}`
- 设置 MQTT LWT，异常断开时发布 `{"Connectivity":{"Connected":false}}`
- LWT topic 为 `node/{node_id}/params/local`，设了 group_id 则为 `node/{node_id}/params/local/{group_id}`
- 自动从 flash 读取持久化 group_id 配置 LWT

相关 API：
```c
esp_err_t esp_rmaker_connectivity_update_lwt(const char *group_id);  /* group_id 变化时更新 LWT */
bool     esp_rmaker_connectivity_is_enabled(void);
```

Kconfig：`CONFIG_ESP_RMAKER_CONNECTIVITY_REPORT_DELAY`（默认 5 秒）—— MQTT 连上后延迟上报 `Connected=true`，避免重启竞态。

### 4. Groups 服务（分组）

```c
#include <esp_rmaker_groups.h>

esp_rmaker_groups_service_enable();
```

启用后 App 可设父分组 ID（`pgrp_id`）。设了之后：
- 所有 local params 上报到 `node/<node_id>/params/local/<pgrp_id>`（而非 `.../params/local/`）
- 可用 `esp_rmaker_publish_direct()` 在 `node/<node_id>/direct/params/local/<pgrp_id>` 直发，绕过云端处理降低延迟

相关 API：
```c
esp_err_t esp_rmaker_store_group_id(const char *group_id);   /* 存 NVS */
esp_err_t esp_rmaker_get_stored_group_id(char **group_id);   /* 读 NVS，调用者释放 */
```

> `esp_rmaker_connectivity_enable()` 内部会自动读持久化 group_id 并配置 LWT。

### 5. 标准服务 helper（自定义集成时）

```c
#include <esp_rmaker_standard_services.h>

/* 时区服务 */
esp_rmaker_time_service_create("Time", "Asia/Shanghai", "CST-8", NULL);
/* 系统服务（空，需自行加参数） */
esp_rmaker_create_system_service("System", NULL);
/* 本地控制服务 */
esp_rmaker_create_local_control_service("LocalControl", pop, sec_type, NULL);
/* 用户认证服务 */
esp_rmaker_create_user_auth_service("UserAuth", bulk_write_cb, bulk_read_cb, NULL);
/* 分组服务 */
esp_rmaker_create_groups_service("Groups", bulk_write_cb, group_id, NULL);
```

## 常见错误

| 错误 | 原因 | 解决��法 |
|---|---|---|
| 调度时间错 | 时区服务未启用 | `esp_rmaker_timezone_service_enable()` |
| reboot 不触发 | 未加 `SYSTEM_SERV_FLAG_REBOOT` | 用 `SYSTEM_SERV_FLAGS_ALL` 或显式 OR |
| Connectivity 状态抖动 | 重启时 LWT 与新连接竞态 | 设 `CONNECTIVITY_REPORT_DELAY`（默认 5 秒） |
| Groups 上报 topic 不变 | 未设 group_id | App 设 group_id 或调 `esp_rmaker_store_group_id` |
| 服务 enable 报错 | 在 `esp_rmaker_start()` 之后调用 | 所有 `*_enable` 移到 start 之前 |

## 参考

- `components/esp_rainmaker/include/esp_rmaker_connectivity.h`
- `components/esp_rainmaker/include/esp_rmaker_groups.h`
- `components/esp_rainmaker/include/esp_rmaker_standard_services.h`
- `components/esp_rainmaker/include/esp_rmaker_core.h`（timezone/system service）
- `examples/led_light/main/app_main.c` — system service 完整用法
