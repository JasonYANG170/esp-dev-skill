# ICD 间歇连接设备（Short / Long Idle Time）

> **适用摘要**: 用 `CONFIG_ENABLE_ICD_SERVER=y` 把电池供电 / sleepy 的 ESP32-H2 / ESP32-C6 做成 Matter ICD（Intermittently Connected Device），按 SIT（Short Idle Time）或 LIT（Long Idle Time）配置 polling / idle / active 参数，并配合 power management、IEEE 802.15.4 sleep、tickless idle 真正进入低功耗。

## 触发意图

- "做电池供电的 Matter 设备 / 低功耗"
- "ICD / sleepy device / 间歇连接"
- "SIT vs LIT"
- "ICD Slow Polling / Fast Polling / Idle Mode Duration"
- "esp32h2 / esp32c6 省电"
- "LIT 需要 client 注册"

## 前置条件

| 条件 | 要求 |
|---|---|
| 参考工程 | `examples/icd_app/`（`main/app_main.cpp`、`main/app_driver.cpp`、`main/Kconfig.projbuild`、`sdkconfig.defaults*`） |
| 目标芯片 | esp32h2（仅 Thread）或 esp32c6（Thread）。仅这两款有 measured 功耗曲线 |
| IDF 版本 | v5.2.2+（README 要求） |
| 关键 Kconfig | `CONFIG_ENABLE_ICD_SERVER=y`、（LIT）`CONFIG_ENABLE_ICD_LIT=y`、`CONFIG_PM_ENABLE=y`、`CONFIG_FREERTOS_USE_TICKLESS_IDLE=y`、`CONFIG_IEEE802154_SLEEP_ENABLE=y` |

## 分步说明

### 1. SIT vs LIT 默认参数对比

来自 `examples/icd_app/README.md`。SIT 是默认；LIT 用 `sdkconfig.defaults.<target>.lit` 覆盖。

| 参数 | SIT | LIT | Kconfig |
|---|---|---|---|
| ICD Fast Polling Interval | 500ms | 500ms | `CONFIG_ICD_FAST_POLL_INTERVAL_MS` |
| ICD Slow Polling Interval | 5000ms | 20000ms | `CONFIG_ICD_SLOW_POLL_INTERVAL_MS` |
| ICD Active Mode Duration | 1000ms | 1000ms | `CONFIG_ICD_ACTIVE_MODE_INTERVAL_MS` |
| ICD Idle Mode Duration | 60s | 600s | `CONFIG_ICD_IDLE_MODE_INTERVAL_SEC` |
| ICD Active Mode Threshold | 1000ms | 5000ms | `CONFIG_ICD_ACTIVE_MODE_THRESHOLD_MS` |

### 2. SIT sdkconfig（默认，`sdkconfig.defaults` + `sdkconfig.defaults.esp32h2`）

```text
CONFIG_ENABLE_ICD_SERVER=y
CONFIG_ICD_FAST_POLL_INTERVAL_MS=500
CONFIG_ICD_IDLE_MODE_INTERVAL_SEC=60
CONFIG_ICD_ACTIVE_MODE_INTERVAL_MS=1000
CONFIG_ICD_ACTIVE_MODE_THRESHOLD_MS=1000
# Slow Polling 用 SIT 上限（spec 默认 5000ms）
```

### 3. LIT sdkconfig（`sdkconfig.defaults.esp32h2.lit`）

```text
# ICD configuration
CONFIG_ENABLE_ICD_SERVER=y
CONFIG_ICD_SLOW_POLL_INTERVAL_MS=20000
CONFIG_ICD_FAST_POLL_INTERVAL_MS=500
CONFIG_ICD_IDLE_MODE_INTERVAL_SEC=600
CONFIG_ICD_ACTIVE_MODE_INTERVAL_MS=1000
CONFIG_ICD_ACTIVE_MODE_THRESHOLD_MS=5000
CONFIG_ENABLE_ICD_LIT=y
CONFIG_ICD_CLIENTS_SUPPORTED_PER_FABRIC=2
CONFIG_ICD_MAX_NOTIFICATION_SUBSCRIBERS=2
```

```bash
# ESP32-H2 LIT
idf.py -D SDKCONFIG_DEFAULTS="sdkconfig.defaults;sdkconfig.defaults.esp32h2;sdkconfig.defaults.esp32h2.lit" set-target esp32h2 build
# ESP32-C6 LIT
idf.py -D SDKCONFIG_DEFAULTS="sdkconfig.defaults;sdkconfig.defaults.esp32c6;sdkconfig.defaults.esp32c6.lit" set-target esp32c6 build
```

### 4. 必须配套的省电 Kconfig（否则 ICD 形同虚设）

ICD server 只是声明参数，真正进低功耗还要打开电源管理 + tickless + 15.4 sleep + BLE sleep（来自 `examples/icd_app/sdkconfig.defaults`）：

```text
# Power Management
CONFIG_PM_ENABLE=y
CONFIG_PM_DFS_INIT_AUTO=y
CONFIG_PM_POWER_DOWN_PERIPHERAL_IN_LIGHT_SLEEP=y
CONFIG_ESP_SLEEP_POWER_DOWN_FLASH=y

# FreeRTOS
CONFIG_FREERTOS_HZ=1000
CONFIG_FREERTOS_USE_TICKLESS_IDLE=y

# IEEE802.15.4 sleep（Thread 收发器睡眠）
CONFIG_IEEE802154_SLEEP_ENABLE=y

# BLE Sleep（仅 commissioning 用 BLE）
CONFIG_BT_LE_SLEEP_ENABLE=y
CONFIG_BT_LE_LP_CLK_SRC_MAIN_XTAL=y
CONFIG_RTC_CLK_SRC_EXT_CRYS=n       # 不用外部 32K 晶振

# OpenThread MTD（最小化 Thread 协议栈，省内存）
CONFIG_OPENTHREAD_MTD=y
CONFIG_OPENTHREAD_CLI=n
```

### 5. `app_main` 结构（ICD 设备就是普通 server + ICD server）

来自 `examples/icd_app/main/app_main.cpp`：先配 `esp_pm_configure()`，再正常建 node + endpoint + start：

```cpp
#if CONFIG_PM_ENABLE
    esp_pm_config_t pm_config = {
        .max_freq_mhz = CONFIG_ESP_DEFAULT_CPU_FREQ_MHZ,
        .min_freq_mhz = CONFIG_ESP_DEFAULT_CPU_FREQ_MHZ,
#if CONFIG_FREERTOS_USE_TICKLESS_IDLE
        .light_sleep_enable = true
#endif
    };
    esp_pm_configure(&pm_config);
#endif

#ifdef CONFIG_ENABLE_USER_ACTIVE_MODE_TRIGGER_BUTTON
    app_driver_button_init();
#endif

    node::config_t node_config;
    node_t *node = node::create(&node_config, app_attribute_update_cb, app_identification_cb);

    endpoint::on_off_light::config_t endpoint_config;
    endpoint_t *app_endpoint = endpoint::on_off_light::create(node, &endpoint_config, ENDPOINT_FLAG_NONE, NULL);

    // Thread 平台配置
#if CHIP_DEVICE_CONFIG_ENABLE_THREAD
    esp_openthread_platform_config_t config = {
        .radio_config = ESP_OPENTHREAD_DEFAULT_RADIO_CONFIG(),
        .host_config  = ESP_OPENTHREAD_DEFAULT_HOST_CONFIG(),
        .port_config  = ESP_OPENTHREAD_DEFAULT_PORT_CONFIG(),
    };
    set_openthread_platform_config(&config);
#endif

    esp_matter::start(app_event_cb);
```

ICD server 由 Kconfig `CONFIG_ENABLE_ICD_SERVER=y` 自动启用，无需额外 API 调用。

### 6. 用户主动触发 Active Mode（按键唤醒）

`examples/icd_app/main/Kconfig.projbuild` 暴露了一个按键，按下后设备保持 Active 持续 `ActiveModeDuration`：

```text
menu "Example Configuration"
    config ENABLE_USER_ACTIVE_MODE_TRIGGER_BUTTON
        bool "Enable User Active Mode Trigger Button"
        default y
    config USER_ACTIVE_MODE_TRIGGER_BUTTON_PIN
        int "User Active Mode Trigger Button Pin"
        range 9 14 if IDF_TARGET_ESP32H2
        range 0 7  if IDF_TARGET_ESP32C6
        default 9 if IDF_TARGET_ESP32H2
        default 7 if IDF_TARGET_ESP32C6
        default 28 if IDF_TARGET_ESP32C5
endmenu
```

> ESP32-C6 DevKit 的 boot 按钮 GPIO9 不能用于唤醒，故示例默认 GPIO7。

`CONFIG_GPIO_BUTTON_SUPPORT_POWER_SAVE=y` 让按键驱动也支持 deep-sleep 唤醒。

### 7. LIT 必须有 client 注册（Matter 1.4 规则）

来自 `examples/icd_app/README.md` 引用 Matter 1.4 spec：

> "A LIT ICD SHALL operate as a SIT ICD if it doesn't have at least one registration with any client on any fabric in the ICD Management cluster."

即：LIT 设备在没有任何 client 通过 ICD Management cluster 注册前，必须按 SIT 行为运行（Slow Polling 不能超过 SIT 上限）。controller 侧通过 ICD Management cluster 的 `RegisterClient` 命令完成注册（`CONFIG_ICD_CLIENTS_SUPPORTED_PER_FABRIC` / `CONFIG_ICD_MAX_NOTIFICATION_SUBSCRIBERS` 控制容量）。

### 8. 实测功耗（参考量）

`examples/icd_app/image/` 给出 20dBm 发射功率下的电流波形图（README 列举）：

- `H2-sit-icd.png` — ESP32-H2 SIT
- `C6-sit-icd.png` — ESP32-C6 SIT
- `H2-lit-icd.png` — ESP32-H2 LIT
- `C6-lit-icd.png` — ESP32-C6 LIT

具体数值随配置变化，以实测为准。

## 常见错误

| 错误 | 原因 | 解决方法 |
|---|---|---|
| LIT 设备仍按 SIT 轮询 | 没有 client 在 ICD Management cluster 注册 | 用 controller 发 `RegisterClient`；或先按 SIT 上限设 Slow Polling |
| 功耗没降 | 只开了 `ENABLE_ICD_SERVER`，没开 PM/tickless | 必须配套 `CONFIG_PM_ENABLE=y` + `FREERTOS_USE_TICKLESS_IDLE=y` + `IEEE802154_SLEEP_ENABLE=y` |
| ESP32-C6 按键唤不醒 | 用了 boot 键 GPIO9（不可唤醒） | 换 GPIO7（见 `USER_ACTIVE_MODE_TRIGGER_BUTTON_PIN`） |
| Thread 收发异常 | 用了 FTD | ICD 设备用 MTD：`CONFIG_OPENTHREAD_MTD=y` |
| LIT 注册失败 | 容量不足 | 调大 `CONFIG_ICD_CLIENTS_SUPPORTED_PER_FABRIC` / `CONFIG_ICD_MAX_NOTIFICATION_SUBSCRIBERS` |
| 外部 32K 晶振导致高功耗 | 用了 ext crystal 源 | `CONFIG_RTC_CLK_SRC_EXT_CRYS=n`，用主 XTAL LP 时钟 |
| Slow/Fast Polling 写错单位 | ms vs s | `IDLE_MODE_INTERVAL_SEC` 是秒，其它是毫秒 |

## 参考

- `D:/esp-skill/espressif-repos/esp-matter/examples/icd_app/README.md` — SIT/LIT 参数表、LIT 注册规则、电流波形图
- `D:/esp-skill/espressif-repos/esp-matter/examples/icd_app/main/app_main.cpp` — `esp_pm_configure()` + 普通 node/endpoint + Thread 平台配置
- `D:/esp-skill/espressif-repos/esp-matter/examples/icd_app/main/app_driver.cpp` — 按键 / 用户 active mode 触发
- `D:/esp-skill/espressif-repos/esp-matter/examples/icd_app/main/Kconfig.projbuild` — `ENABLE_USER_ACTIVE_MODE_TRIGGER_BUTTON` / 引脚范围
- `D:/esp-skill/espressif-repos/esp-matter/examples/icd_app/sdkconfig.defaults`（SIT）
- `D:/esp-skill/espressif-repos/esp-matter/examples/icd_app/sdkconfig.defaults.esp32h2.lit` / `sdkconfig.defaults.esp32c6.lit`（LIT）
- `D:/esp-skill/espressif-repos/esp-matter/examples/icd_app/image/` — `H2-sit-icd.png` / `C6-sit-icd.png` / `H2-lit-icd.png` / `C6-lit-icd.png`
- Matter 1.4 specification — LIT ICD 注册要求
