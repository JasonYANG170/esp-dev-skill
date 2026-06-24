# LP subcore 自动 Light Sleep 低功耗

> **适用摘要**: 在 ESP32-C5 / ESP32-C6（LP subcore）上启用 HP maincore 自动 light sleep，LP subcore 与 RTC RAM 在睡眠期间保持供电，仅在访问 HP RAM/外设时唤醒 maincore。适合电池供电的低功耗场景。**仅 LP subcore 支持，ESP32-P4 不支持。**

## 触发意图

- "light sleep"
- "低功耗"
- "LP subcore 省电"
- "pure RTC RAM app"
- "ESP-AMP 自动睡眠"

## 前置条件

| 条件 | 要求 |
|---|---|
| 参考工程 | `examples/light_sleep/` |
| 目标 | 仅 ESP32-C5 / ESP32-C6（LP subcore）；ESP32-P4 不支持 |
| Kconfig | `CONFIG_ESP_AMP_SYSTEM_AUTO_LIGHT_SLEEP_SUPPORT_ENABLE=y`、`CONFIG_PM_ENABLE=y` |

## 分步说明

### 1. 启用配置（sdkconfig）

```
CONFIG_PM_ENABLE=y
CONFIG_ESP_AMP_SYSTEM_AUTO_LIGHT_SLEEP_SUPPORT_ENABLE=y
# 可选：强制整个 subcore 应用放入 RTC RAM（适合纯低功耗应用）
# CONFIG_ESP_AMP_SUBCORE_BUILD_TYPE_PURE_RTC_RAM_APP=y
```

### 2. maincore 配置电源管理（在 EVENT_SUBCORE_READY 之后调用）

```c
#include "esp_pm.h"
#include "esp_amp.h"

void app_main(void)
{
    assert(esp_amp_init() == 0);
    /* 加载启动 subcore + 握手 ... */
    assert((esp_amp_event_wait(EVENT_SUBCORE_READY, true, true, 10000)
            & EVENT_SUBCORE_READY) == EVENT_SUBCORE_READY);

    /* 在握手后再配置 PM，避免 subcore 初始化期间访问 HP RAM 时 maincore 已睡 */
    esp_pm_config_t pm_config = {
        .max_freq_mhz = CONFIG_ESP_DEFAULT_CPU_FREQ_MHZ,
#if CONFIG_IDF_TARGET_ESP32C5
        .min_freq_mhz = 48,
#else
        .min_freq_mhz = CONFIG_XTAL_FREQ,
#endif
        .light_sleep_enable = true,
    };
    ESP_ERROR_CHECK(esp_pm_configure(&pm_config));
}
```

### 3. subcore 访问 HP RAM/外设必须 skip/resume 成对

```c
#include "esp_amp.h"

/* ESP-AMP 大多数 IPC API 内部已自动包裹；仅在你直接访问 HP RAM/外设时手动包裹 */
void read_hp_peripheral(void)
{
    ESP_AMP_PM_SKIP_LIGHT_SLEEP_ENTER();   // 展开为 esp_amp_system_pm_subcore_skip_light_sleep()
    /* 唤醒 maincore，安全访问 HP 寄存器/HP RAM */
    uint32_t val = READ_PERI_REG(...);
    ESP_AMP_PM_SKIP_LIGHT_SLEEP_EXIT();    // 展开为 esp_amp_system_pm_subcore_resume_light_sleep()
}
```

### 4. subcore 内存放置：ESP 属性宏

```c
/* ISR 必须用 RTC_IRAM_ATTR（light sleep 期间才能执行）；不要用 IRAM_ATTR */
RTC_IRAM_ATTR void lp_isr(void) { ... }

RTC_DATA_ATTR static int counter = 0;          /* 数据放 RTC RAM */
RTC_RODATA_ATTR static const char msg[] = "hi"; /* 常量放 RTC RAM */
```

### 5. subcore 内存放置：linker fragment（批量放置整个组件）

```lf
# subcore/main/main.lf —— 把整个 libmain.a 放入 RTC RAM（rtc scheme）
[mapping:main]
archive: libmain.a
entries:
    * (rtc)
```
> ESP-AMP 定义两个 scheme：`rtc`（放 RTC RAM）与 `default`（放 HP RAM）。`CONFIG_ESP_AMP_SUBCORE_BUILD_TYPE_PURE_RTC_RAM_APP=y` 等价于把所有 section 放入 RTC RAM，免去手写 lf 文件。

### 6. 组件 light sleep 安全性（来自官方文档）

| 组件 | 自动 light sleep 是否安全 | 注意 |
|---|---|---|
| SysInfo（HP RAM） | 不安全 | HP core 睡眠时访问会挂起 LP subcore |
| SysInfo（RTC RAM） | 安全 | 但不支持原子操作 |
| Software Interrupt | 安全 | LP ISR 尽量短 |
| Event | 部分安全 | 避免 `esp_amp_event_poll()`（忙等阻止睡眠） |
| Virtqueue | 安全 | 仅访问数据时唤醒 |
| RPMsg / RPC | 安全 | 基于 Virtqueue |

### 7. 调试：profiling light sleep 时间

```c
static esp_err_t enter_cb(int64_t t, void *a) { ESP_EARLY_LOGW("", "sleep %lld", t); return ESP_OK; }
static esp_err_t exit_cb(int64_t t, void *a)  { ESP_EARLY_LOGW("", "wake %lld", t);  return ESP_OK; }
esp_pm_sleep_cbs_register_config_t cbs = {
    .enter_cb = enter_cb, .exit_cb = exit_cb,
    .enter_cb_prior = 5, .exit_cb_prior = 5,
};
ESP_ERROR_CHECK(esp_pm_light_sleep_register_cbs(&cbs));
/* 注意：回调内不要调用阻塞 API */
```

## 常见错误

| 错误 | 原因 | 解决方法 |
|---|---|---|
| LP subcore 卡死 | HP core 睡眠时访问 HP RAM/外设 | 用 `ESP_AMP_PM_SKIP_LIGHT_SLEEP_ENTER/EXIT` 包裹访问 |
| maincore 永远不再睡眠 | skip/resume 不成对 | 必须成对；检查每处 ENTER 都有对应 EXIT |
| ISR 在睡眠时不执行 | 用了 `IRAM_ATTR`（放 HP RAM，睡眠时 clock-gated） | 改用 `RTC_IRAM_ATTR` |
| heap 访问阻止睡眠 | heap 在 HP RAM | light sleep 启用时避免 malloc；必要时包裹 skip/resume |
| pure RTC RAM 应用仍访问 HP RAM | 该选项不禁止运行时访问 | 共享内存仍在 HP RAM；需手动 skip/resume |
| binary 放不进 RTC RAM | RTC RAM 仅 16KB | 参考官方文档减小体积；或用 `CONFIG_ESP_AMP_SUBCORE_USE_HP_MEM=y` 部分放 HP RAM（牺牲睡眠） |

## 参考

- `examples/light_sleep/` — linker fragment + PM 配置完整示例
- `espressif-repos/esp-amp/docs/light_sleep.md`
- `espressif-repos/esp-amp/docs/subcore_build_tips.md`
