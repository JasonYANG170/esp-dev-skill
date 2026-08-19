# Matter Controller 服务初始化与 RainMaker 联动

> **适用摘要**: `esp.service.matter-controller`（MatterCTL）RainMaker 服务的 6 个参数、7 位 MTCtlStatus 状态位图、控制器 NOC 签发与设备列表更新的手机 App 触发流程，以及固件侧 `matter_controller_enable` / `IP_EVENT_STA_GOT_IP` / Thread Border Router / `matter_controller_client` 的初始化链。

> Version: selected repo/component version; align with the user project and dependency manifest.
> Evidence: `repos/esp-agents-firmware/resources/`, source/examples in `repos/esp-agents-firmware/`, and this recipe path `repos/esp-agents-firmware/recipes/matter_controller_service_init.md`.
> Validation: draft metadata added from repository routing; verify APIs, Kconfig symbols, and component dependencies against the selected version.

## 触发意图

- "Matter Controller 服务"
- "UpdateNOC"
- "UpdateDeviceList"
- "MTCtlStatus 位图"
- "matter_controller_enable"
- "Matter 控制器授权失败 / 不工作"
- "esp.service.matter-controller"

## 前置条件

| 条件 | 要求 |
|---|---|
| 示例 | `examples/matter_controller/` |
| 板子 | `m5stack_cores3`（仅控制器）或 `m5stack_cores3_h2_gateway`（控制器 + Thread Border Router，需 `CONFIG_OPENTHREAD_BORDER_ROUTER=y`） |
| 配置 | `CONFIG_ESP_CLIENT_ONLY_MATTER_CONTROLLER=y`（client-only 互斥项，见 `recipes/matter_controller_control.md`） |
| 服务规格 | `examples/matter_controller/components/matter_controller/rmaker_controller_service/SPEC.md` |
| 手机 App | ESP RainMaker App（注：本示例暂不支持 RainMaker Home App 配网，后续支持） |

> 本 recipe 覆盖**服务初始化层**（service 参数 + 授权 + NOC + 设备列表）；设备列表获取与 cluster 控制的**工具层**见 `recipes/matter_controller_control.md`。

## 分步说明

### 1. 服务与参数（来自 SPEC.md）

`esp.service.matter-controller`（别名 **MatterCTL**）暴露 6 个参数（`matter_controller_std.h`）：

| 参数宏（`matter_controller_std.h`） | 参数名 | 类型 | 默认 | 标志 | 含义 |
|---|---|---|---|---|---|
| `ESP_RMAKER_PARAM_BASE_URL` | BaseURL | string | — | R W P | HTTP REST API 的 endpoint URL |
| `ESP_RMAKER_PARAM_USER_TOKEN` | UserToken | string | — | W P | refresh token，用于 `user/login2` 换 access_token（access_token 1 小时过期） |
| `ESP_RMAKER_PARAM_RMAKER_GROUP_ID` | RMakerGroupID | string | — | R W P | RainMaker 组 ID（绑定 Matter Fabric ID） |
| `ESP_RMAKER_PARAM_MATTER_NODE_ID` | MatterNodeID | string | — | R | 控制器的 Matter Node ID（大写十六进制字符串） |
| `ESP_RMAKER_PARAM_MATTER_CTL_CMD` | MTCtlCMD | int | -1 | W | 云下发给控制器的命令（1=UpdateNOC，2=UpdateDeviceList） |
| `ESP_RMAKER_PARAM_MATTER_CTL_STATUS` | MTCtlStatus | int | 0 | R P | 控制器状态位图（见步骤 2） |

> 标志：R=可读 W=可写 P=持久化。UserToken 仅 W P（写后不可回读，安全考虑）。

### 2. MTCtlStatus 7 位状态位图（来自 SPEC.md + app_matter_controller.h）

`matter_controller_status_t` 是 union：`int raw` 与 7 个 bit 位对应。

| Bit | 字段（`matter_controller_status_t`） | 含义 |
|---|---|---|
| 0 | `base_url_set` | BaseURL 已设 |
| 1 | `user_token_set` | UserToken 已设 |
| 2 | `access_token_set` | access_token 已用 UserToken 换取 |
| 3 | `rmaker_group_id_set` | RMakerGroupID 已设 |
| 4 | `matter_fabric_id_set` | 与 RMakerGroupID 对应的 Matter Fabric ID 已获取 |
| 5 | `matter_node_id_set` | 控制器 Matter Node ID 已获取 |
| 6 | `matter_noc_installed` | 控制器 NOC Chain 已安装 |

调试时用 `matter_controller_report_status(status)` 上报，App 端读 `MTCtlStatus`。授权完成的标志是**所有 7 位都为 1**。

### 3. 手机 App 触发的初始化流程（来自 SPEC.md "Matter Controller Initialization"）

```
[1] 配网: 设备经 Wi-Fi provisioning 拿到网络凭据（见 recipes/device_setup_provisioning.md）

[2] App setparams: {"MatterCTL":{"BaseURL":<url>,"UserToken":<token>,"RMakerGroupID":<gid>}}
    → 固件写 3 个参数，更新 base_url_set/user_token_set/rmaker_group_id_set 位

[3] App 等 MTCtlStatus 报告超时前所有位都变 1
    （含 access_token_set / matter_fabric_id_set / matter_node_id_set / matter_noc_installed）

[4] App 收到 MatterNodeID 报告（控制器已就绪；App 永远不会与该 node ID 建 CASE session）

[5] App setparams: {"MatterCTL":{"MTCtlCMD": 2}}   # UpdateDeviceList
    → 控制器拉取本 Matter Fabric 内的设备列表（加/删设备后都需重发）
```

> MTCtlCMD 的两个枚举（SPEC.md）：
> - `1` = **UpdateNOC** — 控制器授权后生成新 CSR，发回云端，云返回的 NOC 被安装。
> - `2` = **UpdateDeviceList** — 控制器授权后拉取同 Fabric 下的 Matter 设备；加/删设备都需执行。

### 4. 固件侧初始化链（来自 app_controller.cpp `matter_controller_start()`）

`matter_controller_start_task()`（`examples/matter_controller/main/matter/app_controller.h`）创建 `launch_matter` 任务，内部按固定顺序：

```c
/* (a) NVS（外部已 init，此处幂等再调一次） */
nvs_flash_init();

/* (b) RainMaker node：若未初始化则建 node + MatterController 设备并加 write_cb */
esp_rmaker_node_t *node = esp_rmaker_node_init(&rainmaker_cfg, "ESP RainMaker Device", "Controller");
esp_rmaker_device_t *dev =
    esp_rmaker_device_create("MatterController", ESP_RMAKER_DEVICE_MATTER_CONTROLLER, NULL);
esp_rmaker_device_add_cb(dev, write_cb, NULL);
esp_rmaker_device_add_param(dev,
    esp_rmaker_name_param_create(ESP_RMAKER_DEF_NAME_PARAM, "MatterController"));
esp_rmaker_node_add_device(node, dev);

/* (c) 仅 Thread Border Router 板（CONFIG_OPENTHREAD_BORDER_ROUTER）：使能 Thread BR */
#ifdef CONFIG_OPENTHREAD_BORDER_ROUTER
    esp_openthread_platform_config_t thread_cfg = {
        .radio_config = ESP_OPENTHREAD_DEFAULT_RADIO_CONFIG(),
        .host_config  = ESP_OPENTHREAD_DEFAULT_HOST_CONFIG(),
        .port_config  = ESP_OPENTHREAD_DEFAULT_PORT_CONFIG(),
    };
#ifdef CONFIG_AUTO_UPDATE_RCP
    esp_rcp_update_config_t rcp_cfg = ESP_OPENTHREAD_RCP_UPDATE_CONFIG();
    esp_rmaker_thread_br_enable(&thread_cfg, &rcp_cfg);
#else
    esp_rmaker_thread_br_enable(&thread_cfg);
#endif
#endif

/* (d) 启用 Matter Controller service（注册 6 个参数 + write_cb 驱动命令） */
matter_controller_enable(CONFIG_ESP_MATTER_CONTROLLER_VENDOR_ID, app_matter_controller_callback);

/* (e) 启动 Matter stack */
esp_matter::start(NULL);

/* (f) 初始化 matter_controller_client（CASE session 客户端） */
esp_matter::lock::chip_stack_lock(portMAX_DELAY);
esp_matter::controller::matter_controller_client::get_instance().init(0, 0, 5580);
esp_matter::lock::chip_stack_unlock();

/* (g) 初始化 device manager（设备列表缓存） */
init_device_manager(NULL);

/* (h) 注册 IP_EVENT_STA_GOT_IP handler：拿到 IP 后立刻 handle_update + update_device_list */
esp_event_handler_register(IP_EVENT, IP_EVENT_STA_GOT_IP, &update_controller_handler, NULL);
```

> 关键约束（`app_controller.h` 注释）：`matter_controller_start_task()` **依赖 RainMaker service，必须在 `setup_rainmaker_start`（即 `app_agent_start`）之后调用**。

### 5. 拿到 IP 后的自动刷新（update_controller_handler）

`IP_EVENT_STA_GOT_IP` 触发一次控制器刷新，成功后**注销自身**（只跑一次）：

```c
static void update_controller_handler(void *arg, esp_event_base_t base, int32_t id, void *data) {
    if (base == IP_EVENT && id == IP_EVENT_STA_GOT_IP) {
        if (matter_controller_handle_update() == ESP_OK) {
            matter_controller_update_device_list();
            esp_event_handler_unregister(IP_EVENT, IP_EVENT_STA_GOT_IP, &update_controller_handler);
        }
    }
}
```

`matter_controller_handle_update()` 处理未消费的参数写（例如 App 已下发 BaseURL/UserToken/RMakerGroupID）；`matter_controller_update_device_list()` 拉取最新 Fabric 内设备列表。

### 6. callback 五种类型（app_matter_controller_callback.h / .cpp）

`matter_controller_enable` 注册的 `app_matter_controller_callback` 按 `matter_controller_callback_type_t` 分派：

| 枚举 | 处理函数（app_matter_controller_callback.cpp） | 作用 |
|---|---|---|
| `AUTHORIZE` | `app_matter_controller_authorize` | 用 BaseURL+UserToken 调 `fetch_access_token`（buffer 1800 字节），失败回 INVALID_ARG |
| `QUERY_MATTER_FABRIC_ID` | `app_matter_controller_fetch_matter_fabric_id` | 用 access_token+RMakerGroupID 调 `fetch_matter_fabric_id` |
| `SETUP_CONTROLLER` | 先 `update_device_list`，再 `app_matter_controller_setup_controller` | 拉 IPK（`fetch_fabric_ipk`）后 `matter_controller_client::setup_controller(ipk_span)`；静态 `controller_setup` 标志保证只执行一次 |
| `UPDATE_CONTROLLER_NOC` | `app_matter_controller_update_noc` | 更新 NOC（connectedhomeip 上游 TODO，当前直接返回 ESP_OK） |
| `UPDATE_DEVICE` | `app_matter_controller_update_device_list` | 调 `update_device_list(handle)` 拉取并刷新设备列表 |

> AUTHORIZE 失败的常见原因：`base_url`/`user_token` 缺失，或 `access_token` 已存在（避免重复授权）。

### 7. 自定义服务回调（替换 app_matter_controller_callback）

若需自定义 NOC 签发或 fabric 查询逻辑，把 `matter_controller_enable` 的第二个参数换成自己的 `matter_controller_callback_t`（签名见 `app_matter_controller.h`）：

```c
/* 来自 app_matter_controller.h */
typedef esp_err_t (*matter_controller_callback_t)(matter_controller_handle_t *handle,
                                                   matter_controller_callback_type_t type);

/* 注册：vendor_id 取自 CONFIG_ESP_MATTER_CONTROLLER_VENDOR_ID */
matter_controller_enable(CONFIG_ESP_MATTER_CONTROLLER_VENDOR_ID, my_callback);
```

`matter_controller_handle_t` 内含 `base_url` / `user_token` / `access_token` / `rmaker_group_id` / `matter_fabric_id` / `matter_node_id` / `matter_noc_installed` / `matter_vendor_id` / `service` 字段，可在回调里直接读写。

## 常见错误

| 错误 | 原因 | 解决方法 |
|---|---|---|
| App 下发 BaseURL/UserToken 后 MTCtlStatus 不变 | `matter_controller_start_task` 未调，或在 `app_agent_start` 前调 | 必须在 `app_agent_start()` 之后再 `matter_controller_start_task()`（依赖 RainMaker service） |
| `access_token_set` 位一直为 0 | AUTHORIZE 回调里 `fetch_access_token` 失败（UserToken 错 / endpoint 不通） | 确认 BaseURL 指向正确部署、UserToken 是有效的 refresh token；access_token 1 小时过期需重换 |
| `matter_noc_installed` 位为 0 | setup_controller 未跑或 IPK 拉取失败 | 检查 `SETUP_CONTROLLER` 回调：`fetch_fabric_ipk` 依赖 access_token + rmaker_group_id；NOC 已装时会用空 IPK |
| App 点 Update Device List 无效 | 未先建 Thread border，或新设备不在同一 RMakerGroupID | Thread 设备：先 `Update Thread Dataset` 建 border；确认设备与控制器在同一 RainMaker 组（= 同一 Matter Fabric） |
| `matter_controller_client::init` 失败 | chip stack lock 未获取 | 严格按 `chip_stack_lock(portMAX_DELAY)` → `init(...)` → `chip_stack_unlock()` 顺序 |
| Thread BR 不工作 | 非 `m5stack_cores3_h2_gateway` 板，或未配 RCP 固件 | 用 `m5stack_cores3_h2_gateway`；`CONFIG_AUTO_UPDATE_RCP=y` 时需先挂载 SPIFFS（`init_spiffs`）放 RCP 固件 |
| 重复 setup_controller | `controller_setup` 静态标志已置位 | 设计如此；若需重置需重启设备 |

## 参考

- `examples/matter_controller/components/matter_controller/rmaker_controller_service/SPEC.md` — 服务规格（6 参数 / 命令枚举 / 初始化时序 / 状态位图）
- `examples/matter_controller/setup_guide.md` — 配网、Controller Configuration 重新登录、Thread border、Update Device List 的 App 操作
- `examples/matter_controller/README.md` — 示例总览
- `examples/matter_controller/main/matter/app_controller.cpp` — `matter_controller_start` / `update_controller_handler` / `matter_controller_start_task`
- `examples/matter_controller/main/matter/app_controller.h` — `matter_controller_start_task` / `get_device_list` / `control_device`
- `examples/matter_controller/components/matter_controller/rmaker_controller_service/app_matter_controller.h` — `matter_controller_status_t` / `matter_controller_handle_t` / callback 枚举 / `enable`/`handle_update`/`report_status`/`update_device_list`
- `examples/matter_controller/components/matter_controller/rmaker_controller_service/app_matter_controller_callback.h` — `app_matter_controller_callback` 声明
- `examples/matter_controller/components/matter_controller/rmaker_controller_service/app_matter_controller_callback.cpp` — 5 种 callback 实现（authorize / fetch_fabric_id / setup_controller / update_noc / update_device_list）
- `examples/matter_controller/components/matter_controller/rmaker_controller_service/matter_controller_std.h` — RainMaker 参数宏与 `matter_controller_service_create`
- `examples/matter_controller/components/matter_controller/rmaker_controller_service/app_matter_device_manager.h` — `update_device_list` / `fetch_device_list` / `init_device_manager`
