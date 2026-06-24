# 程序计数器（PC）与栈溢出检测

> **适用摘要**: 用 `bist_pc_test()` 覆盖 PC 寄存器位（IRAM/Flash/RTC 放置），用 `bist_cpu_stack_overflow_init/check/test` 做 0xDEADBEEF 哨兵式栈溢出检测（IEC 60730 组件 1.3 与 4.2）。

## 触发意图

- "PC 自检 / 程序计数器"
- "栈溢出检测"
- "stack sentinel / 0xDEADBEEF"
- "stack high watermark"
- "IEC 60730 1.3 / 4.2"

## 前置条件

| 条件 | 要求 |
|---|---|
| 参考工程 | `tests/pc_test/main.c`、`tests/cpu_stack_test/main.c`、`samples/standalone/main.c` |
| 头文件 | `bist_esp.h`（含 `bist_pc.h`、`bist_cpu_stack.h`） |
| 配置 | `CONFIG_ESP_BIST_PROGRAM_COUNTER_TEST=y`、`CONFIG_ESP_BIST_STACK_TEST=y`、`CONFIG_ESP_BIST_STACK_PROTECTION_BLOCK_SIZE`（默认 `0x100`） |
| 链接器 | `src/soc/<target>/ld/linker.ld` 定义 `_stack_overflow_protection_start`、`_stack_top`、`.pc_test_X` 段 |

## 分步说明

### 1. PC 测试原理（来自 `bist_pc.h` / `src/bist/README.md`）

四个测试函数通过 `__attribute__((section(".pc_test_X")))` 放置到不同存储区，各自返回自身地址，主函数 `bist_pc_test()` 比对返回地址：

| 函数 | 放置区 | 地址范围（示例） | 覆盖的 PC 位 |
|---|---|---|---|
| `pc_test_0` | IRAM 末尾（C3 `0x4038xxxx`） | IRAM 高地址 | 位 2–17 |
| `pc_test_1` | IRAM（C3 `0x403Bxxxx`） | IRAM 末段 | 位 2–17（反相） |
| `pc_test_2` | Flash（ICache，`0x4201xxxx`） | Flash，与 pc_test_1 相差 64 KB（偏移 `0xFFF8`） | 位 19–21、25 |
| `pc_test_3` | RTC（`0x5000xxxx`） | RTC RAM | 位 28 |

> 位 0–1 因 4 字节对齐恒为 0；共覆盖 30 个可寻址位中的 24 位。

地址映射（来自 `src/bist/README.md`）：
- IRAM（C3）`0x40380000–0x403BEE00`；（C6）`0x40800000–0x40880000`
- Flash 2 MB `0x42010000–0x421FFFFF`；4 MB `..–0x423FFFFF`；8 MB `..–0x427FFFFF`
- RTC `0x50000000–0x50001FFF`

### 2. PC 测试需要先初始化 WDT（来自 `tests/pc_test/main.c`）

```c
#include "bist_esp.h"
#include "unity.h"
#include "wdt.h"
void setUp(void) {}
void tearDown(void) {}

void test_BIST_PC(void)
{
    TEST_ASSERT_EQUAL(0, wdt_init(10000));   // PC 测试期间需要看门狗在跑
    bist_esp_err_t ret = bist_pc_test();
    wdt_deinit();
    TEST_ASSERT_EQUAL(BIST_ESP_OK, ret);
}

int main(void) {
    UNITY_BEGIN();
    RUN_TEST(test_BIST_PC);
    return UNITY_END();
}
```

### 3. 栈溢出三件套（来自 `bist_cpu_stack.h`）

```c
// 启动一次：在 _stack_overflow_protection_start 写 0xDEADBEEF
bist_esp_err_t bist_cpu_stack_overflow_init(void);

// 周期检查：哨兵被破坏则返回 BIST_ESP_STACK_TEST_OVERFLOW
bist_esp_err_t bist_cpu_stack_overflow_check(void);

// 压力自检：受限递归（最多 20000 次，每次 128 字节局部缓冲）故意触发溢出
bist_esp_err_t bist_cpu_stack_overflow_test(void);

// 读取栈高水位（仍为填充模式的字节数）
uint32_t bist_get_stack_high_watermark(void);
```

### 4. 压力自检（来自 `tests/cpu_stack_test/main.c`）

```c
#include "bist_esp.h"
#include "unity.h"
void setUp(void) {}
void tearDown(void) {}

void test_BIST_Cpu_Stack_Overflow(void)
{
    TEST_ASSERT_EQUAL(BIST_ESP_OK, bist_cpu_stack_overflow_test());
}

int main(void) {
    UNITY_BEGIN();
    RUN_TEST(test_BIST_Cpu_Stack_Overflow);
    return UNITY_END();
}
```

### 5. 集成进应用主循环（来自 `samples/standalone/main.c`）

```c
// post-boot 阶段：先做压力自检（验证检测机制本身）
bist_esp_err_t err = bist_cpu_stack_overflow_test();
if (err == BIST_ESP_STACK_TEST_ERR) fail_safe_exit();

// 之后初始化哨兵（压力测试已破坏旧哨兵，需重写）
bist_cpu_stack_overflow_init();

// 主循环里周期检查
while (1) {
    /* 应用逻辑 */
    if (bist_cpu_stack_overflow_check() == BIST_ESP_STACK_TEST_OVERFLOW) {
        ESP_LOGE(TAG, "Stack overflow detected");
        fail_safe_exit();
    }
    wdt_feed();
}
```

## 常见错误

| 错误 | 原因 | 解决方法 |
|---|---|---|
| `bist_cpu_stack_overflow_check()` 从不触发 | 漏调 `bist_cpu_stack_overflow_init()` | init 必须在 check 之前 |
| `bist_pc_test()` 报错 | 未先 `wdt_init()` | 参照 `tests/pc_test`：先 `wdt_init(10000)`，测完 `wdt_deinit()` |
| PC 测试覆盖不全 | 自定义链接脚本移动了 `.pc_test_X` 段 | 保持 `src/soc/<target>/ld/linker.ld` 原段定义 |
| 高水位恒为最大值 | 栈从未深度使用 | 属正常；可压测后读取以估算实际占用 |
| 栈溢出误报 | `_stack_overflow_protection_start` 段过小 | 调大 `CONFIG_ESP_BIST_STACK_PROTECTION_BLOCK_SIZE`（默认 `0x100`） |

## 参考

- `tests/pc_test/main.c`、`tests/cpu_stack_test/main.c`
- `src/bist/core/cpu/include/bist_pc.h`、`bist_cpu_stack.h`
- `src/bist/README.md`（PC 内存映射、放置策略；栈哨兵原理）
- `docs/en/module_design_and_coding.rst`（`program-counter-test`、`stack-test` 节）
- `samples/standalone/main.c`（集成顺序：压力测试 → init → 周期 check）
