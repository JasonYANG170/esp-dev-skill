---
name: esp-rainmaker-skill
description: >-
  AI Skill for ESP RainMaker cloud IoT agent firmware development on ESP32 SoCs. Used when
  users need to create, modify, or debug ESP RainMaker projects, including node/device/param
  modeling, claiming, OTA, scheduling, scenes, local control, timezones, and MQTT connectivity.
  Trigger words: "ESP RainMaker", "RainMaker", "esp-rainmaker", "ESP32", "esp_rmaker", "claiming",
  "self claim", "assisted claim", "RainMaker OTA", "esp.device", "esp.param", "节点", "乐鑫云", "远程控制"
license: Apache-2.0
metadata:
  author: Community
  version: "1.1.0"
---

# esp-rainmaker-skill

面向 ESP RainMaker 云 IoT 代理（Agent）固件开发的 AI Skill。ESP RainMaker 是乐鑫提供的端到端解决方案，让 ESP32 系列 SoC 无需在云端做任何配置即可实现远程控制与监控。本 Skill 基于真实仓库 `components/esp_rainmaker/include` 头文件、`Kconfig.projbuild`、以及 `examples/` 中的官方示例编写，提供场景化 recipes、真实 API 参考、配置参考、常见陷阱与执行工作流，确保生成的固件代码不会臆造 API。

## Core Principles

1. **绝不臆造 API** — 任何函数名、结构体、宏、Kconfig 选项都必须能在 `resources/` 或仓库头文件中找到；找不到就省略，不要编造。
2. **初始化顺序严格** — `nvs_flash_init()` → `app_network_init()` → `esp_rmaker_node_init()` → 添加设备/服务 → 各 `*_enable()` → `esp_rmaker_start()` → `app_network_start()`。`esp_rmaker_node_init()` 必须在使用任何其它 RainMaker API 之前调用。
3. **先建网络后建节点** — Wi-Fi/Thread 初始化（`app_network_init()`）必须在 `esp_rmaker_node_init()` 之前；而真正连接（`app_network_start()`）在 `esp_rmaker_start()` 之后，最好在 Wi-Fi 已连接后再启动 Agent。
4. **设备模型三要素** — Node（节点）→ Device/Service（设备/服务）→ Param（参数）。设备用 `esp_rmaker_device_create()` 或标准 helper（如 `esp_rmaker_switch_device_create()`）创建，再 `esp_rmaker_node_add_device()` 挂到节点；参数同理 `esp_rmaker_param_create()` + `esp_rmaker_device_add_param()`。
5. **写回调中必须回写参数** — 在 write callback 内处理完硬件动作后，调用 `esp_rmaker_param_update()`（仅更新）或 `esp_rmaker_param_update_and_report()`（更新并上报），否则云端状态不一致。
6. **参数类型决定 UI** — 给参数加 UI Type（`esp_rmaker_param_add_ui_type()`，如 `ESP_RMAKER_UI_TOGGLE`/`ESP_RMAKER_UI_SLIDER`），手机 App 才能渲染对应控件。标准参数 helper（如 `esp_rmaker_power_param_create`）已内置类型。
7. **`*_enable()` 必须在 `esp_rmaker_start()` 之前** — OTA、schedule、scenes、timezone、system service、local control、groups、connectivity 等服务启用 API 都要求在 `esp_rmaker_start()` 前调用。
8. **Claiming 类型由 Kconfig 决定** — `CONFIG_ESP_RMAKER_SELF_CLAIM` / `CONFIG_ESP_RMAKER_ASSISTED_CLAIM` / `CONFIG_ESP_RMAKER_NO_CLAIM`。Self Claim 在 ESP32/ESP32-C2 不可用；Assisted Claim 需要 BT 且 ESP32-S2 不支持。
9. **MQTT 预算机制** — 默认 `CONFIG_ESP_RMAKER_MQTT_ENABLE_BUDGETING=y`，预算耗尽时消息会被丢弃。高频上报需调大 `CONFIG_ESP_RMAKER_MQTT_DEFAULT_BUDGET`/`MAX_BUDGET`，或用 `esp_rmaker_param_update()` 合并后一次性 `esp_rmaker_report_updated_params()`。
10. **值类型用 helper 构造** — `esp_rmaker_bool()`/`esp_rmaker_int()`/`esp_rmaker_float()`/`esp_rmaker_str()`/`esp_rmaker_obj()`/`esp_rmaker_array()` 返回 `esp_rmaker_param_val_t`，用于参数创建与更新。
11. **Read 回调当前不会被调用** — 客户端与节点之间是异步通信，read 请求不会到达节点；read callback 仅供未来使用，业务逻辑应依赖 write callback 与主动上报。
12. **Local Control 与 on-network chal_resp 互斥** — `CONFIG_ESP_RMAKER_LOCAL_CTRL_FEATURE_ENABLE` 与 `CONFIG_ESP_RMAKER_ON_NETWORK_CHAL_RESP_ENABLE` 都使用 protocomm_httpd（单例），不能同时启用。同理 `CONFIG_ESP_RMAKER_LOCAL_CTRL_CHAL_RESP_ENABLE` 与 `CONFIG_ESP_RMAKER_ON_NETWORK_CHAL_RESP_ENABLE` 也互斥。
13. **运行时新增/删除设备必须 `esp_rmaker_report_node_details()`** — 启动期 node config 不含运行期才创建的设备（如 Zigbee 网关动态映射的子设备、控制器动态发现的目标）。每次 `esp_rmaker_node_add_device()`/`remove_device()` 之后立即调用 `esp_rmaker_report_node_details()`，云端才会刷新设备列表。
14. **节点角色不止"被控对象"** — 一个 RainMaker 节点可兼任：(a) 普通业务节点（暴露自己的参数）；(b) **控制器节点**（`esp_rmaker_controller_enable` + User Helper API，以用户身份操作**其它**节点）；(c) **网关/BR**（把 Zigbee/Thread 子设备动态映射成 RainMaker 设备）。控制器/网关服务仍须在 `esp_rmaker_start()` 之前 enable。

## When to Use

**Applicable:**
- 创建新的 ESP RainMaker 固件工程（switch / light / fan / sensor / multi-device 等）
- 定义自定义设备（Device）、参数（Param）与服务（Service）
- 集成 OTA、调度（Scheduling）、场景（Scenes）、时区、系统服务
- 处理 Claiming（Self / Assisted）、Wi-Fi 配网、用户-节点映射
- 启用本地控制（Local Control）、MQTT 直发、Connectivity/Groups 服务
- 以太网主传输 / 双网络 / on-network challenge-response 配网
- 控制器节点（一个节点经云端控制其它节点）、Zigbee 网关（动态映射子设备）、Thread 边界路由器
- 移植或调试已有 RainMaker 示例

**Not applicable:**
- 非 ESP32 系列 SoC 的固件（如 STM32、CH57x 等其它厂商芯片）
- ESP-IDF 基础外设驱动问题（GPIO/UART/SPI 本身），除非与 RainMaker 设备参数绑定
- RainMaker 云端后台配置、手机 App UI 定制（这些不在本仓库固件范围内）
- 纯 Python host 工具（`tools/` 下的 CLI 脚本）开发——本 Skill 聚焦 C 固件

---

## Scenario Quick Reference (Recipes)

当用户意图匹配以下场景时，**先阅读对应 recipe**，其中包含完整调用链、分步说明与常见错误。

### 入门与节点

| recipe | 场景 |
|---|---|
| `recipes/getting_started.md` | 从零创建 RainMaker 节点：NVS、网络、node_init、添加设备、start 完整流程 |
| `recipes/claiming_and_provisioning.md` | Claiming（Self/Assisted/No）与 Wi-Fi/Thread 配网、PoP、用户-节点映射 |
| `recipes/ethernet_connectivity.md` | 以太网主传输 / 双网络（Wi-Fi+以太网）/ on-network challenge-response 配网 |

### 设备建模

| recipe | 场景 |
|---|---|
| `recipes/custom_device.md` | 自定义设备：device_create + 自定义参数 + UI Type + bounds + 写回调 |
| `recipes/multi_device.md` | 一个节点挂多个设备（Switch + Light + Fan + Sensor） |
| `recipes/standard_devices.md` | 标准设备 helper（switch/lightbulb/fan/temp_sensor）与标准参数 |

### 服务与功能

| recipe | 场景 |
|---|---|
| `recipes/ota_update.md` | RainMaker OTA：`esp_rmaker_ota_enable_default()`、OTA 事件、回滚诊断 |
| `recipes/scheduling_scenes.md` | 调度（schedule）与场景（scenes）服务启用与回调源 |
| `recipes/services.md` | 时区、系统（reboot/factory-reset/wifi-reset）、Connectivity、Groups 服务 |

### 进阶

| recipe | 场景 |
|---|---|
| `recipes/local_control.md` | 本地控制（Local Control）：启用、PoP、安全等级、chal_resp |
| `recipes/mqtt_topics.md` | MQTT 直发（`esp_rmaker_publish_direct`）、预算与 Basic Ingest 主题 |
| `recipes/controller_node.md` | 控制器节点 + User Helper API：一个节点经云端控制**其它**节点（list/get/set params、schedules、removenode） |
| `recipes/zigbee_gateway.md` | Zigbee 网关：运行时把加入的 Zigbee 终端动态映射为 RainMaker 设备（预共享 key / install code） |
| `recipes/thread_border_router.md` | Thread 边界路由器（TBR）：`esp_rmaker_thread_br_enable` + NAT64，让 RainMaker-over-Thread 设备经 BR 上云 |

---

## ESP32 芯片与 Claiming 支持矩阵

| 芯片 | Self Claim | Assisted Claim | 备注 |
|---|:---:|:---:|---|
| ESP32 | ❌ | ✅ | Kconfig 中 Self Claim 在 ESP32/ESP32-C2 不可用 |
| ESP32-S2 | ❌ | ❌ | 无 BT，Assisted Claim 不可用；只能 No Claim 或自定义 |
| ESP32-S3 | ✅ | ✅ | Assisted Claim 默认（BT 可用时） |
| ESP32-C2 | ❌ | ✅ | Self Claim 不可用 |
| ESP32-C3 / C5 / C6 / H2 | ✅ | ✅ | 支持 BT 的型号默认 Assisted Claim |

> 默认值见 `Kconfig.projbuild`：`default ESP_RMAKER_ASSISTED_CLAIM if BT_ENABLED && !IDF_TARGET_ESP32S2`，否则 `ESP_RMAKER_SELF_CLAIM`。

## 关键 Kconfig 速查

| Kconfig 符号 | 默认 | 说明 |
|---|---|---|
| `CONFIG_ESP_RMAKER_CLAIM_TYPE` | 0/1/2 | 0=No Claim, 1=Self, 2=Assisted |
| `CONFIG_ESP_RMAKER_CLAIM_KEY_ECDSA` | y | 默认 ECDSA P-256（可切回 RSA 2048） |
| `CONFIG_ESP_RMAKER_MQTT_USE_BASIC_INGEST_TOPICS` | y | 使用 AWS Basic Ingest 主题降低成本 |
| `CONFIG_ESP_RMAKER_MQTT_ENABLE_BUDGETING` | y | 启用 MQTT 预算限流 |
| `CONFIG_ESP_RMAKER_MQTT_DEFAULT_BUDGET` | 100 | 默认预算（64 ~ MAX_BUDGET） |
| `CONFIG_ESP_RMAKER_MQTT_MAX_BUDGET` | 1024 | 最大预算（64 ~ 2048） |
| `CONFIG_ESP_RMAKER_MAX_PARAM_DATA_SIZE` | 1024 | 参数上报 payload 最大字节数 |
| `CONFIG_ESP_RMAKER_OTA_AUTOFETCH` | y | OTA using Topics 时主动拉取 |
| `CONFIG_ESP_RMAKER_OTA_USE_HTTPS` | 默认 | OTA 协议 HTTPS / MQTT 二选一 |
| `CONFIG_ESP_RMAKER_OTA_DISABLE_AUTO_REBOOT` | n | OTA 后是否自动重启 |
| `CONFIG_ESP_RMAKER_LOCAL_CTRL_FEATURE_ENABLE` | n | 本地控制功能开关 |
| `CONFIG_ESP_RMAKER_LOCAL_CTRL_AUTO_ENABLE` | n | RainMaker 启动时自动启用本地控制 |
| `CONFIG_ESP_RMAKER_ENABLE_CHALLENGE_RESPONSE` | y | 配网时启用挑战-响应（取代传统 user mapping） |
| `CONFIG_ESP_RMAKER_CMD_RESP_ENABLE` | y | 命令-响应模块 |
| `CONFIG_ESP_RMAKER_SCHEDULING_MAX_SCHEDULES` | 10 | 最大调度数 |
| `CONFIG_ESP_RMAKER_SCENES_MAX_SCENES` | 10 | 最大场景数 |
| `CONFIG_ESP_RMAKER_NETWORK_OVER_THREAD` | n | RainMaker over Thread（依赖 OPENTHREAD_ENABLED） |

## RainMaker Agent 状态机

```
ESP_RMAKER_STATE_DEINIT
        │  esp_rmaker_node_init()
        ▼
ESP_RMAKER_STATE_INIT_DONE  ── esp_rmaker_start() ──▶  ESP_RMAKER_STATE_STARTING
                                                              │
                                            上报 node config  │
                                                              ▼
                                                ESP_RMAKER_STATE_CONFIG_REPORTED
                                                              │
                                                              ▼
                                                  ESP_RMAKER_STATE_STARTED
                                                              │
                                              esp_rmaker_stop()│
                                                              ▼
                                                ESP_RMAKER_STATE_STOP_REQUESTED
```

对应事件（`RMAKER_EVENT`）：`RMAKER_EVENT_INIT_DONE`、`RMAKER_EVENT_CLAIM_STARTED/SUCCESSFUL/FAILED`、`RMAKER_EVENT_USER_NODE_MAPPING_DONE`、`RMAKER_EVENT_LOCAL_CTRL_STARTED/STOPPED`、`RMAKER_EVENT_STARTED`、`RMAKER_EVENT_CONFIG_REPORTED`。

---

## Critical Pitfalls (Must Read)

### 1. `esp_rmaker_node_init()` 必须在所有 RainMaker API 之前

```c
// ❌ WRONG — 先创建设备再初始化节点
esp_rmaker_device_t *dev = esp_rmaker_device_create("Switch", NULL, NULL);
esp_rmaker_node_t *node = esp_rmaker_node_init(&cfg, "Node", "Switch");

// ✅ CORRECT — 先 node_init，再建设备并挂到节点
esp_rmaker_node_t *node = esp_rmaker_node_init(&cfg, "Node", "Switch");
esp_rmaker_device_t *dev = esp_rmaker_device_create("Switch", ESP_RMAKER_DEVICE_SWITCH, NULL);
esp_rmaker_node_add_device(node, dev);
```

### 2. 服务 `*_enable()` 必须在 `esp_rmaker_start()` 之前

```c
// ❌ WRONG — start 之后再 enable OTA/调度，服务不会生效
esp_rmaker_start();
esp_rmaker_ota_enable_default();
esp_rmaker_schedule_enable();

// ✅ CORRECT — 所有 enable 在 start 之前
esp_rmaker_ota_enable_default();
esp_rmaker_timezone_service_enable();
esp_rmaker_schedule_enable();
esp_rmaker_scenes_enable();
esp_rmaker_start();
```

### 3. 网络初始化顺序：app_network_init 在 node_init 之前，app_network_start 在 esp_rmaker_start 之后

```c
// ❌ WRONG — 先启动网络再初始化节点，或在 node_init 之前就 start 网络
app_network_start(POP_TYPE_RANDOM);
esp_rmaker_node_init(&cfg, "Node", "Switch");

// ✅ CORRECT — 见 examples/switch/main/app_main.c
app_network_init();
esp_rmaker_node_t *node = esp_rmaker_node_init(&cfg, "ESP RainMaker Device", "Switch");
/* ... add devices, enable services ... */
esp_rmaker_start();
app_network_start(POP_TYPE_RANDOM);
```

### 4. write 回调内不回写参数导致云端状态不一致

```c
// ❌ WRONG — 只动作硬件，云端 Power 仍为旧值
if (strcmp(name, ESP_RMAKER_DEF_POWER_NAME) == 0) {
    app_driver_set_state(val.val.b);
    // 漏掉回写
}

// ✅ CORRECT — 处理硬件后调用 esp_rmaker_param_update()
if (strcmp(name, ESP_RMAKER_DEF_POWER_NAME) == 0) {
    app_driver_set_state(val.val.b);
    esp_rmaker_param_update(param, val);
}
```

### 5. 参数值必须用 helper 构造 `esp_rmaker_param_val_t`

```c
// ❌ WRONG — 直接给结构体赋值，类型未设置
esp_rmaker_param_val_t v;
v.val.b = true;
esp_rmaker_param_update_and_report(power_param, v);

// ✅ CORRECT — 用 esp_rmaker_bool()/int()/float()/str()
esp_rmaker_param_update_and_report(power_param, esp_rmaker_bool(true));
esp_rmaker_param_update_and_report(temp_param, esp_rmaker_float(25.4));
```

### 6. 设备/服务必须 `add_device` 到节点才会上报

```c
// ❌ WRONG — 创建后忘记挂载
esp_rmaker_device_t *dev = esp_rmaker_switch_device_create("Switch", NULL, false);
esp_rmaker_device_add_cb(dev, write_cb, NULL);

// ✅ CORRECT — 最后调用 esp_rmaker_node_add_device
esp_rmaker_device_t *dev = esp_rmaker_switch_device_create("Switch", NULL, false);
esp_rmaker_device_add_cb(dev, write_cb, NULL);
esp_rmaker_node_add_device(node, dev);
```

### 7. 事件 base 用宏 `RMAKER_EVENT` / `RMAKER_COMMON_EVENT` / `RMAKER_OTA_EVENT`

```c
// ❌ WRONG — 用字符串字面量作为 event base
esp_event_handler_register("RMAKER_EVENT", ESP_EVENT_ANY_ID, ...);

// ✅ CORRECT — 这些 base 是 ESP_EVENT_DECLARE_BASE 声明的宏
esp_event_handler_register(RMAKER_EVENT, ESP_EVENT_ANY_ID, &event_handler, NULL);
esp_event_handler_register(RMAKER_COMMON_EVENT, ESP_EVENT_ANY_ID, &event_handler, NULL);
esp_event_handler_register(RMAKER_OTA_EVENT, ESP_EVENT_ANY_ID, &event_handler, NULL);
```

### 8. NVS 初始化要处理 `ESP_ERR_NVS_NO_FREE_PAGES`

```c
// ❌ WRONG — 直接 ESP_ERROR_CHECK(nvs_flash_init())
ESP_ERROR_CHECK(nvs_flash_init());

// ✅ CORRECT — 擦除后重试（见所有官方示例）
esp_err_t err = nvs_flash_init();
if (err == ESP_ERR_NVS_NO_FREE_PAGES || err == ESP_ERR_NVS_NEW_VERSION_FOUND) {
    ESP_ERROR_CHECK(nvs_flash_erase());
    err = nvs_flash_init();
}
ESP_ERROR_CHECK(err);
```

### 9. Local Control 与 on-network chal_resp 互斥

```text
# ❌ WRONG — 同时启用会冲突（共用 protocomm_httpd 单例）
CONFIG_ESP_RMAKER_LOCAL_CTRL_FEATURE_ENABLE=y
CONFIG_ESP_RMAKER_ON_NETWORK_CHAL_RESP_ENABLE=y

# ✅ CORRECT — 二选一；需要本地控制则禁用 on-network chal_resp
CONFIG_ESP_RMAKER_LOCAL_CTRL_FEATURE_ENABLE=y
# CONFIG_ESP_RMAKER_ON_NETWORK_CHAL_RESP_ENABLE is not set
```

### 10. OTA 回调返回 `ESP_FAIL` 必须先 `esp_rmaker_ota_report_status`

```c
// ❌ WRONG — 直接返回失败，云端收不到状态
esp_err_t my_ota_cb(esp_rmaker_ota_handle_t h, esp_rmaker_ota_data_t *d) {
    return ESP_FAIL; // 云端状态卡住
}

// ✅ CORRECT — 先 report 再返回（使用自定义 OTA 回调时）
esp_rmaker_ota_report_status(h, OTA_STATUS_FAILED, "download error");
return ESP_FAIL;
```

### 11. `priv_data` 生命周期必须覆盖设备整个生命周期

```c
// ❌ WRONG — priv_data 指向栈上变量，函数返回后悬空
void app_main() {
    my_ctx_t ctx = {...};
    esp_rmaker_device_create("Switch", NULL, &ctx); // ctx 出作用域即失效

// ✅ CORRECT — priv_data 用静态/堆内存
static my_ctx_t ctx = {...};          // 静态
esp_rmaker_device_create("Switch", NULL, &ctx);
```

### 12. 节点名/设备名/参数名在同一作用域内必须唯一

```c
// ❌ WRONG — 同一节点两个设备重名，或同设备两个参数重名
esp_rmaker_switch_device_create("Switch", NULL, false);
esp_rmaker_switch_device_create("Switch", NULL, false); // 重名

// ✅ CORRECT — 名称唯一
esp_rmaker_switch_device_create("Switch1", NULL, false);
esp_rmaker_switch_device_create("Switch2", NULL, false);
```

### 13. 高频上报触发 MQTT 预算耗尽导致丢消息

```c
// ❌ WRONG — 每次都 update_and_report，预算很快耗尽被丢弃
for (int i = 0; i < 1000; i++) {
    esp_rmaker_param_update_and_report(temp, esp_rmaker_float(read()));
}

// ✅ CORRECT — 用 update 合并，最后一次 report，或调大预算
esp_rmaker_param_update(temp, esp_rmaker_float(read()));
/* ... 其它参数 update ... */
esp_rmaker_report_updated_params();
```

---

## Execution Workflow

| Step | 名称 | 说明 |
|------|------|------|
| 1 | Plan | 理解需求：设备类型、参数、是否需要 OTA/调度/场景/本地控制/Thread |
| 2 | Recipe | 在 `recipes/` 找匹配场景，按其调用链执行 |
| 3 | Query | 未覆盖的 API 查 `resources/api_reference.md`；配置项查 `resources/config_reference.md` |
| 4 | Validate | 校验函数签名、头文件包含、Kconfig 符号、初始化顺序 |
| 5 | Confirm | 向用户呈现方案：includes、节点/设备/参数、enable 顺序、事件处理 |
| 6 | Execute | 新工程：以最接近的 `examples/<name>/` 为模板复制改造；已有工程：原地编辑 |
| 7 | Check | 检查：node_init 最先、enable 在 start 前、write 回调有回写、priv_data 生命周期 |
| 8 | Build | `idf.py set-target <chip>` + `idf.py build`（依赖 esp_rainmaker 组件） |
| 9 | Flash/Claim | `idf.py flash monitor`；首次烧录按 Self/Assisted Claim 流程获取云端凭据 |

### Step 6 Detail — 工程创建策略

**目标目录无现有工程（首次创建）：**

1. 根据需求选择最接近的示例作为模板：
   - 开关类 → `examples/switch/`
   - 灯具（多参数 hue/saturation/brightness）→ `examples/led_light/`
   - 风扇 → `examples/fan/`
   - 温度传感器 → `examples/temperature_sensor/`
   - 一个节点多设备 → `examples/multi_device/`
   - GPIO 控制 → `examples/gpio/`
   - 以太网 → `examples/ethernet_switch/`
   - Thread 边界路由器 → `examples/thread_br/`
   - Zigbee 网关 → `examples/zigbee_gateway/`
   - 摄像头（WebRTC/KVS）→ `examples/camera/`
   - HomeKit 集成 → `examples/homekit_switch/`
   - Matter 集成 → `examples/matter/`
   - 控制器（CLI 控制其它节点）→ `examples/rainmaker_controller/`

2. 复制整目录到用户工程目录，保留 `main/`、`sdkconfig.defaults`、`partitions.csv` 等结构。

3. 在复制的代码上修改：改设备名/类型、增删参数、改 write 回调、调 `sdkconfig.defaults`。

4. 向用户说明复制了哪个示例及原因。

**目标目录已有工程：** 原地编辑，除非用户明确要求覆盖。

---

## Failure Strategies

| 情况 | 处理 |
|---|---|
| API 在 `resources/` 找不到 | 停下并告知用户该 API 不存在，不要臆造 |
| Claiming 在 ESP32/ESP32-C2 失败 | 这两款不支持 Self Claim，改用 Assisted Claim 或 No Claim |
| MQTT 消息频繁丢失 | 检查 `CONFIG_ESP_RMAKER_MQTT_*_BUDGET`，合并上报或调大预算 |
| OTA 状态卡住 | 自定义 OTA 回调须 `esp_rmaker_ota_report_status()` 后再返回 |
| 本地控制与 on-network chal_resp 冲突 | 二者互斥（共用 protocomm_httpd），关掉其一 |
| 节点连不上云 | 先确认 claiming 已完成（`RMAKER_EVENT_CLAIM_SUCCESSFUL`），再查 MQTT host/凭据 |
| 参数云端不更新 | write 回调里漏 `esp_rmaker_param_update()`，或设备未 `add_device` |
| Thread 模式编译失败 | 需 `CONFIG_ESP_RMAKER_NETWORK_OVER_THREAD=y` 且 `OPENTHREAD_ENABLED` |

## References

- 场景 recipes → `recipes/` 目录
- API 速查 → `resources/api_reference.md`
- 配置项速查 → `resources/config_reference.md`
- 常见陷阱合集 → `resources/pitfalls.md`
- 示例工程索引 → `resources/example_list.md`
- 仓库源码 → 用户的 `esp-rainmaker/components/esp_rainmaker/include/`、`examples/`
- 官方文档 → https://rainmaker.espressif.com/docs/get-started.html
