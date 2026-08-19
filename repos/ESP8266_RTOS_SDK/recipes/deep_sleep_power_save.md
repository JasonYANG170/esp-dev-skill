# 深度睡眠与 WiFi 省电（Deep sleep & power save）

> **适用摘要**: ESP8266 的两类省电：①WiFi modem sleep（`esp_wifi_set_ps`，STA 连上 AP 后周期性关 RF，保持连接，三种模式 NONE/MIN/MAX）；②deep sleep（`esp_deep_sleep`，关 CPU+RF，定时器或 RST 唤醒，唤醒等同重启）。前者在连接态省电，后者用于周期性采集场景。

> Version: ESP8266 RTOS SDK version used by the project.
> Evidence: `repos/ESP8266_RTOS_SDK/resources/`, source/examples in `repos/ESP8266_RTOS_SDK/`, and this recipe path `repos/ESP8266_RTOS_SDK/recipes/deep_sleep_power_save.md`.
> Validation: draft metadata added from repository routing; verify APIs, Kconfig symbols, and component dependencies against the selected version.

## 触发意图

- "ESP8266 省电 / 低功耗"
- "modem sleep / WiFi sleep"
- "deep sleep 定时唤醒"
- "GPIO 唤醒 / light sleep"

## 前置条件

| 条件 | 要求 |
|---|---|
| 参考示例 | `examples/wifi/power_save`（modem sleep）、`examples/protocols/sntp`（周期性场景常见组合） |
| 头文件 | `components/esp8266/include/esp_sleep.h`、`esp_wifi.h` |
| 硬件 | deep sleep 定时唤醒需要 **XPD_DCDC 经 0Ω 接到 EXT_RSTB**（见 esp_sleep.h attention 1）；否则定时器无法复位唤醒 |
| WiFi 模式 | modem sleep 只在 STA 模式连上 AP 后生效 |

## 分步说明

### 1. WiFi modem sleep（最常用，连上 AP 自动省电）

power_save 示例在 `example_connect()` 之后调 `esp_wifi_set_ps`：

```c
#include "esp_wifi.h"
#include "esp_sleep.h"
#include "esp_netif.h"
#include "esp_event.h"
#include "nvs_flash.h"
#include "protocol_examples_common.h"

#if CONFIG_EXAMPLE_POWER_SAVE_MIN_MODEM
#define DEFAULT_PS_MODE WIFI_PS_MIN_MODEM
#elif CONFIG_EXAMPLE_POWER_SAVE_MAX_MODEM
#define DEFAULT_PS_MODE WIFI_PS_MAX_MODEM
#elif CONFIG_EXAMPLE_POWER_SAVE_NONE
#define DEFAULT_PS_MODE WIFI_PS_NONE
#endif

void app_main(void)
{
    ESP_ERROR_CHECK(nvs_flash_init());
    ESP_ERROR_CHECK(esp_netif_init());
    ESP_ERROR_CHECK(esp_event_loop_create_default());
    ESP_ERROR_CHECK(example_connect());

    esp_wifi_set_ps(DEFAULT_PS_MODE);          // 必须在连接后设
}
```

三种模式（来自 README，由 `Kconfig.projbuild` 的 choice 选择）：

| 模式 | 行为 | 广播数据 | 省电 |
|---|---|---|---|
| `WIFI_PS_NONE` | 默认，全功率 | 不丢 | 无 |
| `WIFI_PS_MIN_MODEM` | 每个 DTIM 醒来收 beacon | 不丢（DTIM 后发广播） | 受 AP DTIM 影响 |
| `WIFI_PS_MAX_MODEM` | 每个 listen_interval 醒来 | **可能丢** | 更多（interval 越长越省） |

> `WIFI_PS_MAX_MODEM` 时若想更省电，可在 STA 配置里设 `wifi_sta_config_t.listen_interval`（多个 beacon 周期）。

### 2. 动态调频 + 自动 light sleep（可选）

sdkconfig 开 `CONFIG_PM_ENABLE` 后，配 `esp_pm_config_esp8266_t`：

```c
#if CONFIG_PM_ENABLE
    esp_pm_config_esp8266_t pm_config = {
        .light_sleep_enable = true,
    };
    ESP_ERROR_CHECK(esp_pm_configure(&pm_config));
#endif
```

> 这会让系统在空闲时自动降频并进入 light sleep（需 tickless idle 支持）。`esp_pm_configure` 签名来自 esp_sleep.h：传 `esp_pm_config_esp8266_t*`。

### 3. Deep sleep（定时唤醒，唤醒后从 app_main 重跑）

```c
#include "esp_sleep.h"
#include "esp_log.h"

void app_main(void)
{
    // 采集/上报后准备睡眠
    ESP_LOGI("DS", "going deep sleep 30s");

    // esp_deep_sleep 不优雅关闭 WiFi/协议栈，先手动停
    esp_wifi_stop();                          // 见 esp_sleep.h attention 3

    esp_deep_sleep(30 * 1000 * 1000);         // 30 秒，单位 us
    // 此行之后不会执行 —— 唤醒等同重启，从 app_main 重新跑
}
```

`esp_deep_sleep(uint64_t time_in_us)`（esp_sleep.h）：
- 传 0 表示无定时唤醒，需在 RST 引脚接 GPIO，下降沿唤醒（attention 2）。
- 唤醒后从 `app_main` 重新开始（RAM 不保持）。

### 4. 射频校准策略（影响唤醒后电流/性能）

`esp_deep_sleep_set_rf_option(option)`（esp_sleep.h）—— 必须在 `esp_wifi_init` 之前调：

| option | 行为 | 唤醒后电流 |
|---|---|---|
| 0 | 由 `esp_init_data_default.bin` 的 byte 108 决定 | — |
| 1 | 唤醒后做射频校准（默认） | 较强 |
| 2 | 不做校准 | 较弱 |
| 4 | 禁用射频校准（等同 modem sleep） | 最弱，唤醒后**不能收发** |

### 5. Light sleep（GPIO 唤醒，保留 RAM）

```c
esp_wifi_stop();                              // 否则返回 ESP_ERR_INVALID_STATE
esp_sleep_enable_timer_wakeup(5 * 1000 * 1000);   // 5s 定时
esp_sleep_enable_gpio_wakeup();               // GPIO 唤醒（仅 light sleep）
esp_light_sleep_start();                      // 返回发生在唤醒后
```

> `esp_light_sleep_start` 若 WiFi 未 stop 会返回 `ESP_ERR_INVALID_STATE`（见 esp_sleep.h 注释）。

### 6. 关键 API

| API | 作用 |
|---|---|
| `esp_wifi_set_ps(wifi_ps_type_t)` | 设 WiFi modem sleep：`WIFI_PS_NONE/MIN_MODEM/MAX_MODEM` |
| `esp_pm_configure(const void *)` | 动态调频 / 自动 light sleep（传 `esp_pm_config_esp8266_t*`） |
| `esp_deep_sleep(uint64_t time_in_us)` | 进 deep sleep，唤醒等同重启 |
| `esp_deep_sleep_set_rf_option(uint8_t option)` | 唤醒后射频校准策略（0/1/2/4） |
| `esp_sleep_enable_timer_wakeup(uint32_t time_in_us)` | 使能定时唤醒源 |
| `esp_light_sleep_start(void)` | 进 light sleep（保留 RAM，GPIO/定时唤醒） |
| `esp_sleep_enable_gpio_wakeup(void)` | 使能 GPIO 唤醒（仅 light sleep） |
| `esp_sleep_disable_wakeup_source(esp_sleep_source_t)` | 禁用某唤醒源 |

## 常见错误

| 错误 | 原因 | 解决方法 |
|---|---|---|
| deep sleep 后不唤醒（彻底死掉） | XPD_DCDC 未接 EXT_RSTB | 硬件上 0Ω 连接（见 esp_sleep.h attention 1） |
| `esp_light_sleep_start` 返回 `ESP_ERR_INVALID_STATE` | WiFi 没 stop | 先 `esp_wifi_stop()` |
| deep sleep 后 WiFi 连不上 | 用了 `set_rf_option(4)` | 改用 1（唤醒后校准）；4 仅用于不收发的极低功耗 |
| modem sleep 不生效 | 不是 STA / 没 `set_ps` / 在连接前设 | STA 连上 AP 后再 `esp_wifi_set_ps` |
| MAX_MODEM 丢广播 | listen_interval 太长 | 减小 `listen_interval` 或改 MIN_MODEM |
| deep sleep 醒来状态全丢 | deep sleep 不保持 RAM | 关键数据写 NVS/flash 再睡 |

## 参考

- `examples/wifi/power_save/main/power_save.c` — modem sleep（`esp_wifi_set_ps` + `esp_pm_configure`）
- `examples/wifi/power_save/main/Kconfig.projbuild` — `CONFIG_EXAMPLE_POWER_SAVE_NONE/MIN_MODEM/MAX_MODEM`
- `examples/wifi/power_save/README.md` — 三种模式说明
- `components/esp8266/include/esp_sleep.h` — `esp_deep_sleep` / `esp_light_sleep_start` / `esp_deep_sleep_set_rf_option`
- `docs/en/api-reference/system/sleep_modes.rst` — API 参考（指向 esp_sleep.h）
