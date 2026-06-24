---
name: esp-zigbee-sdk-skill
description: >-
  AI Skill for Espressif ESP Zigbee SDK (v2.x) firmware development. Used when users need to
  create, modify, or debug Zigbee 3.0 devices on ESP32-H2 / ESP32-C6, including Coordinator /
  Router / End Device roles, ZHA home-automation device types, ZCL clusters & commands, BDB
  commissioning (formation / steering / touchlink / finding & binding), gateway/RCP, sleepy
  end devices, OTA upgrade, and custom clusters.
  Trigger words: "esp-zigbee-sdk", "ESP Zigbee", "ezbee", "ezb_", "Zigbee", "紫蜂", "ZC", "ZR", "ZED", "ZHA", "ZCL", "BDB", "touchlink", "ESP32-H2", "ESP32-C6", "RCP gateway", "OTA upgrade"
tags:
  - embedded
  - zigbee
  - esp32
  - esp32-h2
  - esp32-c6
  - esp-idf
  - zcl
  - iot
  - firmware
  - espressif
license: Apache-2.0
compatibility: "Target: ESP32-H2 / ESP32-C6 (native 802.15.4 SoC) or ESP32-C3/S3/P4 + ESP32-H2/C6 RCP. Build: ESP-IDF v5.2+ (examples target v5.5.4), idf.py, esp-zigbee-lib >= 2.0.0 from ESP Registry"
metadata:
  author: Community
  version: "1.1.0"
---

# esp-zigbee-sdk-skill

面向 Espressif **ESP Zigbee SDK v2.x**（基于 ESP-IDF 与乐鑫自研 Zigbee 协议栈，预编译库 `esp-zigbee-lib`）的 AI 开发技能。提供基于仓库真实文档与代码的 Recipes、API 速查、配置参考、状态机与陷阱清单，用于在 ESP32-H2 / ESP32-C6 等芯片上开发 Zigbee 3.0（ZC/ZR/ZED）设备、网关、休眠终端与 OTA。所有函数名、结构体、宏、Kconfig、文件路径均来自仓库，不存在则不写入。

> 重要：本技能面向 **v2.x 主线**（`ezb_` / `ezbee/` 头文件 API，仓库 `main` 分支）。v1.x（ZBOSS、`esp_zb_` API）为 LTS 维护分支，请参考仓库 `release/v1.0` 与迁移指南。

## Core Principles

1. **绝不臆造 API** — 先查 `resources/api_reference.md`；找不到对应签名即视为不存在。v2.x 的活动 API 前缀为 `ezb_`（头文件在 `components/esp-zigbee-lib/include/ezbee/`）。
2. **主任务结构固定** — Zigbee 必须运行在独立 FreeRTOS 任务里，调用顺序为 `esp_zigbee_init` → 配置/建数据模型 → `esp_zigbee_start(false)` → `esp_zigbee_launch_mainloop()`（不返回）→ `esp_zigbee_deinit` → `vTaskDelete`。
3. **`esp_zigbee_start(false)` 是 no-autostart 模式** — 协议栈框架启动完成会回调 `EZB_ZDO_SIGNAL_SKIP_STARTUP`，应用须在此调用 `ezb_bdb_start_top_level_commissioning(EZB_BDB_MODE_INITIALIZATION)`，再由 `EZB_BDB_SIGNAL_DEVICE_FIRST_START` 驱动组网/入网。
4. **应用信号处理器是事件中枢** — 通过 `ezb_app_signal_add_handler()` 注册的 `ezb_app_signal_handler_t` 回调，处理 ZDO/BDB/NWK 全部信号；用 `ezb_app_signal_get_type()` 取类型、`ezb_app_signal_get_params()` 取载荷。
5. **ZCL 数据模型构建顺序** — `ezb_af_create_device_desc()` → `ezb_zha_create_<device>(ep_id, &cfg)` 得到 endpoint desc → （可选）`ezb_af_endpoint_get_cluster_desc` + `ezb_zcl_basic_cluster_desc_add_attr` 补 Basic 属性 → `ezb_af_device_add_endpoint_desc` → `ezb_af_device_desc_register`。
6. **线程安全靠全局锁** — 除"Zigbee 任务主循环回调内"与"`esp_zigbee_task_queue_post` 投递的回调内"两种情形外，**任何**调用 Zigbee API 前必须 `esp_zigbee_lock_acquire(portMAX_DELAY)`，之后 `esp_zigbee_lock_release()`。
7. **设备角色由 `esp_zigbee_config_t.device_config.device_type` 决定** — `EZB_NWK_DEVICE_TYPE_COORDINATOR/ROUTER/END_DEVICE`（`ezbee/nwk.h`）。ZC/ZR 用 `zczr_config.max_children`，ZED 用 `zed_config.ed_timeout` + `keep_alive`。
8. **BDB 模式必须与角色匹配** — 只有 ZC/ZR 能 `EZB_BDB_MODE_NETWORK_FORMATION`；ZED/ZR 入网用 `EZB_BDB_MODE_NETWORK_STEERING`；Touchlink 用 `EZB_BDB_MODE_TOUCHLINK_INITIATOR/TARGET`。
9. **属性变更走 ZCL Core Action 回调** — 注册 `ezb_zcl_core_action_handler_register()`，在 `EZB_ZCL_CORE_SET_ATTR_VALUE_CB_ID` 里取 `ezb_zcl_set_attr_value_message_t` 驱动外设（如点灯）。
10. **休眠终端必须关 RxOnWhenIdle** — `ezb_nwk_set_rx_on_when_idle(false)`，并在 sdkconfig 开 `CONFIG_PM_ENABLE=y`、`CONFIG_FREERTOS_USE_TICKLESS_IDLE=y`；网关/共存场景需另行配置。
11. **RCP 网关用 UART 模式** — 非 802.15.4 SoC（如 ESP32-C3/S3/P4）须 `ESP_ZIGBEE_RADIO_MODE_UART_RCP` 指定 UART 引脚与 460800 波特率，并烧录 ESP32-H2/C6 的 `ot_rcp` 固件。
12. **存储分区不可省** — `partitions.csv` 必须包含 `zb_storage`（data/nvs/16K）与 `zb_fct`（data/fat/1K），并在 `app_main` 调 `nvs_flash_init_partition(ESP_ZIGBEE_STORAGE_PARTITION_NAME)`。
13. **Deep sleep 与 light sleep 是两套流程** — deep sleep 由应用直接调 `esp_deep_sleep_start()`（不需要 `CONFIG_PM_ENABLE`），每次唤醒从 reset 重启、协议栈重新 init 并自动 rejoin 之前的网络（`EZB_BDB_SIGNAL_DEVICE_REBOOT`）；唤醒源配置（`esp_sleep_enable_timer_wakeup` + `esp_sleep_enable_ext1_wakeup`）必须在 `app_main` 任何 Zigbee 调用之前完成。跨睡眠数据须用 `RTC_DATA_ATTR`。周期 > 30 分钟才优选 deep sleep，否则用 light sleep。

## When to Use

**Applicable（适用）:**
- 在 ESP32-H2 / ESP32-C6 上创建 Zigbee 3.0 设备（Coordinator / Router / End Device / Sleepy End Device）
- 实现 ZHA 标准设备（on/off light、dimmable/color light、switch、temperature sensor、thermostat、door lock、window covering 等）
- 发送/接收 ZCL 命令与属性（on/off、level、color_control、temperature_measurement、groups、scenes、identify）
- BDB commissioning：网络形成（formation）、网络引导（steering 入网）、发现与绑定（finding & binding）、Touchlink
- 构建 Zigbee Gateway（Wi-Fi/Ethernet + RCP）或 Radio Co-Processor（RCP）
- Zigbee OTA 升级（ota_server / ota_client）
- 自定义 cluster / 自定义设备（customized devices / data stream）
- 从 v1.x（ZBOSS, `esp_zb_`）迁移到 v2.x（`ezb_`）

**Not applicable（不适用）:**
- BLE / Thread / Matter 开发（使用 esp-matter / openthread）
- 纯 Wi-Fi / 以太网应用（ESP-IDF 本身）
- 非 ESP32 系列芯片的 Zigbee
- PCB / 硬件原理图设计
- v1.x ZBOSS API 的深度实现细节（仅提供迁移指引）

---

## Scenario Quick Reference (Recipes)

当用户意图匹配下方场景时，**先阅读对应 recipe**——其中包含完整调用链、分步说明、常见错误与可复制代码。

### 设备角色与组网

| recipe | scenario |
|---|---|
| `recipes/coordinator_light.md` | ZC 角色：on/off light，形成网络 + 开放入网（formation → steering） |
| `recipes/router_switch.md` | ZR/ZED 角色：on/off switch，steering 入网 + ZDO 发现/绑定灯 |
| `recipes/sleepy_end_device.md` | Sleepy End Device：light sleep、RxOnWhenIdle=false、按键唤醒 |
| `recipes/deep_sleep_end_device.md` | Deep Sleep End Device：深睡断电、RTC timer + EXT1 唤醒、wake-from-reset、唤醒后 rejoin 网络 |

### ZHA 设备与 ZCL

| recipe | scenario |
|---|---|
| `recipes/zha_device_model.md` | 构建 ZHA 数据模型：device desc → endpoint → cluster → Basic 属性 |
| `recipes/zcl_attribute_report.md` | 属性上报：`ezb_zcl_set_attr_value` 写本地属性 + `ezb_zcl_report_attr_cmd_req` 主动上报 |
| `recipes/zcl_command_send.md` | 发送 ZCL 命令（on/off toggle、level move-to、color、通用 read/write/report） |
| `recipes/zcl_core_action.md` | 接收属性写入：`ezb_zcl_core_action_handler_register` + `EZB_ZCL_CORE_SET_ATTR_VALUE_CB_ID` |

### 高级场景

| recipe | scenario |
|---|---|
| `recipes/zigbee_gateway_rcp.md` | Zigbee 网关：主芯片 + RCP（UART），coex，RCP 自动升级 |
| `recipes/touchlink.md` | Touchlink commissioning：initiator 与 target |
| `recipes/ota_upgrade.md` | Zigbee OTA：ota_server 下发 + ota_client 接收刷写 |
| `recipes/custom_cluster.md` | 自定义 cluster：注册 handlers、自定义命令收发 |

---

## 关键参考表

### 芯片与角色支持

| 芯片 | 802.15.4 SoC | Zigbee 角色 | 典型用法 |
|---|---|---|---|
| ESP32-H2 | 是（native） | ZC / ZR / ZED / Sleepy / RCP | 直接做 Zigbee 设备，或作为网关的 RCP |
| ESP32-C6 | 是（native） | ZC / ZR / ZED / Sleepy / RCP | Wi-Fi 6 + 15.4 共存，可单芯片网关 |
| ESP32-C3 / S3 / P4 | 否 | 网关主控（+ RCP） | 经 UART 连接 H2/C6 RCP |
| ESP32（原版） | 否 | 网关主控（+ RCP） | 经 UART 连接 RCP |

> 仅 `CONFIG_SOC_IEEE802154_SUPPORTED=y` 的芯片可用 `ESP_ZIGBEE_RADIO_MODE_NATIVE`，否则必须 `ESP_ZIGBEE_RADIO_MODE_UART_RCP`。

### 网络设备类型（`ezbee/nwk.h`）

| 枚举 | 值 | 含义 |
|---|---|---|
| `EZB_NWK_DEVICE_TYPE_COORDINATOR` | 0x0 | 协调器，可形成网络（Trust Center） |
| `EZB_NWK_DEVICE_TYPE_ROUTER` | 0x1 | 路由器，可入网并转发 |
| `EZB_NWK_DEVICE_TYPE_END_DEVICE` | 0x2 | 终端，可入网、可休眠 |
| `EZB_NWK_DEVICE_TYPE_NONE` | 0x3 | 未知 |

### BDB commissioning 模式与信号（`ezbee/bdb.h` / `ezbee/app_signals.h`）

| 模式（`ezb_bdb_comm_mode_t`） | 值 | 触发信号 | 适用角色 |
|---|---|---|---|
| `EZB_BDB_MODE_INITIALIZATION` | 0x01 | `EZB_BDB_SIGNAL_DEVICE_FIRST_START` / `EZB_BDB_SIGNAL_DEVICE_REBOOT` | 全部 |
| `EZB_BDB_MODE_TOUCHLINK_INITIATOR` | 0x02 | `EZB_BDB_SIGNAL_TOUCHLINK_INITIATOR_FINISHED` | initiator |
| `EZB_BDB_MODE_NETWORK_STEERING` | 0x04 | `EZB_BDB_SIGNAL_STEERING` | ZR/ZED 入网 |
| `EZB_BDB_MODE_NETWORK_FORMATION` | 0x08 | `EZB_BDB_SIGNAL_FORMATION` | ZC/ZR 形成网络 |
| `EZB_BDB_MODE_FINDING_N_BINDING` | 0x10 | `EZB_BDB_SIGNAL_FINDING_AND_BINDING_INITIATOR/TARGET_FINISHED` | initiator/target |
| `EZB_BDB_MODE_TOUCHLINK_TARGET` | 0x20 | `EZB_BDB_SIGNAL_TOUCHLINK_TARGET_FINISHED` | target |

### 信道掩码（2.4GHz，信道 11–26）

| 宏/值 | 含义 |
|---|---|
| `0x00000800` | 仅信道 11 |
| `(1U << 13)` | 仅信道 13（示例默认主信道） |
| `0x07FFF800` | 全部信道 11–26（示例默认次信道） |
| 有效范围 | `0x00000800` – `0x07FFF800` |

### ZCL Core Action 回调 ID（节选，`ezbee/zcl/zcl_core.h`）

| ID | 消息结构 | 说明 |
|---|---|---|
| `EZB_ZCL_CORE_SET_ATTR_VALUE_CB_ID` | `ezb_zcl_set_attr_value_message_t` | 属性被设置（点灯入口） |
| `EZB_ZCL_CORE_DEFAULT_RSP_CB_ID` | `ezb_zcl_cmd_default_rsp_message_t` | 默认响应 |
| `EZB_ZCL_CORE_REPORT_ATTR_CB_ID` | — | 收到属性上报 |
| `EZB_ZCL_CORE_READ_ATTR_RSP_CB_ID` | — | 读属性响应 |
| `EZB_ZCL_CORE_MANUF_SPEC_CMD_CB_ID` | — | 厂商自定义命令 |

---

## Critical Pitfalls (Must Read)

下面是最常见、会导致固件不可用的错误，每条给出错误写法与正确写法。

### 1. Zigbee API 调用前未加锁

```c
// ❌ WRONG — 从应用任务直接调用 ZCL 命令，未持锁
void button_handler(void) {
    ezb_zcl_on_off_toggle_cmd_req(&cmd_req);  // 协议栈可能正在主循环中，竞态
}

// ✅ CORRECT — acquire/release 包裹（回调主循环内除外）
void button_handler(void) {
    esp_zigbee_lock_acquire(portMAX_DELAY);
    ezb_zcl_on_off_toggle_cmd_req(&cmd_req);
    esp_zigbee_lock_release();
}
```

### 2. 主任务没有 `esp_zigbee_launch_mainloop()` 或顺序错误

```c
// ❌ WRONG — 缺少 mainloop，协议栈不会运转；或 init/start 顺序颠倒
static void zigbee_task(void *pv) {
    esp_zigbee_start(false);
    esp_zigbee_init(&config);   // 顺序错误
}

// ✅ CORRECT — 标准 init → 配置 → start → mainloop → deinit → delete
static void esp_zigbee_stack_main_task(void *pvParameters) {
    esp_zigbee_config_t config = ESP_ZIGBEE_DEFAULT_CONFIG();
    ESP_ERROR_CHECK(esp_zigbee_init(&config));
    ESP_ERROR_CHECK(esp_zigbee_setup_commissioning());      // 信道/信号处理器
    ESP_ERROR_CHECK(esp_zigbee_create_zha_on_off_light_device()); // 数据模型
    ESP_ERROR_CHECK(esp_zigbee_start(false));
    esp_zigbee_launch_mainloop();   // 阻塞，不返回
    esp_zigbee_deinit();
    vTaskDelete(NULL);
}
```

### 3. 漏掉 `EZB_ZDO_SIGNAL_SKIP_STARTUP` → BDB 永不初始化

```c
// ❌ WRONG — 没有处理 SKIP_STARTUP，栈框架启动后无人触发 BDB
static bool app_signal_handler(const ezb_app_signal_t *sig) {
    switch (ezb_app_signal_get_type(sig)) {
    case EZB_BDB_SIGNAL_DEVICE_FIRST_START: /* ... */ break;
    /* 缺少 SKIP_STARTUP：FIRST_START 永远不会到来 */
    }
    return true;
}

// ✅ CORRECT — SKIP_STARTUP 触发 INITIALIZATION
case EZB_ZDO_SIGNAL_SKIP_STARTUP:
    ESP_LOGI(TAG, "Initialize Zigbee stack");
    ezb_bdb_start_top_level_commissioning(EZB_BDB_MODE_INITIALIZATION);
    break;
```

### 4. ZC 用了 steering 而非 formation（或反之）

```c
// ❌ WRONG — ZC 工厂新设备直接 steering，网络都还没形成
if (ezb_bdb_is_factory_new()) {
    ezb_bdb_start_top_level_commissioning(EZB_BDB_MODE_NETWORK_STEERING);
}

// ✅ CORRECT — ZC：factory-new 先 FORMATION 再 STEERING；非 factory-new 直接开放网络
if (ezb_bdb_is_factory_new()) {
    ezb_bdb_start_top_level_commissioning(EZB_BDB_MODE_NETWORK_FORMATION);
} else {
    ezb_bdb_open_network(180);
}
// 在 EZB_BDB_SIGNAL_FORMATION 成功分支里再启动 STEERING
```

### 5. ZED 用了 NETWORK_FORMATION

```c
// ❌ WRONG — 终端设备尝试形成网络，必然 EZB_BDB_STATUS_NOT_PERMITTED
if (ezb_bdb_is_factory_new()) {
    ezb_bdb_start_top_level_commissioning(EZB_BDB_MODE_NETWORK_FORMATION);
}

// ✅ CORRECT — ZED/ZR 入网只能用 STEERING
if (ezb_bdb_is_factory_new()) {
    ezb_bdb_start_top_level_commissioning(EZB_BDB_MODE_NETWORK_STEERING);
}
```

### 6. 设备类型配置与角色不符

```c
// ❌ WRONG — 想做 switch（终端）却填 COORDINATOR
#define ESP_ZIGBEE_ZED_CONFIG() { \
    .device_type = EZB_NWK_DEVICE_TYPE_COORDINATOR, \  // 错
    .zed_config = { .ed_timeout = EZB_NWK_ED_TIMEOUT_64MIN, .keep_alive = 4000 }, }

// ✅ CORRECT — 终端用 END_DEVICE
#define ESP_ZIGBEE_ZED_CONFIG() { \
    .device_type = EZB_NWK_DEVICE_TYPE_END_DEVICE, \
    .install_code_policy = false, \
    .zed_config = { .ed_timeout = EZB_NWK_ED_TIMEOUT_64MIN, .keep_alive = 4000 }, }
```

### 7. 数据模型未 register / 顺序错

```c
// ❌ WRONG — 创建了 ep_desc 但没加进 device_desc，或没 register
ezb_af_ep_desc_t ep = ezb_zha_create_on_off_light(EP_ID, &cfg);
ezb_zcl_core_action_handler_register(handler);  // 栈里根本没这个 endpoint

// ✅ CORRECT — create_device_desc → add_endpoint_desc → register
ezb_af_device_desc_t dev = ezb_af_create_device_desc();
ezb_af_ep_desc_t    ep   = ezb_zha_create_on_off_light(EP_ID, &cfg);
/* 可选：补 Basic 厂商/型号属性 */
ESP_ERROR_CHECK(ezb_af_device_add_endpoint_desc(dev, ep));
ESP_ERROR_CHECK(ezb_af_device_desc_register(dev));
ezb_zcl_core_action_handler_register(handler);
```

### 8. 缺少 `zb_storage` / `zb_fct` 分区

```text
# ❌ WRONG — 使用 ESP-IDF 默认 partitions.csv，缺 Zigbee 存储分区
nvs,      data, nvs, , 0x4000,
phy_init, data, phy, , 0x1000,
factory,  app,  factory, , 1M,

# ✅ CORRECT — 必须含 zb_storage 与 zb_fct
# Name,   Type, SubType, Offset, Size, Flags
nvs,        data, nvs,      , 0x6000,
otadata,    data, ota,      , 0x2000,
phy_init,   data, phy,      , 0x1000,
factory,    app,  factory,  , 940K,
zb_storage, data, nvs,      , 16K,
zb_fct,     data, fat,      , 1K,
```

### 9. `app_main` 未初始化 zb_storage NVS 分区

```c
// ❌ WRONG — 只初始化默认 nvs，Zigbee 持久化数据无分区可用
void app_main(void) {
    nvs_flash_init();
    xTaskCreate(esp_zigbee_stack_main_task, "Zigbee_main", 4096, NULL, 5, NULL);
}

// ✅ CORRECT — 同时初始化专用分区
void app_main(void) {
    ESP_ERROR_CHECK(nvs_flash_init());
    ESP_ERROR_CHECK(nvs_flash_init_partition(ESP_ZIGBEE_STORAGE_PARTITION_NAME)); // "zb_storage"
    xTaskCreate(esp_zigbee_stack_main_task, "Zigbee_main", 4096, NULL, 5, NULL);
}
```

### 10. RCP 网关在非 15.4 SoC 上用 NATIVE 模式

```c
// ❌ WRONG — ESP32-C3 没有 15.4 radio，用 NATIVE 会编译告警且无法运行
#define ESP_ZIGBEE_PLATFORM_CONFIG() { \
    .radio_config = { .radio_mode = ESP_ZIGBEE_RADIO_MODE_NATIVE }, }

// ✅ CORRECT — 非 15.4 SoC 用 UART_RCP，并指定 UART 引脚/波特率
#define ESP_ZIGBEE_UART_CONFIG() { \
    .port = 1, \
    .uart_config = { .baud_rate = 460800, .data_bits = UART_DATA_8_BITS, \
                     .parity = UART_PARITY_DISABLE, .stop_bits = UART_STOP_BITS_1, \
                     .flow_ctrl = UART_HW_FLOWCTRL_DISABLE, .rx_flow_ctrl_thresh = 0, \
                     .source_clk = UART_SCLK_DEFAULT }, \
    .rx_pin = CONFIG_PIN_TO_RCP_TX, .tx_pin = CONFIG_PIN_TO_RCP_RX, }
#define ESP_ZIGBEE_PLATFORM_CONFIG() { \
    .storage_partition_name = ESP_ZIGBEE_STORAGE_PARTITION_NAME, \
    .radio_config = { .radio_mode = ESP_ZIGBEE_RADIO_MODE_UART_RCP, \
                      .radio_uart_config = ESP_ZIGBEE_UART_CONFIG() }, }
```

### 11. Sleepy 终端未关闭 RxOnWhenIdle / 未开 PM

```c
// ❌ WRONG — 终端保持 RxOnWhenIdle=true，永不休眠，功耗高
esp_err_t esp_zigbee_setup_commissioning(void) {
    ezb_bdb_set_primary_channel_set(ESP_ZIGBEE_PRIMARY_CHANNEL_MASK);
    ezb_app_signal_add_handler(handler);
    return ESP_OK;
}

// ✅ CORRECT — 终端关 RxOnWhenIdle，sdkconfig 开 CONFIG_PM_ENABLE + CONFIG_FREERTOS_USE_TICKLESS_IDLE
ezb_nwk_set_rx_on_when_idle(false);
```

### 12. 属性上报用错方向（client 发 report 时 dst_addr_mode=NONE 未绑定）

```c
// ❌ WRONG — switch 未绑定时用 NONE 地址上报，命令无处可去
ezb_zcl_report_attr_cmd_t cmd = {
    .cmd_ctrl = { .dst_addr.addr_mode = EZB_ADDR_MODE_NONE,
                  .src_ep = SWITCH_EP, .cluster_id = EZB_ZCL_CLUSTER_ID_ON_OFF },
};

// ✅ CORRECT — 先完成 ZDO bind，再以 NONE 地址由绑定表路由；或显式指定 SHORT/GROUP 地址
// 已绑定的 switch 上报当前属性（temperature_sensor 示例做法）：
ezb_zcl_report_attr_cmd_t report_attr_cmd = {
    .cmd_ctrl = { .fc.direction       = EZB_ZCL_CMD_DIRECTION_TO_CLI,
                  .dst_addr.addr_mode = EZB_ADDR_MODE_NONE,
                  .src_ep             = ESP_ZIGBEE_HA_TEMPERATURE_SENSOR_EP_ID,
                  .cluster_id         = EZB_ZCL_CLUSTER_ID_TEMPERATURE_MEASUREMENT },
    .payload  = { .attr_id = EZB_ZCL_ATTR_TEMPERATURE_MEASUREMENT_MEASURED_VALUE_ID },
};
esp_zigbee_lock_acquire(portMAX_DELAY);
ezb_zcl_report_attr_cmd_req(&report_attr_cmd);
esp_zigbee_lock_release();
```

### 13. commissioning 失败后不重试

```c
// ❌ WRONG — formation/steering 失败后什么都不做，设备卡死
case EZB_BDB_SIGNAL_FORMATION: {
    ezb_bdb_comm_status_t status = *(...*)ezb_app_signal_get_params(app_signal);
    if (status != EZB_BDB_STATUS_SUCCESS) {
        ESP_LOGW(TAG, "failed");   // 没有重试
    }
} break;

// ✅ CORRECT — 用 alarm_timer 延迟重试（仓库示例统一模式）
case EZB_BDB_SIGNAL_FORMATION: {
    ezb_bdb_comm_status_t status = *(ezb_bdb_comm_status_t *)ezb_app_signal_get_params(app_signal);
    if (status == EZB_BDB_STATUS_SUCCESS) {
        ezb_bdb_start_top_level_commissioning(EZB_BDB_MODE_NETWORK_STEERING);
    } else {
        ESP_LOGW(TAG, "Failed to form network with status(0x%02x)", status);
        alarm_timer_schedule(esp_zigbee_alarm_bdb_commissioning, EZB_BDB_MODE_NETWORK_FORMATION, 1000);
    }
} break;
```

### 14. 主任务栈太小

```c
// ❌ WRONG — Zigbee 主任务 2KB 栈，运行期可能栈溢出/断言
xTaskCreate(esp_zigbee_stack_main_task, "zb", 2048, NULL, 5, NULL);

// ✅ CORRECT — 示例统一 4096；可用 uxTaskGetStackHighWaterMark 监控
xTaskCreate(esp_zigbee_stack_main_task, "Zigbee_main", 4096, NULL, 5, NULL);
```

---

## Execution Workflow

| Step | Name | Description |
|------|------|-------------|
| 1 | Plan | 明确设备角色（ZC/ZR/ZED/Sleepy）、ZHA 设备类型、所需 cluster、是否 RCP 网关、是否 OTA |
| 2 | Recipe | 匹配 `recipes/` 中场景，先读对应 recipe 的调用链与代码 |
| 3 | Query | recipe 未覆盖的 API，查 `resources/api_reference.md`；配置项查 `resources/config_reference.md` |
| 4 | Validate | 校验：设备类型、BDB 模式与角色匹配；分区表；NVS 初始化；锁；mainloop 顺序 |
| 5 | Confirm | 向用户确认：芯片目标、信道掩码、endpoint ID、ZHA 设备类型、外设绑定 |
| 6 | Execute | **新项目：** 复制最接近的 `examples/<...>` 到工作目录后修改（推荐）。**已有项目：** 原地编辑 |
| 7 | Check | 检查：锁包裹、SKIP_STARTUP 处理、formation/steering 分支、RxOnWhenIdle、UART/RCP 引脚 |
| 8 | Build | `idf.py set-target esp32h2`（或 esp32c6）→ `idf.py build` |
| 9 | Flash | `idf.py -p PORT erase_flash flash monitor`（首次必须 erase_flash） |
| 10 | Debug | 串口日志 + Wireshark（Pyspinel + ot_rcp 抓包，默认 key `ZigbeeAlliance09`）；可开 `ZB_DEBUG_MODE` |

### Step 6 Detail — 项���创建策略

**目标目录无项目时（首次创建）：**
1. 按需求选最接近的示例（基于 `examples/`）：
   - ZC on/off 灯 → `examples/home_automation_devices/on_off_light/`
   - ZR/ZED 开关 + 发现绑定 → `examples/home_automation_devices/on_off_switch/`
   - 调光/彩光 → `examples/home_automation_devices/color_dimmable_light/`、`color_dimmer_switch/`
   - 温度传感器/温控 → `examples/home_automation_devices/temperature_sensor/`、`thermostat/`
   - 休眠终端 → `examples/sleepy_devices/light_sleep_end_device/`、`deep_sleep_end_device/`
   - 网关/RCP → `examples/zigbee_gateway/`
   - Touchlink → `examples/touchlink/touchlink_initiator/`、`touchlink_target/`
   - OTA → `examples/ota_upgrade/ota_server/`、`ota_client/`
   - 自定义 cluster/设备 → `examples/customized_devices/data_producer/`、`data_consumer/`
   - 全设备类型 → `examples/all_device_types_app/`
2. 复制整个示例目录（含 `main/`、`partitions.csv`、`sdkconfig.defaults`、`idf_component.yml`）。
3. 修改 `main/<example>.h` 中的 `ESP_ZIGBEE_*_CONFIG()`、信道掩码、endpoint ID、`ESP_MANUFACTURER_NAME` / `ESP_MODEL_IDENTIFIER`。
4. 用 `idf_component.yml` 声明依赖（`espressif/esp-zigbee-lib` 版本 `>=2.0.0`，IDF `>=5.2`）。

**目标目录已有项目时：** 原地编辑，不要覆盖除非用户明确要求。

---

## Failure Strategies

| Situation | Action |
|---|---|
| API 在 `resources/` 找不到 | 停下并告知用户；不要臆造。确认是否 v1.x 的 `esp_zb_` API（需迁移或用 `compat/` 层） |
| 编译报错找不到 `esp_zigbee.h` | 检查 `idf_component.yml` 是否声明 `espressif/esp-zigbee-lib`；运行 `idf.py reconfigure` |
| `EZB_BDB_STATUS_NOT_PERMITTED` | 角色与 BDB 模式不匹配（如 ZED 用了 FORMATION），改为匹配模式 |
| 入网失败反复 | 用 `alarm_timer_schedule` 延迟 1000ms 重试；检查主信道掩码、PAN、是否有可用 ZC |
| 设备上线即掉线 | 终端检查 keep_alive（建议 4000ms）与 ed_timeout；路由检查供电；用 `ezb_nwk_get_next_neighbor` |
| 抓包看不到加密内容 | Wireshark 添加预配置 key `5A:69:67:42:65:65:41:6C:6C:69:61:6E:63:65:30:39` |
| 断言失败且栈在库内 | 收集完整串口日志 + `build/*.elf`，按 `developing.rst` 提 GitHub issue |
| RCP 不通信 | 复查 UART 引脚（`CONFIG_PIN_TO_RCP_TX/RX`）、波特率 460800、RCP 固件已烧录 |
| 想用 v1.x 旧 API | 参考 `docs/en/migration-guide/v2.x/` 迁移；或直接用 `compat/` 兼容头 |

## References

- 场景 Recipes → `recipes/` 目录
- API 速查 → `resources/api_reference.md`
- 配置参考（Kconfig / sdkconfig / 分区 / idf_component.yml） → `resources/config_reference.md`
- 陷阱汇总 → `resources/pitfalls.md`
- 示例索引 → `resources/example_list.md`
- 仓库原始文档 → `espressif-repos/esp-zigbee-sdk/docs/en/`（`introduction.rst`、`developing.rst`、`faq.rst`、`api-reference/`）
