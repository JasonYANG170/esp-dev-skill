# 控制器节点与 User Helper API

> **适用摘要**: 把一个 ESP32 设备变成 RainMaker **控制器节点**，通过云端 User API（`CONFIG_ENABLE_RM_USER_HELPER_API`）枚举、查询、设置**其它**节点的参数 / 配置 / 调度 / 在线状态，并可移除用户-节点映射；支持子节点数据下行回调。

## 触发意图

- "RainMaker 控制器 / controller node"
- "一个节点控制其它节点"
- "固件里查 / 改别的节点的参数"
- "esp_rmaker_controller_enable"
- "User Helper API / RM_USER_HELPER_API"
- "esp_rmaker_auth_service / refresh token 调 User API"

## 前置条件

| 条件 | 要求 |
|---|---|
| 节点 | 已 `esp_rmaker_node_init()`；控制器服务须在 `esp_rmaker_start()` 之前 enable |
| Kconfig | `CONFIG_ENABLE_RM_USER_HELPER_API=y`（启用 helper 层 `app_rmaker_user_helper_api.h`） |
| 组件依赖 | `examples/common/rmaker_user_api/`（提供 core + helper API），`esp_rmaker_auth_service.h`（提供 refresh token） |
| 参考示例 | `examples/rainmaker_controller/`（CLI + auth service + controller_cb） |

## 能力概览

控制器节点是 RainMaker 独有的"固件侧编排者"模型：节点本身仍是一个普通 RainMaker 节点（有 device/param/OTA/调度等），同时额外启用一个**控制器服务**，让它能以**用户身份**调用云端 REST API，操作**同一用户名下所有其它节点**。这与传统的"一个节点只暴露自己的参数"模型完全不同。

| 能力 | helper API（启用 `CONFIG_ENABLE_RM_USER_HELPER_API` 后） |
|---|---|
| 列出节点 | `app_rmaker_user_helper_api_get_nodes_list(&list, &count)` |
| 节点参数 | `app_rmaker_user_helper_api_get_node_params(node_id, &params)` / `set_node_params(node_id, payload, &resp)` |
| 节点配置 | `app_rmaker_user_helper_api_get_node_config(node_id, &config)` |
| 在线状态 | `app_rmaker_user_helper_api_get_node_connection_status(node_id, &online)` |
| 用户映射 | `app_rmaker_user_helper_api_set_node_mapping(...)` / `get_node_mapping_status(...)` |
| 任意端点 | `app_rmaker_user_api_generic(&req, &status, &resp)`（core 层，自定义 query/payload） |

> 所有返回字符串均由调用者 `free()`。

## 分步说明

### 1. sdkconfig.defaults（启用 User Helper API + 控制器相关）

```text
# 启用 User Helper API（app_rmaker_user_helper_api.h 才可用）
CONFIG_ENABLE_RM_USER_HELPER_API=y

# 额外安全：恢复出厂时校验 user id
CONFIG_ESP_RMAKER_USER_ID_CHECK=y

# 控制台参数命令（get/set params CLI）
CONFIG_ESP_RMAKER_CONSOLE_PARAM_CMDS_ENABLE=y

# Secure Local Control
CONFIG_ESP_RMAKER_LOCAL_CTRL_AUTO_ENABLE=y
CONFIG_ESP_RMAKER_LOCAL_CTRL_SECURITY_1=y

# 滚回支持（与 OTA 配合）
CONFIG_BOOTLOADER_APP_ROLLBACK_ENABLE=y
```

### 2. 在 `app_main` 中启用控制器服务（start 之前）

控制器配置来自专用头文件 `esp_rmaker_controller.h`：

```c
#include <esp_rmaker_core.h>
#include <esp_rmaker_controller.h>
#include <esp_rmaker_auth_service.h>
#include <app_network.h>

/* 子节点数据下行回调：当被控节点向控制器回推数据时触发（不要阻塞） */
static esp_err_t controller_cb(const char *node_id, const char *data,
                               size_t data_size, void *priv_data)
{
    ESP_LOGI(TAG, "Controller received data from node %s: %.*s",
             node_id, data_size, data);
    return ESP_OK;
}

/* 在 app_main 中，esp_rmaker_node_init() 之后、esp_rmaker_start() 之前： */
esp_rmaker_controller_config_t controller_config = {
    .report_node_details = false,
    .cb = controller_cb,
    .priv_data = NULL,
};
esp_rmaker_controller_enable(&controller_config);

/* auth service 提供刷新令牌，User API 用它登录 */
esp_rmaker_auth_service_enable();

/* 标准服务照常 */
esp_rmaker_ota_enable_default();
esp_rmaker_timezone_service_enable();
esp_rmaker_schedule_enable();
esp_rmaker_scenes_enable();

esp_rmaker_start();
app_network_start((app_network_pop_type_t)CONFIG_APP_POP_TYPE);
```

> `esp_rmaker_controller_enable()` 必须在 `esp_rmaker_node_init()` 之后、`esp_rmaker_start()` 之前调用（见头文件 `@note`）。

### 3. 给控制器一个 `esp.device.controller` 设备（App 图标）

控制器节点本身也需挂一个设备，App 才能显示对应图标（设备类型字符串为字面量，**未**定义为标准宏）：

```c
esp_rmaker_device_t *controller_device =
    esp_rmaker_device_create("RMController", "esp.device.controller", NULL);
esp_rmaker_device_add_param(controller_device,
    esp_rmaker_name_param_create(ESP_RMAKER_DEF_NAME_PARAM, "ESP-RM-Controller"));
esp_rmaker_node_add_device(node, controller_device);
```

### 4. 用 Auth Service 的刷新令牌初始化 User API

控制器**不需要**在固件里硬编码用户名/密码：`esp_rmaker_auth_service_enable()` 会在节点 claimed 后从云获取刷新令牌并保存。首次执行 CLI 命令时再懒加载 User API：

```c
#include <esp_rmaker_auth_service.h>
#include "app_rmaker_user_api.h"
#include "app_rmaker_user_helper_api.h"

/* 首次 CLI 命令时调用（见 rainmaker_controller 的 app_cli_command.c） */
char *refresh_token = NULL;
char *base_url = NULL;
esp_rmaker_auth_service_get_user_token(&refresh_token);   /* 调用者 free */
esp_rmaker_auth_service_get_base_url(&base_url);          /* 调用者 free */

app_rmaker_user_api_config_t cfg = {
    .base_url      = base_url,
    .refresh_token = refresh_token,
};
app_rmaker_user_api_init(&cfg);
free(refresh_token);
free(base_url);
```

可选：注册登录成功/失败回调，把状态回写给 auth service（云端才会刷新令牌状态）：

```c
static void on_login_failure(int code, const char *reason) {
    esp_rmaker_user_auth_service_token_status_update(
        ESP_RMAKER_USER_AUTH_SERVICE_TOKEN_STATUS_EXPIRED_OR_INVALID);
}
static void on_login_success(void) {
    esp_rmaker_user_auth_service_token_status_update(
        ESP_RMAKER_USER_AUTH_SERVICE_TOKEN_STATUS_VERIFIED);
}
app_rmaker_user_api_register_login_failure_callback(on_login_failure);
app_rmaker_user_api_register_login_success_callback(on_login_success);
```

### 5. 用 Helper API 查询 / 设置其它节点（调用者负责 free）

```c
char *nodes_list = NULL;
uint16_t nodes_count = 0;
app_rmaker_user_helper_api_get_nodes_list(&nodes_list, &nodes_count);
/* nodes_list 为 JSON 字符串；用完 free(nodes_list) */

const char *target_node_id = "3485187E7F68";
char *params = NULL;
app_rmaker_user_helper_api_get_node_params(target_node_id, &params);
free(params);

/* 设置参数（payload 为 JSON，单引号包起来在 CLI 里传入） */
char *resp = NULL;
app_rmaker_user_helper_api_set_node_params(
    target_node_id,
    "{\"Light\":{\"power\":true}}",
    &resp);
free(resp);

bool online = false;
app_rmaker_user_helper_api_get_node_connection_status(target_node_id, &online);
```

### 6. 任意端点：用 core 层 `app_rmaker_user_api_generic`

helper 没覆盖的端点（如带详细 query 的 `user/nodes`，或移除用户-节点映射 `user/nodes/mapping`）用 core 层：

```c
app_rmaker_user_api_request_config_t req = {
    .reuse_session   = true,
    .api_type        = APP_RMAKER_USER_API_TYPE_GET,
    .api_name        = "user/nodes",
    .api_version     = "v1",
    .api_query_params= "node_details=true&status=true&config=true&params=true&show_tags=true&is_matter=false",
    .api_payload     = NULL,
};
int  status_code = 0;
char *response   = NULL;
app_rmaker_user_api_generic(&req, &status_code, &response);
/* 用完 free(response) */
```

移除用户-节点映射（PUT `user/nodes/mapping`，body `{"node_id":"...","operation":"remove"}`）也是走 `generic` + `APP_RMAKER_USER_API_TYPE_PUT`。

### 7. CLI 命令表（参考 `rainmaker_controller`）

示例把上面这些操作封装为 UART console 命令，命令名/含义如下（在 `app_cli_command.c` 中用 `esp_console_cmd_register` 注册）：

| 命令 | 说明 |
|---|---|
| `getnodes` | 列出该用户名下所有节点 ID |
| `getnodedetails` | 一次性拉所有节点的 config/status/params（带 `show_tags` 等 query） |
| `getparams <node_id>` | 取某节点参数 |
| `setparams <node_id> <json>` | 设某节点参数（JSON 用单引号） |
| `getnodeconfig <node_id>` | 取节点配置 |
| `getnodestatus <node_id>` | 在线 / 离线 |
| `getschedules <node_id>` | 取节点调度 |
| `setschedule <node_id> <json>` | 设节点调度 |
| `removenode <node_id>` | 移除用户-节点映射（`PUT user/nodes/mapping`） |
| `getheapstatus` | 打印 free / minimum free heap |

## 常见错误

| 错误 | 原因 | 解决方法 |
|---|---|---|
| 编译报 `app_rmaker_user_helper_api.h: No such file` | 未依赖 `rmaker_user_api` 组件或未启用 helper | 工程加 `rmaker_user_api` 依赖，且 `CONFIG_ENABLE_RM_USER_HELPER_API=y` |
| `app_rmaker_user_api_init` 返回失败 | refresh_token / base_url 为 NULL | 先 `esp_rmaker_auth_service_enable()` 且节点已 claimed，再取 token |
| 401 / 登录失败 | 刷新令牌过期或未拿到 | 注册 `login_failure_callback` 并 `token_status_update(EXPIRED_OR_INVALID)`，重新走 claiming |
| 控制器收不到子节点数据 | `controller_cb` 未设置或返回慢 | `esp_rmaker_controller_enable` 时传 `.cb`；回调内不要阻塞 |
| `controller_enable` 报错 | 在 `esp_rmaker_start()` 之后调用 | 移到 start 之前（与 OTA/schedule 等 enable 同期） |
| 内存泄漏 | helper 返回的字符串未 free | 每个返回 `char **` 的 API，调用者必须 `free()` |
| setparams JSON 解析失败 | payload 不是合法 JSON | 用单引号包裹 JSON，避免 shell 吞掉双引号 |

## 参考项目

- `examples/rainmaker_controller/` — 控制器节点完整示例（CLI + auth service + controller_cb）
  - `examples/rainmaker_controller/main/app_main.c` — controller_enable / auth_service_enable / 设备挂载
  - `examples/rainmaker_controller/main/app_cli_command.c` — 全部 CLI 命令注册与分发
  - `examples/rainmaker_controller/main/rainmaker_cli_handler.c` — User API 封装（含 `removenode` 走 generic）
  - `examples/rainmaker_controller/sdkconfig.defaults` — `CONFIG_ENABLE_RM_USER_HELPER_API=y`
- `examples/common/rmaker_user_api/` — User API 组件（core + helper）
  - `include/app_rmaker_user_api.h`、`include/app_rmaker_user_helper_api.h`
  - `README.md` — API 表与初始化示例
- `components/esp_rainmaker/include/esp_rmaker_controller.h` — `esp_rmaker_controller_enable/disable/get_active_group_id`、`esp_rmaker_controller_config_t`、`esp_rmaker_controller_cb_t`
- `components/esp_rainmaker/include/esp_rmaker_auth_service.h` — `esp_rmaker_auth_service_enable`、`get_user_token`、`get_base_url`、`token_status_update`
- `recipes/services.md` — 标准服务（OTA/调度/场景/系统）启用顺序
