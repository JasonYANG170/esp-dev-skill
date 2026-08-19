# Matter 设备控制（controller）

> **适用摘要**: 在 matter_controller 示例中获取 Matter 设备列表、控制 OnOff / LevelControl / ColorControl cluster，以及 Thread Border Router 的板子选择与配置。

> Version: selected repo/component version; align with the user project and dependency manifest.
> Validation: draft metadata added from repository routing; verify APIs, Kconfig symbols, and component dependencies against the selected version.

## 触发意图

- "Matter 控制"
- "get_device_list"
- "control_device"
- "Thread Border Router"
- "用语音控制 Matter 设备"

## 前置条件

| 条件 | 要求 |
|---|---|
| 示例 | `examples/matter_controller/` |
| 板子 | `m5stack_cores3`（仅控制器）或 `m5stack_cores3_h2_gateway`（控制器 + Thread Border Router�� |
| 配网 | 须用 ESP RainMaker App commission（见 `recipes/device_setup_provisioning.md`） |
| 配置 | `CONFIG_ESP_CLIENT_ONLY_MATTER_CONTROLLER=y`（见 Kconfig 互斥约束） |

## 分步说明

### 1. 启动 Matter 控制器任务

matter_controller 示例的 `app_main` 在 Agent 启动后再启动控制器任务（来自 `examples/matter_controller/main/app_main.c`）：

```c
app_agent_init(&agent_config);
app_tools_register();
app_agent_start();
/* 最后一步：启动 Matter controller（依赖 RainMaker service，须在 setup 之后） */
matter_controller_start_task();
```

`matter_controller_start_task()` 声明于 `examples/matter_controller/main/matter/app_controller.h`，内部依赖 RainMaker REST API（`RMAKER_REST_API_ENABLED`，由 client-only Kconfig 自动 select）。

### 2. 获取设备列表（工具层）

工具 `get_device_list` 由 `app_tools_get_device_list_handler`（`examples/matter_controller/main/app_tools.c`）实现：

```c
char *device_list = NULL;
matter_controller_get_device_list(&device_list);   /* 返回 JSON 字符串 */
*result = strdup(device_list ? device_list : "Failed to get device list.");
free(device_list);
```

`agent_config.json` 中 `get_device_list` 无参数。LLM 每次应重新调用（设备列表时效敏感）。

### 3. 控制设备（工具层）

工具 `control_device` 由 `app_tools_control_device_handler` 实现，参数：

| 参数 | 类型 | 说明 |
|---|---|---|
| `node_id` | STRING | 设备列表中的 node_id（十六进制字符串，内部 `strtoull(node_id, NULL, 16)`） |
| `cluster_id` | NUMBER | Cluster ID：OnOff=6 / LevelControl=8 / ColorControl=768 |
| `command_id` | NUMBER | 命令 ID（见下表） |
| `command_args` | STRING | 命令参数 JSON 对象 |

内部调用：

```c
matter_controller_control_device(&result,
                                 strtoull(node_id, NULL, 16),
                                 cluster_id, command_id,
                                 command_params_json);
```

### 4. 支持的 cluster / command（来自 agent_config.json 文档）

**OnOff (cluster_id=6)**

| command_id | 命令 | args |
|---|---|---|
| 0 | TurnOff | `{}` |
| 1 | TurnOn | `{}` |
| 2 | Toggle | `{}` |

**LevelControl (cluster_id=8)**

| command_id | 命令 | args（只改变量字段） |
|---|---|---|
| 0 | MoveToLevel | `{"0:U8":<level 0-254>, "1:U16":0, "2:U8":0, "3:U8":0}` |

**ColorControl (cluster_id=768)**

| command_id | 命令 | args |
|---|---|---|
| 0 | MoveToHue | `{"0:U8":<hue 0-254>, "1:U8":0, "2:U16":0, "3:U8":0, "4:U8":0}` |
| 3 | MoveToSaturation | `{"0:U8":<sat 0-254>, "1:U16":0, "2:U8":0, "3:U8":0}` |

> 色相参考：0=Red, 42=Yellow, 85=Green, 127=Cyan, 169=Blue, 212=Purple。
> 重要：传 `command_args` 时**只修改变量字段**，结构/固定字段保持不变。

### 5. 调用示例（C 工具层）

```c
/* 开灯：OnOff TurnOn */
matter_controller_control_device(&r, strtoull(node_id, NULL, 16), 6, 1, "{}");

/* 调亮度到 200：LevelControl MoveToLevel */
matter_controller_control_device(&r, strtoull(node_id, NULL, 16), 8, 0,
    "{\"0:U8\":200,\"1:U16\":0,\"2:U8\":0,\"3:U8\":0}");

/* 设色相为绿(85)：ColorControl MoveToHue */
matter_controller_control_device(&r, strtoull(node_id, NULL, 16), 768, 0,
    "{\"0:U8\":85,\"1:U8\":0,\"2:U16\":0,\"3:U8\":0,\"4:U8\":0}");
```

### 6. Thread Border Router 配置

需 `m5stack_cores3_h2_gateway` 板（M5Stack CoreS3 + H2 Gateway Module）：

```bash
idf.py select-board --board m5stack_cores3_h2_gateway
idf.py build flash monitor
```

在 ESP RainMaker App 设备页：
- 先 `Update Thread Dataset` 建立 Thread border
- 加新 Matter 设备到同组后点 `Update Device List` 通知控制器

### 7. 底层组件 API（来自 components/matter_controller）

直接编程时可用的句柄与服务（`examples/matter_controller/components/matter_controller/`）：

```c
/* rmaker_controller_service/app_matter_controller.h */
matter_controller_status_t status;   /* 位域：base_url_set / user_token_set / ... / matter_noc_installed */
matter_controller_enable(vendor_id, callback);   /* callback 类型见 matter_controller_callback_type_t */
matter_controller_handle_update();
matter_controller_report_status(status);
matter_controller_update_device_list();

/* rmaker_controller_service/app_matter_device_manager.h */
matter_device_t *fetch_device_list();
update_device_list(controller_handle);
init_device_manager(dev_list_update_cb);
```

RainMaker Matter Controller 服务参数宏（`matter_controller_std.h`）：
`ESP_RMAKER_DEVICE_MATTER_CONTROLLER`、`ESP_RMAKER_SERVICE_MATTER_CONTROLLER`、`ESP_RMAKER_PARAM_BASE_URL`、`_USER_TOKEN`、`_RMAKER_GROUP_ID`、`_MATTER_NODE_ID`、`_MATTER_CTL_CMD`、`_MATTER_CTL_STATUS` 等。

## 常见错误

| 错误 | 原因 | 解决方法 |
|---|---|---|
| 控制器不启动 | client-only 互斥项未关 | 关掉 ESP_MATTER_ENABLE_MATTER_SERVER / COMMISSIONER / WIFI/THREAD_NETWORK_COMMISSIONING_DRIVER / ENABLE_CHIPOBLE |
| 设备列表为空 | 未 Update Device List | App 内点 Update Device List，或调 `matter_controller_update_device_list()` |
| 控制无响应 | node_id/cluster_id/command_id 不符 schema | 严格按上表 cluster/command；args 只改变量字段 |
| Thread 设备加入不了 | 未先建 Thread border | 先 Update Thread Dataset |
| `matter_controller_start_task` 失败 | 在 setup 之前调用 | 必须在 `app_agent_start()` 之后调用（依赖 RainMaker service） |

## 参考

- `examples/matter_controller/README.md` — Matter 控制器示例说明
- `examples/matter_controller/setup_guide.md` — 配网与 Thread Border Router 步骤
- `examples/matter_controller/agent_config.json` — `control_device`/`get_device_list` 完整 schema（含 cluster/command 文档）
- `examples/matter_controller/main/app_tools.c` — 工具 handler 实现
- `examples/matter_controller/main/matter/app_controller.h` — `matter_controller_start_task` / `get_device_list` / `control_device`
- `examples/matter_controller/components/matter_controller/Kconfig` — client-only 互斥约束
- `examples/matter_controller/components/matter_controller/rmaker_controller_service/app_matter_controller.h` — 控制器句柄与状态
- `examples/matter_controller/components/matter_controller/rmaker_controller_service/matter_controller_std.h` — RainMaker 参数宏
