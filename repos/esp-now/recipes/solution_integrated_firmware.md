# 多特性集成固件（综合方案）

> **适用摘要**: 用条件编译在单个二进制中同时集成 Wi-Fi 配网（BLE/SoftAP）、ESP-NOW 配网、设备控制、无线调试、批量 OTA、安全握手与时间同步；通过 `CONFIG_APP_ESPNOW_INITIATOR`/`RESPONDER` 切换角色，单按键复用配网/绑定/控制/复位，共享 LED 表达状态（参考 `examples/solution`）。

> Version: selected repo/component version; align with the user project and dependency manifest.
> Validation: draft metadata added from repository routing; verify APIs, Kconfig symbols, and component dependencies against the selected version.

## 触发意图

- "ESP-NOW 综合方案"
- "多特性集成"
- "一个固件集成所有功能"
- "product firmware"
- "CONFIG_APP_ESPNOW_INITIATOR"
- "solution 示例"
- "按键复用 配网 绑定 控制"

## 前置条件

| 条件 | 要求 |
|---|---|
| Kconfig | `Example Configuration` 下按需开启 `CONFIG_APP_ESPNOW_CONTROL`/`_DEBUG`/`_OTA`/`_SECURITY`/`_PROVISION`/`_TIMESYNC`；`APP_ESPNOW_SOLUTION_MODE` 选 `Initiator Mode` 或 `Responder Mode` |
| 自定义组件 | `components/wifi_prov`（initiator 上层 Wi-Fi 配网）、`components/espnow_device`（initiator/responder 封装） |
| 头文件 | `espnow.h`、`espnow_ctrl.h`、`espnow_prov.h`、`espnow_security.h`、`espnow_security_handshake.h`、`espnow_console.h`、`espnow_log.h`、`espnow_ota.h`、`espnow_time.h`、`iot_button.h`、`led_strip.h`、`network_provisioning/manager.h` |
| 硬件 | ≥2 块 ESP32 系列开发板；1 个按键（`WIFI_PROV_KEY_GPIO=GPIO_NUM_2`、`CONTROL_KEY_GPIO=GPIO_NUM_9`(C2/C3) 或 `GPIO_NUM_0`）；1 颗 RGB LED（`LED_STRIP_GPIO`，C2 为三路 GPIO） |
| 参考示例 | `examples/solution/main/app_main.c`、`examples/solution/components/espnow_device/{initiator.c,responder.c}` |

## 分步说明

### 1. 角色选择与特性开关（Kconfig）

`examples/solution/main/Kconfig.projbuild` 用一个 choice 决定整个二进制的角色，并 `select` 出强约束：

```
choice APP_ESPNOW_SOLUTION_MODE
    config APP_ESPNOW_INITIATOR
        bool "Initiator Mode"
        select APP_WIFI_PROVISION     # initiator 强制带 Wi-Fi 配网
    config APP_ESPNOW_RESPONDER
        bool "Responder Mode"
endchoice
```

`menuconfig → Example Configuration` 中可独立开关的特性：

| Kconfig | 默认 | 作用 |
|---|---|---|
| `CONFIG_APP_ESPNOW_CONTROL` | y | 控制绑定/发送 |
| `CONFIG_APP_ESPNOW_DEBUG` | y | console + log 抓取 |
| `CONFIG_APP_ESPNOW_OTA` | y | responder 接收 OTA |
| `CONFIG_APP_ESPNOW_SECURITY` | y | 安全握手 + 加密 |
| `CONFIG_APP_ESPNOW_PROVISION` | y | ESP-NOW 配网 |
| `CONFIG_APP_ESPNOW_TIMESYNC` | n | 节点时间同步 |
| `CONFIG_APP_WIFI_PROVISION` | y(initiator) | BLE/SoftAP 上层配网 |
| `CONFIG_APP_ESPNOW_QUEUE_SIZE` | 32 | 发送队列，节点多时调大（如 100） |
| `CONFIG_APP_ESPNOW_SESSION_POP` | "espnow_pop" | 安全握手 PoP |
| `CONFIG_APP_ESPNOW_TIMESYNC_INTERVAL_MS` | 60000 | initiator 广播间隔 |
| `CONFIG_APP_ESPNOW_TIMESYNC_MAX_DRIFT_MS` | 100 | responder 漂移阈值 |

> `APP_WIFI_PROVISION` 子菜单可选 BLE 或 SoftAP transport；initiator 选 BLE 时 Kconfig 自动 `select BT_ENABLED`。

### 2. app_main：固定初始化顺序 + sec_enable

```c
void app_main()
{
    espnow_storage_init();                 // 1. NVS

    app_wifi_init();                       // 2. Wi-Fi STA start（或 wifi_prov_init）

    espnow_config_t espnow_config = ESPNOW_INIT_CONFIG_DEFAULT();
    espnow_config.qsize = CONFIG_APP_ESPNOW_QUEUE_SIZE;
#ifdef CONFIG_APP_ESPNOW_SECURITY
    espnow_config.sec_enable = 1;          // 3. 仅打开加密能力
#endif
    espnow_init(&espnow_config);           // 4. espnow_init 必须在 Wi-Fi start 之后

    app_led_init();                        // 5. 共享 LED

#if defined(CONFIG_APP_WIFI_PROVISION) || defined(CONFIG_APP_ESPNOW_PROVISION)
    app_wifi_prov_button_init();           // 6. 配网按键（GPIO2）
#endif

#if CONFIG_APP_ESPNOW_INITIATOR
    app_espnow_initiator_register();       // 7a. 注册事件、创建 event group
#ifdef CONFIG_APP_ESPNOW_CONTROL
    app_control_button_init();             //    控制按键（GPIO9/0）
#endif
#ifdef CONFIG_APP_ESPNOW_TIMESYNC
    espnow_time_initiator_config_t tc = {
        .sync_interval_ms = CONFIG_APP_ESPNOW_TIMESYNC_INTERVAL_MS,
    };
    espnow_time_initiator_start(&tc);
#endif
    app_espnow_initiator();                //    启动 sec/console/prov 任务
#elif CONFIG_APP_ESPNOW_RESPONDER
    app_espnow_responder_register();
#ifdef CONFIG_APP_ESPNOW_CONTROL
    app_control_responder_init();
#endif
#ifdef CONFIG_APP_ESPNOW_TIMESYNC
    espnow_time_responder_config_t tc = {
        .max_drift_ms = CONFIG_APP_ESPNOW_TIMESYNC_MAX_DRIFT_MS,
    };
    esp_event_handler_register(ESP_EVENT_ESPNOW, ESP_EVENT_ESPNOW_TIMESYNC_SYNCED,
                               app_timesync_event_handler, NULL);
    espnow_time_responder_start(&tc);
    espnow_time_responder_request();
#endif
    app_espnow_responder();
#endif
}
```

> `app_wifi_init()` 内部分支：`CONFIG_APP_WIFI_PROVISION` 时调 `wifi_prov_init()`（自定义组件封装 BLE/SoftAP + `network_prov_mgr`），否则直接 `esp_wifi_set_mode(STA)`/`set_ps(WIFI_PS_NONE)`/`start()`。

### 3. 关键约束：安全握手必须先于加密配网/控制

initiator 的 `app_espnow_initiator()` 先 `xTaskCreate(app_espnow_initiator_sec_task,...)` 完成握手、分发 app key。responder 侧的 ESP-NOW 配网 initiator 任务（responder 反过来向 initiator 请求 Wi-Fi 配置）在循环里**显式等待密钥就绪**：

```c
// components/espnow_device/responder.c — app_espnow_prov_initiator_init()
for (;;) {
#ifdef CONFIG_APP_ESPNOW_SECURITY
    uint8_t key_info[APP_KEY_LEN];
    if (espnow_get_key(key_info) != ESP_OK) {   // 密钥未就绪
        vTaskDelay(pdMS_TO_TICKS(1000));
        continue;                                // 等 initiator 握手把 key 派发下来
    }
#endif
    ret = espnow_prov_initiator_scan(responder_addr, &responder_info, &rx_ctrl,
                                     pdMS_TO_TICKS(3 * 1000));
    /* ... espnow_prov_initiator_send ... */
}
```

即：**安全未握手前 `espnow_get_key` 返回失败，配网/控制数据不会在加密管道发出**。initiator 端 `app_espnow_initiator_sec_task` 走标准 scan → `espnow_sec_initiator_start(key, pop, addrs, num, &result)` → `result_free`，握手成功后 responder 通过 `ESP_EVENT_ESPNOW_SEC_OK` 再次 `espnow_set_key`/`set_dec_key`。

### 4. 单按键复用：按键事件 → 功能映射

`iot_button` 在两个 GPIO 上分别注册单击/双击/长按，映射到不同功能：

**配网按键 `WIFI_PROV_KEY_GPIO`（GPIO2，两端共用）：**

| 事件 | initiator | responder |
|---|---|---|
| `BUTTON_SINGLE_CLICK` | `app_wifi_prov_over_espnow_start_press_cb` → `app_espnow_prov_beacon_start(30)` 启动 30s ESP-NOW 配网 beacon | （无，仅 initiator） |
| `BUTTON_DOUBLE_CLICK` | `wifi_prov()` 启动 BLE/SoftAP 上层配网 | `app_espnow_prov_responder_start()` 反向请求 Wi-Fi 配置 |
| `BUTTON_LONG_PRESS_START` | `network_prov_mgr_reset_wifi_provisioning()` + `esp_wifi_disconnect()` + `esp_restart()` | 同 |

**控制按键 `CONTROL_KEY_GPIO`（仅 initiator + control）：**

| 事件 | 动作 |
|---|---|
| `BUTTON_SINGLE_CLICK` | `espnow_ctrl_initiator_send(KEY_1, POWER, status)` 翻转状态 |
| `BUTTON_DOUBLE_CLICK` | `espnow_ctrl_initiator_bind(KEY_1, true)` |
| `BUTTON_LONG_PRESS_START` | `espnow_ctrl_initiator_bind(KEY_1, false)` 解绑 |

按键回调内用 `iot_button_get_event(arg)` 与预期事件比对（`ESP_ERROR_CHECK(!(EVENT == iot_button_get_event(arg)))`），确保同回调注册多次时分支正确。

### 5. 共享 LED 状态语义

单颗 RGB LED 在所有功能间复用，`app_led_set_color(r,g,b)` 集中表达状态：

| 颜色 | 含义 |
|---|---|
| 白 `(255,255,255)` | 配网进行中（`app_wifi_prov_start_press_cb` 进入 `APP_WIFI_PROV_START`） |
| 绿 `(0,255,0)` | Wi-Fi 已连/拿到 IP（`IP_EVENT_STA_GOT_IP`）/ responder 被绑定（`ESP_EVENT_ESPNOW_CTRL_BIND`） |
| 红 `(255,0,0)` | Wi-Fi 断开（`WIFI_EVENT_STA_DISCONNECTED`）/ 被解绑（`ESP_EVENT_ESPNOW_CTRL_UNBIND`） |
| 亮/灭 `(255,255,255)`/`(0,0,0)` | responder 控制值 on/off（`app_responder_ctrl_data_cb`） |

> C2 用三路 GPIO（`LED_RED/GREEN/BLUE_GPIO`），其余芯片用 `led_strip` RMT 驱动（`LED_STRIP_GPIO`）；IDF v4.x 与 v5.x 句柄类型不同，示例用 `ESP_IDF_VERSION` 宏分流。

### 6. responder 侧各特性启动（`app_espnow_responder()`）

```c
void app_espnow_responder()
{
#ifdef CONFIG_APP_ESPNOW_SECURITY
    const char *pop_data = CONFIG_APP_ESPNOW_SESSION_POP;
    uint8_t key_info[APP_KEY_LEN];
    if (espnow_get_key(key_info) == ESP_OK) {     // 历史密钥先恢复
        espnow_set_key(key_info);
        espnow_set_dec_key(key_info);
    }
    espnow_sec_responder_start(pop_data);          // 等待 initiator 握手
#endif
#ifdef CONFIG_APP_ESPNOW_DEBUG
    espnow_timesync_start();                        // 日志时间戳基准
    espnow_console_config_t cc = { .monitor_command.uart = true,
                                   .monitor_command.espnow = true };
    espnow_log_config_t lc = { .log_level_uart   = ESP_LOG_INFO,
                               .log_level_espnow = ESP_LOG_INFO,
                               .log_level_flash  = ESP_LOG_INFO };
    espnow_console_init(&cc);
    espnow_console_commands_register();
    espnow_log_init(&lc);
#endif
#ifdef CONFIG_APP_ESPNOW_OTA
    espnow_ota_config_t oc = { .skip_version_check       = true,
                               .progress_report_interval = 10 };
    espnow_ota_responder_start(&oc);
#endif
}
```

initiator 端的 debug 启动类似：`espnow_console_init` + `espnow_console_commands_register` + `espnow_set_config_for_data_type(ESPNOW_DATA_TYPE_DEBUG_LOG, true, app_espnow_debug_recv_process)`；OTA 在 initiator 侧通过 console 命令触发（见 `wireless_debug` recipe）。

### 7. 构建两套固件并烧录

首次必须 `erase_flash` 清掉残留 NVS/密钥，否则握手 PoP/key 不一致会报 `mbedtls_ccm_auth_decrypt error -15`：

```shell
# responder 固件
cd examples/solution/
rm -rf build/
export PROJECT_NAME=Resp
idf.py set-target esp32c3
idf.py erase_flash flash build monitor     # 产物 build/Resp.bin

# initiator 固件（默认 PROJECT_NAME=Init）
unset PROJECT_NAME
idf.py build erase_flash flash monitor
```

两块板子上电后 initiator 自动扫描 responder 握手（日志 `Devices security completed, successed_num: 1`），之后即可按按键复用流程操作。

## 常见错误

| 错误 | 原因 | 解决方法 |
|---|---|---|
| `mbedtls_ccm_auth_decrypt error code -15` | initiator 擦过 flash 重生成新 key，与 responder 旧 key 不一致 | 擦 responder flash 并复位，重新握手 |
| responder 收不到加密配网/控制包 | 安全握手未完成就发数据 | 确认日志出现 `SEC_OK` / `successed_num` 后再触发；`app_espnow_prov_initiator_init` 已在 `espnow_get_key` 上轮询等 |
| 单击触发错功能 | 同一按键注册多个回调，事件判定缺失 | 回调内 `ESP_ERROR_CHECK(!(EXPECTED == iot_button_get_event(arg)))` 守卫 |
| initiator 双击进了配网而非 BLE 配网 | 状态机 `s_wifi_prov_status` 未在 `APP_WIFI_PROV_INIT` | 先长按复位清状态再双击；或检查 `network_prov_mgr_reset_wifi_provisioning` 是否执行 |
| LED 不亮 | 芯片 `LED_STRIP_GPIO` 不对 | 按 `CONFIG_IDF_TARGET_*` 分支改 `app_main.c` 顶部宏（C3=8、S3=38、ESP32=18） |
| 节点多时发送队列满 | 默认 `qsize=32` | `CONFIG_APP_ESPNOW_QUEUE_SIZE` 调到 100 |
| BLE 配网与 ESP-NOW 抢 Wi-Fi | BLE 期间 Wi-Fi ps type 切换 | 示例已处理：`wifi_prov` 内 `set ps type:1` 配网完恢复 `ps type:0` |
| C3 构建报 `ADC_BUTTON_WIDTH` 未声明 | `button_adc.c` 旧定义 | 按示例 README 把 `#if` 加上 `CONFIG_IDF_TARGET_ESP32C3` 分支 |

## 参考

- `examples/solution/README.md` — 流程说明、initiator/responder 串口日志、troubleshooting
- `examples/solution/main/app_main.c` — app_main、LED、按键回调、Wi-Fi 事件处理
- `examples/solution/main/Kconfig.projbuild` — 全部 `CONFIG_APP_ESPNOW_*` / `CONFIG_APP_WIFI_PROVISION_*` 选项
- `examples/solution/components/espnow_device/initiator.c` — sec 任务、debug/prov responder 封装、`app_espnow_initiator_register`
- `examples/solution/components/espnow_device/responder.c` — sec responder、ESP-NOW 配网 initiator 轮询、debug/OTA 启动
- `examples/solution/components/wifi_prov/{wifi_prov.c,include/wifi_prov.h}` — BLE/SoftAP 上层配网封装（`wifi_prov_init`/`wifi_prov`）
- `User_Guide.md` — 功能矩阵与 initiator/responder 角色定义
- 关联 recipes：`security_handshake.md`、`provisioning_wifi.md`、`control_initiator.md`、`wireless_debug.md`、`ota_batch_upgrade.md`、`time_sync.md`
