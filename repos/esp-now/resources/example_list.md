# ESP-NOW 官方示例索引

> 路径取自 `espressif/esp-now` 仓库 `examples/` 真实目录。描述基于各示例 `main/app_main.c` 与 README/User Guide。

## 示例总表

| 示例路径 | 角色/说明 |
|---|---|
| `examples/get-started` | 入门：storage + Wi-Fi + espnow_init，UART 与 ESP-NOW 广播透传收发用户数据（`ESPNOW_DATA_TYPE_DATA`） |
| `examples/control` | 设备控制一体示例：按键（initiator 绑定/解绑/发送控制）+ LED（responder 接收控制并动作），含 control 事件处理与 LED 驱动 |
| `examples/coin_cell_demo/bulb` | 硬币电池灯（responder）：接收控制、状态持久化到 NVS（`bulb_key`）、LED 驱动 |
| `examples/coin_cell_demo/switch` | 硬币电池开关（initiator）：light sleep 充电、power lock、状态机（send/bind/unbind），含 coin-cell V1 与普通按键两套流程 |
| `examples/ota` | 批量 OTA：initiator 从 HTTP 下载固件写本地分区并分发（断点续传），responder `espnow_ota_responder_start` 接收 |
| `examples/security` | 安全握手：initiator 分发 app key（ECDH+PoP）、responder 握手，加解密 ESP-NOW 用户数据 |
| `examples/provisioning` | Wi-Fi 配网：responder 广播 beacon 并校验后下发 SSID/密码，initiator 扫描并应用配置 |
| `examples/wireless_debug` | 无线调试：`monitor`（调试主机抓日志/下发命令）与 `monitored`（被调试设备）双角色，封装在 `components/espnow_device` |
| `examples/solution` | 综合方案：控制 + 配网 + 时间同步（`CONFIG_APP_ESPNOW_TIMESYNC`/`_CONTROL`/`_INITIATOR`/`_RESPONDER`）+ Wi-Fi 配网（`components/wifi_prov`） |

## 详细说明

### examples/get-started
- 入口：`main/app_main.c`
- 关键流程：`espnow_storage_init` → `app_wifi_init`(STA) → `espnow_init` → `espnow_set_config_for_data_type(DATA, true, app_uart_write_handle)` → UART 读任务循环 `espnow_send(..., ESPNOW_ADDR_BROADCAST, ...)`。
- 适合作为所有 ESP-NOW 工程的起点模板。

### examples/control
- 入口：`main/app_main.c`（initiator + responder 同文件，按按键行为区分）
- initiator：双击 `espnow_ctrl_initiator_bind(KEY_1,true)`、单击 `espnow_ctrl_initiator_send(KEY_1,POWER,status)`、长按解绑。
- responder：`espnow_ctrl_responder_bind(30s,-55,NULL)` + `espnow_ctrl_responder_data(cb)`，cb 内根据 value 控 LED。
- 各芯片默认 GPIO 见 `CONFIG_IDF_TARGET_*` 分支（C3/C2/C6 KEY=9；ESP32/S2/S3 KEY=0）。

### examples/coin_cell_demo/{bulb,switch}
- switch：低功耗核心。`app_main` 设 `cfg.send_max_timeout = portMAX_DELAY`、`esp_now_set_wake_window(0)`；`control_task` 状态机用 `set_light_sleep` 充电；状态用 `espnow_storage_get/set("bulb_key")` 持久化。
- bulb：responder 侧，状态持久化、LED 驱动。

### examples/ota
- initiator：`app_firmware_download`（HTTP + `esp_ota_begin/write/end`）→ `esp_partition_get_sha256` → `app_firmware_send`（scan → build dest list → `espnow_ota_initiator_send` → `result_free`）。
- responder：`espnow_ota_config_t{skip_version_check=true, progress_report_interval=10}` → `espnow_ota_responder_start`。
- 需联网（`example_connect`）。

### examples/security
- initiator：随机/读取 `key_info[APP_KEY_LEN]` → `espnow_set_key`+`espnow_set_dec_key` → scan → `espnow_sec_initiator_start(key,pop,addrs,num,&result)` → `result_free`。
- responder：`esp_event_handler_register(ESP_EVENT_ESPNOW, ANY_ID, cb)` 监听 `SEC_OK/_FAIL` → `espnow_sec_responder_start(pop)`。
- 发送时 `frame_head.security = s_sec_flag`。

### examples/provisioning
- responder：`espnow_prov_responder_start(&info, 30s, &wifi_config, app_recv_cb)`；回调校验 initiator 返回 ESP_OK 才下发。
- initiator：`espnow_prov_initiator_scan` → `espnow_prov_initiator_send`；接收回调内 memcpy + `esp_wifi_set_config` + `esp_wifi_connect`。

### examples/wireless_debug
- 入口极简：`app_espnow_monitor_device_start()` 或 `app_espnow_monitored_device_start()`（来自 `components/espnow_device/monitor.h`，示例内部封装）。
- 底层公开 API 在 `espnow_log.h`/`espnow_console.h`/`espnow_cmd.h`。

### examples/solution
- 综合：条件编译 `CONFIG_APP_ESPNOW_TIMESYNC`/`_CONTROL`/`_INITIATOR`/`_RESPONDER` + `CONFIG_APP_WIFI_PROVISION` + `CONFIG_APP_ESPNOW_SECURITY`/`_DEBUG`/`_OTA`/`_PROVISION`。
- 入口：`main/app_main.c`（角色 choice、LED、按键复用、Wi-Fi 事件）+ `main/Kconfig.projbuild`（全部 `CONFIG_APP_ESPNOW_*` 选项）。
- 自定义组件：`components/espnow_device/{initiator.c,responder.c}`（封装 sec/console/prov 启动）、`components/wifi_prov/`（BLE/SoftAP 上层配网）。
- 关键约束：responder 的 ESP-NOW 配网任务在 `espnow_get_key()` 上轮询，安全握手完成前不发加密业务数据。
- 单按键复用（`WIFI_PROV_KEY_GPIO=2`、`CONTROL_KEY_GPIO=9/0`）：单击/双击/长按映射到配网 beacon / BLE 配网 / 绑定 / 控制 / 复位；共享 RGB LED 表达白/绿/红状态。
- 适合作为产品级参考，配套 recipe：`recipes/solution_integrated_firmware.md`。

## 通过组件管理器下载示例

```shell
idf.py create-project-from-example "espressif/esp-now=*:get-started"
idf.py create-project-from-example "espressif/esp-now=*:control"
idf.py create-project-from-example "espressif/esp-now=*:coin_cell_demo/bulb"
idf.py create-project-from-example "espressif/esp-now=*:coin_cell_demo/switch"
idf.py create-project-from-example "espressif/esp-now=*:ota"
idf.py create-project-from-example "espressif/esp-now=*:security"
idf.py create-project-from-example "espressif/esp-now=*:provisioning"
idf.py create-project-from-example "espressif/esp-now=*:wireless_debug"
idf.py create-project-from-example "espressif/esp-now=*:solution"
```

> 旧版包管理器若报 `CMakeLists.txt not found`，运行 `pip install -U idf-component-manager` 升级（见仓库 README Q&A）。
