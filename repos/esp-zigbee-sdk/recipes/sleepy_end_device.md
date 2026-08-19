# 休眠终端（Sleepy End Device）：Light Sleep

> **适用摘要**: 实现 Zigbee End Device 的 light sleep 低功耗模式：关闭 RxOnWhenIdle、配置 `esp_pm`、用 EXT1 唤醒，按键触发 ZCL 命令并保持与父节点的 keep-alive。

> Version: selected repo/component version; align with the user project and dependency manifest.
> Evidence: `repos/esp-zigbee-sdk/resources/`, source/examples in `repos/esp-zigbee-sdk/`, and this recipe path `repos/esp-zigbee-sdk/recipes/sleepy_end_device.md`.
> Validation: draft metadata added from repository routing; verify APIs, Kconfig symbols, and component dependencies against the selected version.

## 触发意图

- "Zigbee 低功耗 / 省电"
- "sleepy end device"
- "light sleep Zigbee"
- "终端休眠唤醒"
- "电池供电 Zigbee"

## 前置条件

| 条件 | 要求 |
|---|---|
| 硬件 | ESP32-H2/C6（电池/低功耗场景） |
| sdkconfig | `CONFIG_PM_ENABLE=y`、`CONFIG_FREERTOS_USE_TICKLESS_IDLE=y`（可选 `CONFIG_PM_LIGHT_SLEEP_CALLBACKS=y`） |
| 参考项目 | `examples/sleepy_devices/light_sleep_end_device/`、`examples/sleepy_devices/deep_sleep_end_device/` |

## 分步说明

### 1. 设备配置（ZED + keep_alive）

```c
#define ESP_ZIGBEE_ZED_CONFIG()                         \
    {                                                   \
        .device_type = EZB_NWK_DEVICE_TYPE_END_DEVICE,  \
        .install_code_policy = false,                   \
        .zed_config = {                                 \
            .ed_timeout = EZB_NWK_ED_TIMEOUT_64MIN,     \
            .keep_alive = 4000,   /* 毫秒，与父节点保活 */ \
        },                                              \
    }
```

### 2. setup 中关闭 RxOnWhenIdle

休眠终端**必须**关闭 `RxOnWhenIdle`，否则协议栈不会让芯片进入轻睡。

```c
esp_err_t esp_zigbee_setup_commissioning(void)
{
    ezb_aps_secur_enable_distributed_security(false);
    ESP_ERROR_CHECK(ezb_bdb_set_primary_channel_set(ESP_ZIGBEE_PRIMARY_CHANNEL_MASK));
    ESP_ERROR_CHECK(ezb_bdb_set_secondary_channel_set(ESP_ZIGBEE_SECONDARY_CHANNEL_MASK));
    ESP_ERROR_CHECK(ezb_app_signal_add_handler(esp_zigbee_app_signal_handler));
    ezb_nwk_set_rx_on_when_idle(false);   // 关键
    return ESP_OK;
}
```

### 3. 延迟驱动初始化：EXT1 唤醒 + GPIO 唤醒

在 `EZB_BDB_SIGNAL_DEVICE_FIRST_START` 成功后做 `deferred_driver_init`，配置唤醒源（BOOT 按键）。

```c
static esp_err_t deferred_driver_init(void)
{
    switch_driver_config_t config = {
        .gpio_num = CONFIG_GPIO_EXT1_WAKEUP_SOURCE,
        .event_cb = button_event_handler,
    };
    switch_driver_init(&config);

    ESP_ERROR_CHECK(esp_sleep_enable_ext1_wakeup(BIT(CONFIG_GPIO_EXT1_WAKEUP_SOURCE), ESP_EXT1_WAKEUP_ANY_LOW));
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
```

### 4. `esp_pm` 轻睡配置（`CONFIG_PM_ENABLE` 时）

```c
#ifdef CONFIG_PM_ENABLE
static esp_err_t esp_pm_light_sleep_config(void)
{
#if CONFIG_FREERTOS_USE_TICKLESS_IDLE
    int cur_cpu = CONFIG_ESP_DEFAULT_CPU_FREQ_MHZ;
    esp_pm_config_t pm_config = {
        .max_freq_mhz       = cur_cpu,
        .min_freq_mhz       = cur_cpu,
        .light_sleep_enable = true,
    };
    return esp_pm_configure(&pm_config);
#endif
    return ESP_OK;
}
#endif
```

### 5. app_main 调用 PM 配置 + 标准 Zigbee 任务

```c
void app_main(void)
{
    ESP_ERROR_CHECK(nvs_flash_init());
    ESP_ERROR_CHECK(nvs_flash_init_partition(ESP_ZIGBEE_STORAGE_PARTITION_NAME));
#ifdef CONFIG_PM_ENABLE
    ESP_ERROR_CHECK(esp_pm_light_sleep_config());
#endif
    ESP_LOGI(TAG, "Start ESP Zigbee Stack");
    xTaskCreate(esp_zigbee_stack_main_task, "Zigbee_main", 4096, NULL, 5, NULL);
}
```

按键发命令的写法与普通 switch 完全一致（`ezb_zcl_on_off_toggle_cmd_req` + 锁包裹）。

## 常见错误

| 错误 | 原因 | 解决方法 |
|---|---|---|
| 功耗不下降 | 未关 RxOnWhenIdle | `ezb_nwk_set_rx_on_when_idle(false)` |
| 不进入 light sleep | 未开 PM | sdkconfig 设 `CONFIG_PM_ENABLE=y` 与 `CONFIG_FREERTOS_USE_TICKLESS_IDLE=y` |
| 唤醒后掉线 | keep_alive 间隔过长被父节点老化 | `keep_alive=4000`；`ed_timeout` 与父节点协商一致 |
| GPIO 唤醒不灵 | 未配 rtc_gpio 上下拉 | 按上面 `rtc_gpio_pullup_en` 配置 |
| 深睡场景需求 | light sleep 不够省 | 参考 `examples/sleepy_devices/deep_sleep_end_device/`（深睡 + 恢复） |

## 参考项目

- `examples/sleepy_devices/light_sleep_end_device/` — light sleep 终端 switch
- `examples/sleepy_devices/deep_sleep_end_device/` — deep sleep 终端
- 仓库 `docs/en/faq.rst` "Zigbee Light Sleep Mode" 章节
