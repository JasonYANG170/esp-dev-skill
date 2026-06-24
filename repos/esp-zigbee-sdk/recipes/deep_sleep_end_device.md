# 深度休眠终端（Deep Sleep End Device）

> **适用摘要**: 实现 Zigbee End Device 的 deep sleep 最低功耗模式：每次唤醒都从 reset 重启、重新初始化协议栈并 rejoin 网络，用 RTC timer + EXT1（BOOT 按键）双唤醒源，`RTC_DATA_ATTR` 跨睡眠保存时间戳。与 light sleep（Zigbee 任务常驻、keep-alive 自动维持）是两套完全不同的流程。

## 触发意图

- "Zigbee 深度休眠 / 深睡"
- "deep sleep end device"
- "deep sleep Zigbee"
- "电池供电 Zigbee（长周期，>30 分钟）"
- "RTC timer 唤醒 Zigbee"
- "唤醒即重启 / wake from reset"

## 前置条件

| 条件 | 要求 |
|---|---|
| 硬件 | ESP32-H2 / ESP32-C6（native 802.15.4 SoC） |
| sdkconfig.defaults | `CONFIG_ZB_ENABLED=y`、`CONFIG_ZB_ZED=y`、`CONFIG_PARTITION_TABLE_CUSTOM=y`、`CONFIG_NEWLIB_TIME_SYSCALL_USE_RTC_HRT=y`、`CONFIG_RTC_CLK_SRC_INT_RC=y`、`CONFIG_BOOTLOADER_SKIP_VALIDATE_IN_DEEP_SLEEP=y` |
| partitions.csv | 必须含 `zb_storage`（data/nvs/16K）与 `zb_fct`（data/fat/1K） |
| idf_component.yml | 依赖 `switch_driver`、`alarm_timer`（`examples/utils/`） |
| 参考项目 | `examples/sleepy_devices/deep_sleep_end_device/` |

> 注意：deep sleep **不需要** `CONFIG_PM_ENABLE` / `CONFIG_FREERTOS_USE_TICKLESS_IDLE`（那是 light sleep 的配置）。deep sleep 由应用直接调 `esp_deep_sleep_start()`，Zigbee 协议栈本身不参与睡眠决策。

## 分步说明

### 1. 关键差异：deep sleep vs light sleep

| 维度 | light sleep | deep sleep |
|---|---|---|
| Zigbee 任务 | 常驻运行 | 睡眠时整个芯片断电，任务终止 |
| keep-alive | PM 自动维护 | 唤醒后需 rejoin 网络恢复 |
| 唤醒后 | 继续运行 | **从 reset 重启**（`rst:0x5 (SLEEP_WAKEUP)`） |
| 功耗 | 较低 | 最低（典型 ESP32-H2 ~数 µA） |
| 适用场景 | 周期 < 30 分钟 | 周期 > 30 分钟的长睡眠 |

### 2. 设备配置（与 light sleep 相同的 ZED 配置）

```c
#define ESP_ZIGBEE_ZED_CONFIG()                         \
    {                                                   \
        .device_type = EZB_NWK_DEVICE_TYPE_END_DEVICE,  \
        .install_code_policy = false,                   \
        .zed_config = {                                 \
            .ed_timeout = EZB_NWK_ED_TIMEOUT_64MIN,     \
            .keep_alive = 4000,   /* 毫秒 */             \
        },                                              \
    }
```

### 3. `app_main` 第一件事：配置唤醒源 + 识别唤醒原因

deep sleep 每次唤醒都是从 reset 重新执行 `app_main`，所以**唤醒源配置必须在协议栈启动之前**完成。用 `esp_sleep_get_wakeup_causes()` 判断本次是 timer、EXT1 还是首次上电。

```c
static RTC_DATA_ATTR struct timeval s_sleep_time;   // RTC 域，跨深睡保留

static esp_err_t esp_deep_sleep_weakup_config(void)
{
    /* oneshot 定时器：唤醒后工作 DEEP_SLEEP_TIME_WAKEUP_SEC 秒，再进入深睡 */
    const esp_timer_create_args_t oneshot_args = {
        .callback = &esp_deep_sleep_start_sleep,
        .name     = "deep_sleep_timer",
    };
    ESP_RETURN_ON_ERROR(esp_timer_create(&oneshot_args, &s_oneshot_timer), TAG, "Failed to create timer");

    /* 计算本次实际睡了多久（RTC_DATA_ATTR 的 s_sleep_time 在入睡前被记录） */
    struct timeval now;
    gettimeofday(&now, NULL);
    int sleep_time_ms = (int)((now.tv_sec - s_sleep_time.tv_sec) * 1000 +
                              (now.tv_usec - s_sleep_time.tv_usec) / 1000);

    uint32_t causes = esp_sleep_get_wakeup_causes();
    if (causes & BIT(ESP_SLEEP_WAKEUP_EXT1)) {
        ESP_LOGI(TAG, "Wake up from GPIO, deep sleep for %d milliseconds", sleep_time_ms);
    } else if (causes & BIT(ESP_SLEEP_WAKEUP_TIMER)) {
        ESP_LOGI(TAG, "Wake up from timer, deep sleep for %d milliseconds", sleep_time_ms);
    } else {
        ESP_LOGI(TAG, "Wake up from unknown cause, deep sleep for %d milliseconds", sleep_time_ms);
    }

    /* 配置下一次睡眠的唤醒源：RTC timer + EXT1（BOOT 按键） */
    ESP_RETURN_ON_ERROR(esp_sleep_enable_timer_wakeup((uint64_t)DEEP_SLEEP_TIME_SLEEP_SEC * 1000000),
                        TAG, "Failed to enable timer wakeup");
    ESP_RETURN_ON_ERROR(esp_sleep_enable_ext1_wakeup(BIT(CONFIG_GPIO_EXT1_WAKEUP_SOURCE),
                        ESP_EXT1_WAKEUP_ANY_LOW), TAG, "Failed to enable ext1 wakeup");
    ESP_ERROR_CHECK(gpio_wakeup_enable(CONFIG_GPIO_EXT1_WAKEUP_SOURCE, GPIO_INTR_LOW_LEVEL));

#if SOC_RTCIO_INPUT_OUTPUT_SUPPORTED
    rtc_gpio_pulldown_dis(CONFIG_GPIO_EXT1_WAKEUP_SOURCE);
    rtc_gpio_pullup_en(CONFIG_GPIO_EXT1_WAKEUP_SOURCE);
#else
    gpio_pulldown_dis(CONFIG_GPIO_EXT1_WAKEUP_SOURCE);
    gpio_pullup_en(CONFIG_GPIO_EXT1_WAKEUP_SOURCE);
#endif
    return ESP_OK;
}

void app_main(void)
{
    ESP_ERROR_CHECK(nvs_flash_init());
    ESP_ERROR_CHECK(nvs_flash_init_partition(ESP_ZIGBEE_STORAGE_PARTITION_NAME));
    ESP_ERROR_CHECK(esp_deep_sleep_weakup_config());   // 必须在任何 Zigbee 调用之前

    ESP_LOGI(TAG, "Start ESP Zigbee Stack");
    xTaskCreate(esp_zigbee_stack_main_task, "Zigbee_main", 4096, NULL, 5, NULL);
}
```

> `CONFIG_GPIO_EXT1_WAKEUP_SOURCE` 由 `examples/utils/switch_driver/Kconfig` 定义：H2 默认 9、C6 默认 7（BOOT 按键）。

### 4. 入睡入口：oneshot timer 到期调用 `esp_deep_sleep_start()`

不能用 `esp_deep_sleep_start()` 直接在信号处理器里调（会立即断电，ZCL 命令发不完）。示例用一个 `esp_timer` 在入网/绑定完成后延迟 `DEEP_SLEEP_TIME_WAKEUP_SEC` 秒再真正断电，确保所有报文发完。

```c
#define DEEP_SLEEP_TIME_SLEEP_SEC  20   // 睡眠时长（秒）
#define DEEP_SLEEP_TIME_WAKEUP_SEC 5    // 唤醒后工作时间（秒）

static void esp_deep_sleep_start_sleep(void *arg)
{
    ESP_LOGI(TAG, "Enter deep sleep for %d seconds", DEEP_SLEEP_TIME_SLEEP_SEC);
    gettimeofday(&s_sleep_time, NULL);   // 记录入睡时刻，供下次唤醒计算
    esp_deep_sleep_start();              // 整芯片断电，不返回
}

static esp_err_t esp_deep_sleep_enter_sleep(void)
{
    zdo_find_ha_light_device();          // 唤醒后业务：发现并绑定灯
    ESP_LOGI(TAG, "Enter deep sleep after %d seconds", DEEP_SLEEP_TIME_WAKEUP_SEC);
    ESP_RETURN_ON_ERROR(esp_timer_start_once(s_oneshot_timer,
                        DEEP_SLEEP_TIME_WAKEUP_SEC * 1000000), TAG, "Failed to start timer wakeup");
    return ESP_OK;
}
```

### 5. 信号处理器：区分首次入网 vs 唤醒后 rejoin

deep sleep 的核心是 `EZB_BDB_SIGNAL_DEVICE_FIRST_START` 与 `EZB_BDB_SIGNAL_DEVICE_REBOOT` **共用一个分支**：factory-new 时走 STEERING 入网；非 factory-new（已入过网，从深睡唤醒）直接调 `esp_deep_sleep_enter_sleep()` 做业务后再次入睡。协议栈会自动 rejoin 之前的网络。

```c
static bool esp_zigbee_app_signal_handler(const ezb_app_signal_t *app_signal)
{
    ezb_app_signal_type_t signal_type = ezb_app_signal_get_type(app_signal);

    switch (signal_type) {
    case EZB_ZDO_SIGNAL_SKIP_STARTUP:
        ESP_LOGI(TAG, "Initialize Zigbee stack");
        ezb_bdb_start_top_level_commissioning(EZB_BDB_MODE_INITIALIZATION);
        break;
    case EZB_BDB_SIGNAL_DEVICE_FIRST_START:
    case EZB_BDB_SIGNAL_DEVICE_REBOOT: {
        ezb_bdb_comm_status_t status = *((ezb_bdb_comm_status_t *)ezb_app_signal_get_params(app_signal));
        if (status == EZB_BDB_STATUS_SUCCESS) {
            ESP_LOGI(TAG, "Deferred driver initialization %s", deferred_driver_init() ? "failed" : "successful");
            ESP_LOGI(TAG, "Device started up in%s factory-reset mode", ezb_bdb_is_factory_new() ? "" : " non");
            if (ezb_bdb_is_factory_new()) {
                /* 首次上电：入网 */
                ezb_bdb_start_top_level_commissioning(EZB_BDB_MODE_NETWORK_STEERING);
            } else {
                /* 从深睡唤醒（已入过网）：协议栈已 rejoin，直接做业务后再次入睡 */
                ESP_LOGI(TAG, "Device rebooted");
                ESP_ERROR_CHECK(esp_deep_sleep_enter_sleep());
            }
        } else {
            ESP_LOGW(TAG, "%s failed with status(0x%02x), please retry",
                     ezb_app_signal_to_string(signal_type), status);
            alarm_timer_schedule(esp_zigbee_alarm_bdb_commissioning, EZB_BDB_MODE_INITIALIZATION, 1000);
        }
    } break;
    case EZB_BDB_SIGNAL_STEERING: {
        ezb_bdb_comm_status_t status = *((ezb_bdb_comm_status_t *)ezb_app_signal_get_params(app_signal));
        if (status == EZB_BDB_STATUS_SUCCESS) {
            ezb_extpanid_t extended_pan_id;
            ezb_nwk_get_extended_panid(&extended_pan_id);
            ESP_LOGI(TAG, "Joined network successfully: PAN ID(0x%04hx, EXT: 0x%llx), Channel(%d), Short Address(0x%04hx)",
                     ezb_nwk_get_panid(), extended_pan_id.u64,
                     ezb_nwk_get_current_channel(), ezb_nwk_get_short_address());
            ESP_ERROR_CHECK(esp_deep_sleep_enter_sleep());   /* 入网成功 → 业务 → 入睡 */
        } else {
            ESP_LOGW(TAG, "Failed to join network with status(0x%02x)", status);
            alarm_timer_schedule(esp_zigbee_alarm_bdb_commissioning, EZB_BDB_MODE_NETWORK_STEERING, 1000);
        }
    } break;
    default:
        ESP_LOGI(TAG, "Zigbee APP Signal: %s(type: 0x%02x)", ezb_app_signal_to_string(signal_type), signal_type);
        break;
    }
    return true;
}
```

### 6. 按键发命令（唤醒后业务示例）

EXT1 唤醒后做 `deferred_driver_init` 初始化 BOOT 按键，按键 toggle 灯。命令包裹在 `esp_zigbee_lock` 内。

```c
static void button_event_handler(switch_driver_handle_t handle)
{
    ESP_RETURN_ON_FALSE(handle != SWITCH_INV_HANDLE, , TAG, "Invalid switch handle");

    ezb_zcl_on_off_cmd_t cmd_req = {
        .cmd_ctrl = {
            .dst_addr.addr_mode = EZB_ADDR_MODE_NONE,   /* 走绑定表 */
            .src_ep             = ESP_ZIGBEE_HA_ON_OFF_SWITCH_EP_ID,
        },
    };
    esp_zigbee_lock_acquire(portMAX_DELAY);
    ezb_zcl_on_off_toggle_cmd_req(&cmd_req);
    esp_zigbee_lock_release();
    ESP_EARLY_LOGI(TAG, "Sent ZCL On/Off Toggle request");
}
```

### 7. partitions.csv 与 sdkconfig.defaults

必须与示例一致，否则深睡后 zb_storage 数据丢失导致无法 rejoin。

```text
# partitions.csv
# Name,   Type, SubType, Offset,  Size, Flags
nvs,        data, nvs,      ,        0x6000,
otadata,    data, ota,      ,        0x2000,
phy_init,   data, phy,      ,        0x1000,
factory,    app,  factory,  ,        940K,
zb_storage, data, nvs,      ,        16K,
zb_fct,     data, fat,      ,        1K,
```

```ini
# sdkconfig.defaults（深睡关键项）
CONFIG_PARTITION_TABLE_CUSTOM=y
CONFIG_PARTITION_TABLE_CUSTOM_FILENAME="partitions.csv"
CONFIG_ZB_ENABLED=y
CONFIG_ZB_ZED=y
CONFIG_NEWLIB_TIME_SYSCALL_USE_RTC_HRT=y     # gettimeofday 走 RTC，深睡后时间连续
CONFIG_RTC_CLK_SRC_INT_RC=y
CONFIG_BOOTLOADER_SKIP_VALIDATE_IN_DEEP_SLEEP=y   # 跳过 bootloader 校验，加速唤醒启动
```

## 常见错误

| 错误 | 原因 | 解决方法 |
|---|---|---|
| 唤醒后无法 rejoin 网络 | `zb_storage` 分区缺失或未初始化 | partitions.csv 含 `zb_storage`；`app_main` 调 `nvs_flash_init_partition(ESP_ZIGBEE_STORAGE_PARTITION_NAME)` |
| 唤醒后 `gettimeofday` 跳变 | 未用 RTC 时钟源 | sdkconfig 设 `CONFIG_NEWLIB_TIME_SYSCALL_USE_RTC_HRT=y` + `CONFIG_RTC_CLK_SRC_INT_RC=y` |
| 唤醒启动慢 | bootloader 每次校验 flash | 设 `CONFIG_BOOTLOADER_SKIP_VALIDATE_IN_DEEP_SLEEP=y` |
| EXT1 按键唤醒不灵 | rtc_gpio 上下拉未配 | `rtc_gpio_pullup_en` + `rtc_gpio_pulldown_dis`（C6 用 `gpio_*`） |
| 命令发完才断电需求未满足 | 直接在信号回调里 `esp_deep_sleep_start()` | 用 `esp_timer_start_once` 延迟几秒再断电 |
| 睡眠时长不准 | `esp_sleep_enable_timer_wakeup` 单位错 | 参数单位是 **微秒**（`sec * 1000000`） |
| 误用 light sleep 配置 | deep sleep 不需要 PM | deep sleep 不开 `CONFIG_PM_ENABLE`/`CONFIG_FREERTOS_USE_TICKLESS_IDLE`，直接调 `esp_deep_sleep_start()` |
| 周期短却用 deep sleep | 频繁 rejoin 报文开销大 | 周期 < 30 分钟用 light sleep（见 `sleepy_end_device.md`） |

## 参考项目

- `examples/sleepy_devices/deep_sleep_end_device/` — 官方 deep sleep 终端示例（本 recipe 全部代码来源）
- `examples/utils/switch_driver/` — BOOT 按键驱动，定义 `CONFIG_GPIO_EXT1_WAKEUP_SOURCE`（H2=9, C6=7）
- `examples/utils/alarm_timer/` — commissioning 失败延迟重试
- 仓库 `docs/en/faq.rst` "Zigbee Light Sleep Mode" 章节（对比 light sleep 配置）
- ESP-IDF `examples/system/deep_sleep/`、`examples/system/deep_sleep_wake_stub/`（更多唤醒方式与 wake stub）
