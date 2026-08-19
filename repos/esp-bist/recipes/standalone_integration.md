# 完整 IEC 60730 集成：post-boot + 运行时自检 + 窗口看门狗

> **适用摘要**: 按 `samples/standalone/main.c` 的范式，把 BIST 库完整集成进一个安全关键应用：启动自检、运行时周期自检、窗口看门狗、fail-safe 退出。

> Version: selected repo/component version; align with the user project and dependency manifest.
> Evidence: `repos/esp-bist/resources/`, source/examples in `repos/esp-bist/`, and this recipe path `repos/esp-bist/recipes/standalone_integration.md`.
> Validation: draft metadata added from repository routing; verify APIs, Kconfig symbols, and component dependencies against the selected version.

## 触发意图

- "IEC 60730 Class B 集成"
- "BIST 怎么放进主循环"
- "fail-safe / 安全状态"
- "窗口看门狗 + BIST"
- "post-boot tests"

## 前置条件

| 条件 | 要求 |
|---|---|
| 参考工程 | `samples/standalone/main.c` |
| 头文件 | `bist_esp.h`、`bist_log.h`、`wdt.h`、`gpio.h` |
| 配置 | `CONFIG_ESP_BIST_WDT_TIMEOUT_US`、`CONFIG_ESP_BIST_WDT_WINDOWED_UNDERFLOW_TIMEOUT_US` |

## 分步说明

### 1. fail-safe 处理与 WDT 回调

```c
#include "bist_esp.h"
#include "bist_log.h"
#include "wdt.h"
#include "esp_xt_wdt.h"
#include "gpio.h"
#include "rom/ets_sys.h"
#include "esp_attr.h"
#include "soc/soc_caps.h"

#define LED_GPIO 7
#define BTN_GPIO 9
static const char *TAG = "app";

static void fail_safe_exit(void)
{
    ESP_LOGE(TAG, "Fail safe exit");
    while (1) { ; }              // 已知安全状态；若 WDT 仍在跑会触发复位
}

void IRAM_ATTR wdt_callback(void *args)
{
    ESP_LOGE(TAG, "User WDT callback triggered");
}
```

### 2. post-boot 自检（一次性，启动时）

```c
static void post_boot_tests(void)
{
    bist_esp_err_t err;

    err = bist_ext_crystal_fail_test();        // C6/H2 无 XT WDT 时自动跳过 → OK
    if (err == BIST_ESP_CLOCK_TEST_ERR) { ESP_LOGE(TAG, "Ext crystal"); fail_safe_exit(); }

    err = bist_main_crystal_test();            // 40 MHz 漂移 ≤ CONFIG_ESP_BIST_CLOCK_PERCENT_FREQUENCY_DRIFT
    if (err == BIST_ESP_CLOCK_TEST_ERR) { ESP_LOGE(TAG, "Main crystal"); fail_safe_exit(); }

    err = bist_ram_test_march_a();             // 非破坏式：备份到 .dram0.safe_ram
    if (err == BIST_ESP_RAM_TEST_ERR) { ESP_LOGE(TAG, "RAM"); fail_safe_exit(); }

    err = bist_flash_test();                   // 依赖后处理注入的 CRC
    if (err == BIST_ESP_FLASH_TEST_ERR) { ESP_LOGE(TAG, "Flash"); fail_safe_exit(); }

    err = bist_cpu_stack_overflow_test();      // 故意溢出以验证检测机制
    if (err == BIST_ESP_STACK_TEST_ERR) { ESP_LOGE(TAG, "Stack"); fail_safe_exit(); }

    err = bist_gpio_output_test(LED_GPIO);
    if (err == BIST_ESP_IO_TEST_ERR) { ESP_LOGE(TAG, "GPIO out"); fail_safe_exit(); }

    err = bist_gpio_input_test(BTN_GPIO, 1);
    if (err == BIST_ESP_IO_TEST_ERR) { ESP_LOGE(TAG, "GPIO in"); fail_safe_exit(); }

    ESP_LOGI(TAG, "All post boot tests passed");
}
```

> `bist_wdt_test()` 是双启动序列（首次触发复位，下次校验复位原因）。standalone 例程未在 post_boot 中默认调用它；如需纳入，参见 `recipes/watchdog_windowed_test.md`。

### 3. 运行时周期自检（每个主循环迭代）

```c
static void runtime_tests(void)
{
    bist_esp_err_t err;

    err = bist_cpu_regs_test();
    if (err == BIST_ESP_CPU_TEST_ERR) { ESP_LOGE(TAG, "CPU reg"); fail_safe_exit(); }

    err = bist_cpu_csr_regs_test();
    if (err == BIST_ESP_CPU_CSR_TEST_ERR) { ESP_LOGE(TAG, "CSR"); fail_safe_exit(); }

    err = bist_cpu_stack_overflow_check();     // 检查 0xDEADBEEF 哨兵
    if (err == BIST_ESP_STACK_TEST_OVERFLOW) { ESP_LOGE(TAG, "Stack overflow"); fail_safe_exit(); }

    err = bist_pc_test();
    if (err == BIST_ESP_PC_TEST_ERR) { ESP_LOGE(TAG, "PC"); fail_safe_exit(); }
}
```

### 4. `main()`：初始化顺序与主循环

```c
static void configure_led(void)     { gpio_reset_pin(LED_GPIO); gpio_set_direction(LED_GPIO, GPIO_MODE_OUTPUT); }
static void configure_button(void)  { gpio_set_direction(BTN_GPIO, GPIO_MODE_INPUT); }
static void set_led(bool s)         { gpio_set_level(LED_GPIO, s); }
static bool get_button(void)        { return gpio_get_level(BTN_GPIO); }

int main(void)
{
    ESP_LOGI(TAG, "BIST standalone start");

    post_boot_tests();

    bist_cpu_stack_overflow_init();           // 必须在 runtime check 之前

#if SOC_XT_WDT_SUPPORTED
    esp_xt_wdt_register_callback((esp_xt_callback_t)fail_safe_exit, NULL);
#endif
    wdt_register_callback(wdt_callback, NULL);

    runtime_tests();                          // 验证一次检测路径
    configure_led();
    configure_button();

    if (wdt_init(CONFIG_ESP_BIST_WDT_TIMEOUT_US) != 0) fail_safe_exit();
    if (wdt_init_windowed(CONFIG_ESP_BIST_WDT_WINDOWED_UNDERFLOW_TIMEOUT_US) != 0) fail_safe_exit();

    while (1) {
        set_led(get_button());                // 应用逻辑
        runtime_tests();                       // 连续监控
        wdt_feed();                            // 必须在窗口内喂狗
    }
    return 0;
}
```

### 5. 执行流（state machine）

```
Boot → post_boot_tests ──fail──► fail_safe_exit()
                 │ pass
                 ▼
   bist_cpu_stack_overflow_init()
   注册 XT WDT / MWDT 回调
   configure_gpio() → wdt_init() → wdt_init_windowed()
                 │
                 ▼
   ┌── 主循环 ──────────────────────┐
   │ 应用逻辑 → runtime_tests() → wdt_feed() │
   └──────────── loop ─────────���───┘
```

## 常见错误

| 错误 | 原因 | 解决方法 |
|---|---|---|
| 运行时栈溢出从不触发 | 漏调 `bist_cpu_stack_overflow_init()` | post-boot 之后立即初始化哨兵 |
| WDT 回调里崩 | 回调未放 IRAM / 访问 flash | 加 `IRAM_ATTR`，回调只做极简日志 |
| 窗口看门狗频繁报下溢 | 主循环快于 `UNDERFLOW_TIMEOUT_US` | 调大下溢值或循环内加 `ets_delay_us()` |
| `bist_gpio_input_test` 失败 | 按键未接到期望电平 | 确认硬件：BTN 接 GND 则期望 0，接 VDD 则期望 1 |
| C6/H2 上 `esp_xt_wdt_register_callback` 编译失败 | 该 SoC 无 XT WDT | 用 `#if SOC_XT_WDT_SUPPORTED` 包裹 |

## 参考

- `samples/standalone/main.c`（完整范式）
- `docs/en/application_guide.rst`（IEC 60730 组件映射、执行顺序、时序约束）
- `docs/en/software_architecture.rst`（三层架构、IRAM 放置、`.dram0.safe_ram`）
