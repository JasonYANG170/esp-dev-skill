# 时钟测试：32 kHz 外部晶振 + 40 MHz 主晶振

> **适用摘要**: 用 `bist_ext_crystal_fail_test()` 通过 XT WDT 监测外部 32.768 kHz 晶振（仅 `SOC_XT_WDT_SUPPORTED` 的 SoC，如 C3）；用 `bist_main_crystal_test()` 测 40 MHz 主晶振相对 32 kHz 参考的频率漂移（IEC 60730 组件 3）。

> Version: selected repo/component version; align with the user project and dependency manifest.
> Validation: draft metadata added from repository routing; verify APIs, Kconfig symbols, and component dependencies against the selected version.

## 触发意图

- "时钟自检 / clock test"
- "32 kHz 晶振 / XT WDT"
- "40 MHz 主晶振漂移"
- "frequency drift"
- "IEC 60730 3"

## 前置条件

| 条件 | 要求 |
|---|---|
| 参考工程 | `tests/clock_test/main.c` |
| 头文件 | `bist_esp.h`（含 `bist_clock_fail.h`） |
| 配置 | `CONFIG_ESP_BIST_CLOCK_TEST=y`、`CONFIG_ESP_BIST_CLOCK_PERCENT_FREQUENCY_DRIFT`（默认 1，范围 0–100） |
| 硬件 | 主晶振测试需要正常工作的 32 kHz 晶振作为参考 |

## 分步说明

### 1. 两个测试函数（来自 `bist_clock_fail.h`）

```c
// 32.768 kHz 外部晶振：有 XT WDT 的 SoC 用 200 周期超时监测；
// 无 XT WDT 的 SoC（如 C6/H2）直接返回 BIST_ESP_OK（跳过）
bist_esp_err_t bist_ext_crystal_fail_test(void);

// 40 MHz 主晶振：测 500 个周期内 40 MHz / 32 kHz 的比值，
// 漂移超过 CONFIG_ESP_BIST_CLOCK_PERCENT_FREQUENCY_DRIFT 则失败
bist_esp_err_t bist_main_crystal_test(void);
```

### 2. SoC 差异（来自 `bist_clock_fail.h` / `src/bist/README.md`）

| SoC | XT WDT 支持 | `bist_ext_crystal_fail_test()` 行为 |
|---|---|---|
| ESP32-C3 | 是（`SOC_XT_WDT_SUPPORTED`） | 监测 32 kHz；失败 200 周期触发中断 → 返回 `BIST_ESP_CLOCK_TEST_ERR` |
| ESP32-C6 / H2 / C5 | 否 | 跳过，返回 `BIST_ESP_OK`。改用 `bist_main_crystal_test()` 验证主晶振 |

### 3. 单测（来自 `tests/clock_test/main.c`）

```c
#include "bist_esp.h"
#include "unity.h"
#include "rom/ets_sys.h"
void setUp(void) {}
void tearDown(void) {}

void test_BIST_ext_crystal_fail(void) {
    TEST_ASSERT_EQUAL(BIST_ESP_OK, bist_ext_crystal_fail_test());
}
void test_BIST_main_crystal(void) {
    TEST_ASSERT_EQUAL(BIST_ESP_OK, bist_main_crystal_test());
}

int main(void) {
    UNITY_BEGIN();
    RUN_TEST(test_BIST_ext_crystal_fail);
    RUN_TEST(test_BIST_main_crystal);
    return UNITY_END();
}
```

### 4. 集成进 post-boot 自检（来自 `samples/standalone/main.c`）

```c
static void post_boot_tests(void)
{
    bist_esp_err_t err;

    err = bist_ext_crystal_fail_test();   // C6/H2 自动跳过
    if (err == BIST_ESP_CLOCK_TEST_ERR) {
        ESP_LOGE(TAG, "External crystal fail test failed");
        fail_safe_exit();
    }

    err = bist_main_crystal_test();       // 始终验证 40 MHz
    if (err == BIST_ESP_CLOCK_TEST_ERR) {
        ESP_LOGE(TAG, "Main crystal test failed");
        fail_safe_exit();
    }
}
```

### 5. 注册 XT WDT 回调（仅 C3，来自 `samples/standalone/main.c`）

```c
#if SOC_XT_WDT_SUPPORTED
    esp_xt_wdt_register_callback((esp_xt_callback_t)fail_safe_exit, NULL);
#endif
```
> `esp_xt_wdt.h` 来自 IDF HAL，非 BIST 树；务必用 `SOC_XT_WDT_SUPPORTED` 守卫。

### 6. 调整漂移容差

```
# bist.conf
CONFIG_ESP_BIST_CLOCK_TEST=y
CONFIG_ESP_BIST_CLOCK_PERCENT_FREQUENCY_DRIFT=1     # 默认 ±1%，可按晶振规格放宽
```

## 常见错误

| 错误 | 原因 | 解决方法 |
|---|---|---|
| C6/H2 上 32 kHz 测不出晶振故障 | 无 XT WDT，测试被跳过 | 改用 `bist_main_crystal_test()`；或外接监测电路 |
| 主晶振测试误报漂移 | 32 kHz 参考晶振本身异常 | 先确认 32 kHz 晶振硬件；适当放宽 `CONFIG_ESP_BIST_CLOCK_PERCENT_FREQUENCY_DRIFT` |
| `esp_xt_wdt_register_callback` 在 C6 编译失败 | 该 SoC 无此 HAL | 用 `#if SOC_XT_WDT_SUPPORTED` 包裹 |
| 主晶振测试超时 | 500 周期测量窗口被中断打断 | 测试期间避免高优先级中断；放 post-boot 早期执行 |
| 漂移百分比设为 0 | 任何微小偏差都判失败 | 按晶振 datasheet 设合理值（典型 1%） |

## 参考

- `tests/clock_test/main.c`
- `src/bist/core/clock/include/bist_clock_fail.h`
- `src/bist/README.md`（Clock Testing 节）
- `samples/standalone/main.c`（post-boot 调用 + XT WDT 回调注册）
- `src/bist/Kconfig`（`CONFIG_ESP_BIST_CLOCK_PERCENT_FREQUENCY_DRIFT`）
