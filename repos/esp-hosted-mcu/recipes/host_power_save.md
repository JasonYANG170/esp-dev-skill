# 主机省电（Host Power Save / Deep Sleep）

> **适用摘要**: 让 host MCU 进入低功耗状态（当前支持 deep sleep），由协处理器在需要时通过 GPIO 唤醒 host，同时保持网络在线。需要 host 与 slave 双方在 menuconfig 开启"Allow host to power save"。

## 触发意图

- "ESP-Hosted host 省电"
- "host deep sleep + slave 唤醒"
- "esp_hosted_power_save_start"
- "电池设备 ESP-Hosted"
- "host 待机"

## 前置条件

| 条件 | 要求 |
|---|---|
| 头文件 | `#include "esp_hosted.h"`（含 `esp_hosted_power_save.h`） |
| GPIO | host 唤醒引脚必须是 RTC-capable GPIO，且未作他用 |
| Kconfig | host 与 slave 都启用 "Allow host to power save" |
| 参考文档 | `docs/feature_host_power_save.md` |

## 分步说明

### 1. 双方 menuconfig 配置

**Host 侧**（`Component config -> ESP-Hosted config`）：
```text
[*] Allow host to power save
    [*] Allow host to enter deep sleep.
        (<gpio_num>) Host in: Host Wakeup GPIO     # RTC-capable GPIO
        Host Wakeup GPIO Level
            (X) High                                 # 默认
```

**Slave 侧**（`Example Configuration`）：
```text
[*] Allow host to power save
    [*] Allow host to enter deep sleep.
        (<gpio_num>) Slave out: Host wakeup GPIO    # slave 输出唤醒 host 的 GPIO
        Host Wakeup GPIO Level
            (X) High
```

> host 与 slave 的唤醒 GPIO 电平极性需匹配，且物理相连。

### 2. 枚举与 API（来自 `host/api/include/esp_hosted_power_save.h`）

```c
typedef enum {
    HOSTED_WAKEUP_UNDEFINED = 0,
    HOSTED_WAKEUP_NORMAL_REBOOT,
    HOSTED_WAKEUP_DEEP_SLEEP,
} esp_hosted_wakeup_reason_t;

typedef enum {
    HOSTED_POWER_SAVE_TYPE_NONE = 0,
    HOSTED_POWER_SAVE_TYPE_LIGHT_SLEEP,    // 尚未支持（Coming Soon）
    HOSTED_POWER_SAVE_TYPE_DEEP_SLEEP,
} esp_hosted_power_save_type_t;

int esp_hosted_power_save_init(void);     // 通常由 esp_hosted_init() 自动调用
int esp_hosted_power_save_deinit(void);   // 通常由 esp_hosted_deinit() 自动调用
int esp_hosted_power_save_enabled(void);  // Kconfig 是否启用
int esp_hosted_woke_from_power_save(void);// 本次启动是否源于 deep sleep 唤醒
int esp_hosted_power_saving(void);        // 当前是否处于省电模式
int esp_hosted_power_save_start(esp_hosted_power_save_type_t type);  // 不返回，进入 deep sleep 后重启
int esp_hosted_power_save_timer_start(uint32_t time_ms);            // 定时进入 deep sleep
int esp_hosted_power_save_timer_stop(void);
```

### 3. 典型用法

```c
#include "esp_hosted.h"
#include "esp_log.h"

void app_main(void)
{
    // ... 正常 esp_hosted_init/connect_to_slave + Wi-Fi ...

    if (esp_hosted_woke_from_power_save()) {
        ESP_LOGI(TAG, "woke from deep sleep (reason=%d)", HOSTED_WAKEUP_DEEP_SLEEP);
        // 可在此恢复网络/会话状态
    }

    // 方式 A：立即进入 deep sleep（不返回，唤醒即重启）
    if (esp_hosted_power_save_enabled()) {
        esp_hosted_power_save_start(HOSTED_POWER_SAVE_TYPE_DEEP_SLEEP);
        // 不会执行到这里
    }

    // 方式 B：定时进入（例如空闲 60 秒后）
    // esp_hosted_power_save_timer_start(60 * 1000);
    // 需要时：esp_hosted_power_save_timer_stop();
}
```

> `esp_hosted_power_save_start(DEEP_SLEEP)` 进入 deep sleep 后**不返回**；host 由 slave GPIO 唤醒后会**重启**，因此在 `app_main` 开头用 `esp_hosted_woke_from_power_save()` 判断是否是省电唤醒后的重启，并恢复必要状态。

### 4. 与 Network Split 配合

host 省电常与 Network Split（把部分网络流量留在协处理器、部分交给 host）配合，确保 host 睡眠期间网络仍由协处理器维持。详见 `docs/feature_network_split.md` 与 `examples/host_network_split__power_save/`、`examples/host_shuts_down_slave_to_power_save/`。

## 常见错误

| 错误 | 原因 | 解决方法 |
|---|---|---|
| 唤醒不生效 | 唤醒 GPIO 非 RTC-capable 或被占用 | 选未使用的 RTC GPIO；核对 host 唤醒引脚 |
| host 醒来状态丢失 | deep sleep 后重启 | 用 `esp_hosted_woke_from_power_save()` 分支恢复状态/会话 |
| 用 LIGHT_SLEEP | 当前仅支持 DEEP_SLEEP | LIGHT_SLEEP 标注 Coming Soon |
| 唤醒极性不匹配 | host/slave Level 不一致 | 双方设同一极性并物理相连 |
| 未启用 Kconfig | `power_save_enabled()` 返回 0 | host 与 slave 都开 "Allow host to power save" |

## 参考

- `host/api/include/esp_hosted_power_save.h`
- `docs/feature_host_power_save.md`
- `docs/feature_network_split.md`
- `examples/host_network_split__power_save/`（network split + 省电）
- `examples/host_shuts_down_slave_to_power_save/`
