# 看门狗测试：复位验证 + 窗口看门狗

> **适用摘要**: 用 `bist_wdt_test()` 通过双启动序列验证 MWDT 能触发复位（IEC 60730 6.3）；用 `wdt_init()` + `wdt_init_windowed()` + `wdt_feed()` 实现下溢/溢出双向监控，并按 `tests/windowed_wdt_test` 的三个用例验证。

## 触发意图

- "看门狗自检 / WDT"
- "窗口看门狗 / windowed WDT"
- "下溢 / underflow 检测"
- "喂狗太早 / 太晚"
- "IEC 60730 6.3 时序监控"

## 前置条件

| 条件 | 要求 |
|---|---|
| 参考工程 | `tests/wdt_test/main.c`、`tests/windowed_wdt_test/main.c` |
| 头文件 | `wdt.h`、`bist_esp.h` |
| 配置 | `CONFIG_ESP_BIST_WDT_TIMEOUT_US`（默认 50000）、`CONFIG_ESP_BIST_WDT_WINDOWED_UNDERFLOW_TIMEOUT_US`（默认 10） |

## 分步说明

### 1. WDT 复位测试是双启动序列（来自 `bist_wdt.h` / `tests/wdt_test/main.c`）

首次执行：以 100 µs 超时初始化 MWDT，等 1000 µs，触发复位。再次启动：检查复位原因是否为 `RESET_REASON_CORE_MWDT0`。

```c
#include "bist_esp.h"
#include "unity.h"
#include "wdt.h"
void setUp(void) {}
void tearDown(void) {}

void test_BIST_WDT(void)
{
    TEST_ASSERT_EQUAL(BIST_ESP_OK, bist_wdt_test());
}

void test_BIST_WDT_INIT_SUB_TICK_TIMEOUT(void)
{
    /* 1 us 低于一个 RTC tick（32768 Hz 时约 31 us） */
    TEST_ASSERT_EQUAL_INT(-1, wdt_init(1));
}

int main(void) {
    UNITY_BEGIN();
    RUN_TEST(test_BIST_WDT);
    RUN_TEST(test_BIST_WDT_INIT_SUB_TICK_TIMEOUT);
    return UNITY_END();
}
```

### 2. WDT 驱动 API（来自 `wdt.h`）

```c
int  wdt_init(uint32_t timeout_us);                       // ≥ 500 us；返回 0/-1
void wdt_deinit(void);
void wdt_feed(void);
void wdt_register_callback(void (*cb)(void *), void *arg);// 注册 MWDT 超时前回调
int  wdt_init_windowed(uint32_t underflow_timeout_us);    // 启用下溢窗口；返回 0/-1
void wdt_windowed_deinit(void);
bool wdt_is_underflow_detected(void);                     // 喂狗过早时为 true
```

### 3. 窗口看门狗工作原理（来自 `src/bist/README.md`）

- 私有定时器（`CONFIG_ESP_BIST_WDT_UNDERFLOW_US` / `wdt_init_windowed` 入参）定义窗口**下界**
- 主 MWDT（`CONFIG_ESP_BIST_WDT_TIMEOUT_US` / `wdt_init` 入参）定义窗口**上界**
- 喂狗过早（早于下界）：私有定时器标记下溢，阻止本次喂狗
- 喂狗过晚（晚于上界）：MWDT 触发中断 → 复位
- 窗口内喂狗：正常

### 4. 三个验证用例（来自 `tests/windowed_wdt_test/main.c`）

```c
#include "bist_esp.h"
#include "unity.h"
#include "wdt.h"
#include "rom/ets_sys.h"

#define OVERFLOW_US    50000
#define UNDERFLOW_US   2000
#define FEED_MARGIN_US 1000
#define CONSECUTIVE    10

void setUp(void) {
    TEST_ASSERT_EQUAL(0, wdt_init(OVERFLOW_US));
    TEST_ASSERT_EQUAL(0, wdt_init_windowed(UNDERFLOW_US));
}
void tearDown(void) {
    wdt_windowed_deinit();
    wdt_deinit();
}

// (a) 正常：等待 >= 下界后喂狗，不应报下溢
void test_BIST_WINDOWED_WDT_NORMAL(void) {
    ets_delay_us(UNDERFLOW_US + FEED_MARGIN_US);
    wdt_feed();
    TEST_ASSERT_FALSE(wdt_is_underflow_detected());

    ets_delay_us(UNDERFLOW_US + FEED_MARGIN_US);
    wdt_feed();
    TEST_ASSERT_FALSE(wdt_is_underflow_detected());
}

// (b) 下溢：首次喂狗清掉 window_open_flag 后立即再喂 → 必须报下溢
void test_BIST_WINDOWED_WDT_UNDERFLOW(void) {
    wdt_feed();                 // 首次允许（init 时 window_open_flag=true）
    ets_delay_us(100);          // 远早于下界
    wdt_feed();
    TEST_ASSERT_TRUE(wdt_is_underflow_detected());
}

// (c) 连续：连续 N 次窗口内喂狗都不报下溢
void test_BIST_WINDOWED_WDT_CONSECUTIVE(void) {
    for (int i = 0; i < CONSECUTIVE; i++) {
        ets_delay_us(UNDERFLOW_US + FEED_MARGIN_US);
        wdt_feed();
        TEST_ASSERT_FALSE(wdt_is_underflow_detected());
    }
}

int main(void) {
    UNITY_BEGIN();
    RUN_TEST(test_BIST_WINDOWED_WDT_NORMAL);
    RUN_TEST(test_BIST_WINDOWED_WDT_UNDERFLOW);
    RUN_TEST(test_BIST_WINDOWED_WDT_CONSECUTIVE);
    return UNITY_END();
}
```

### 5. 集成进应用（来自 `samples/standalone/main.c`）

```c
void IRAM_ATTR wdt_callback(void *args) {
    ESP_LOGE(TAG, "User WDT callback triggered");
}

// 在 post_boot 与 GPIO 配置之后：
wdt_register_callback(wdt_callback, NULL);                 // 先注册回调
if (wdt_init(CONFIG_ESP_BIST_WDT_TIMEOUT_US) != 0) fail_safe_exit();
if (wdt_init_windowed(CONFIG_ESP_BIST_WDT_WINDOWED_UNDERFLOW_TIMEOUT_US) != 0)
    fail_safe_exit();

while (1) {
    /* 应用逻辑（耗时 >= 下界） */
    runtime_tests();
    wdt_feed();
    if (wdt_is_underflow_detected()) fail_safe_exit();
}
```

## 常见错误

| 错误 | 原因 | 解决方法 |
|---|---|---|
| `wdt_init(1)` 返回 `-1` | 低于一个 MWDT tick（≈ 31 µs） | 至少传 500 µs |
| `bist_wdt_test()` 单次跑不过 | 首次执行必触发复位 | 接受双启动设计；成功仅在第二次启动由复位原因判定 |
| 频繁报下溢 | 主循环快于 `UNDERFLOW_TIMEOUT_US` | 调大下界或循环内 `ets_delay_us()` |
| WDT 回调崩溃 | 未放 IRAM / 访问 flash | 加 `IRAM_ATTR`，回调只做极简日志 |
| `wdt_init_windowed(0)` 失败 | 入参为 0 | 传非零下界值 |
| 连续喂狗中偶发下溢 | 某次循环快于下界 | 增大 `FEED_MARGIN_US` 或下界 |

## 参考

- `tests/wdt_test/main.c`、`tests/windowed_wdt_test/main.c`
- `src/bist/core/wdt/include/bist_wdt.h`、`src/bist/drivers/include/wdt.h`
- `src/bist/README.md`（Windowed Watchdog、间接时隙监控）
- `samples/standalone/main.c`（集成顺序）
