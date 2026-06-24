# 性能指标采集：RISC-V Performance Counter CSRs

> **适用摘要**: 用 `bist_metrics.h` 提供的 `BIST_METRICS_*` 宏，基于 RISC-V 性能计数器 CSR（`0x7e0` PCER / `0x7e1` PCMR / `0x7e2` PCCR）测量 BIST 测试的周期数、指令数、load/store、分支、hazard 等微架构事件。

## 触发意图

- "BIST 性能测量"
- "cycles / instructions 计数"
- "性能计数器 CSR"
- "test 执行耗时"
- "BIST_METRICS"

## 前置条件

| 条件 | 要求 |
|---|---|
| 头文件 | `bist_metrics.h`（依赖 `riscv/csr.h`、`bist_conf.h`、`bist_log.h`） |
| 配置 | `CONFIG_ESP_BIST_METRICS=y`（默认 `y`） |
| 目标 | RISC-V SoC（C3/C5/C6/H2 等） |

## 分步说明

### 1. CSR 与事件模式（来自 `bist_metrics.h`）

| CSR | 地址 | 用途 |
|---|---|---|
| `CSR_PCER` | `0x7e0` | Performance Counter Event Register（选择事件） |
| `CSR_PCMR` | `0x7e1` | Performance Counter Mode Register（active/always counting） |
| `CSR_PCCR` | `0x7e2` | Performance Counter Count Register（计数值） |

事件模式（`bist_metrics_mode_t`，同一时刻只能测一个）：

| 值 | 名称 | 含义 |
|---|---|---|
| 0 | `BIST_METRICS_MODE_CYCLE` | CPU 周期 |
| 1 | `BIST_METRICS_MODE_INST` | 退役指令 |
| 2 | `BIST_METRICS_MODE_LD_HAZARDS` | Load-use hazard |
| 3 | `BIST_METRICS_MODE_JMP_HAZARDS` | Jump-register hazard |
| 4 | `BIST_METRICS_MODE_IDLE` | 等待内存周期 |
| 5 | `BIST_METRICS_MODE_LOAD` | Load 指令 |
| 6 | `BIST_METRICS_MODE_STORE` | Store 指令 |
| 7 | `BIST_METRICS_MODE_JMP_UNCOND` | 无条件跳转 |
| 8 | `BIST_METRICS_MODE_BRANCH` | 分支指令 |
| 9 | `BIST_METRICS_MODE_BRANCH_TAKEN` | 命中的分支 |
| 10 | `BIST_METRICS_MODE_INST_COMP` | 压缩指令 |

### 2. 数据结构与宏（来自 `bist_metrics.h`）

```c
typedef struct {
    uint32_t start_value;
    uint32_t end_value;
    bool     valid;
} bist_metrics_t;

#define BIST_METRICS_INIT(mode_)                        \
    do {                                                \
        RV_WRITE_CSR(CSR_PCER, (1u << (mode_)));        \
        RV_WRITE_CSR(CSR_PCMR, 1u);                     \
        RV_WRITE_CSR(CSR_PCCR, 0u);                     \
    } while (0)

#define BIST_METRICS_BEGIN(metrics_)                    \
    do {                                                \
        (metrics_).valid = false;                       \
        (metrics_).start_value = RV_READ_CSR(CSR_PCCR); \
    } while (0)

#define BIST_METRICS_END(metrics_)                      \
    do {                                                \
        (metrics_).end_value = RV_READ_CSR(CSR_PCCR);   \
        (metrics_).valid = true;                        \
    } while (0)

// 输出格式："METRICS: <name> Mode: <mode> delta=<count>"
#define BIST_METRICS_PRINT(name_, metrics_) /* 见头文件 */
```

### 3. 测量一段 BIST 测试

```c
#include "bist_esp.h"
#include "bist_log.h"
#include "bist_metrics.h"

static const char *TAG = "metrics";

void measure_ram_test(void)
{
    bist_metrics_t m;

    BIST_METRICS_INIT(BIST_METRICS_MODE_CYCLE);   // 选事件：CPU 周期
    BIST_METRICS_BEGIN(m);
    bist_ram_test_march_a();                       // 被测代码
    BIST_METRICS_END(m);

    BIST_METRICS_PRINT("ram_test_march_a", m);     // 控制台输出 delta
}
```

### 4. 测多个事件（每次只能一个）

```c
void measure_multiple(void)
{
    bist_metrics_t m;

    BIST_METRICS_INIT(BIST_METRICS_MODE_INST);
    BIST_METRICS_BEGIN(m);
    bist_cpu_regs_test();
    BIST_METRICS_END(m);
    BIST_METRICS_PRINT("cpu_regs_inst", m);

    BIST_METRICS_INIT(BIST_METRICS_MODE_LOAD);
    BIST_METRICS_BEGIN(m);
    bist_cpu_regs_test();
    BIST_METRICS_END(m);
    BIST_METRICS_PRINT("cpu_regs_load", m);
}
```

### 5. 解析输出

日志格式（CI 可解析）：
```
I (xxx) bist_metrics: METRICS: ram_test_march_a Mode: 0 delta=12345
```
- `Mode` 对应 `bist_metrics_mode_t` 枚举值（由 PCER 当前位推导）
- `delta` = `end_value - start_value`，即该事件在该代码段的累计计数

> 参考量级（来自 `docs/en/module_design_and_coding.rst`）：`bist_cpu_regs_test` ≈ 24.8 µs / 992 cycles / 362 instructions / 1346 字节代码。可据此校验你的测量。

## 常见错误

| 错误 | 原因 | 解决方法 |
|---|---|---|
| `delta` 恒为 0 | 未先 `BIST_METRICS_INIT` | INIT 配置 PCER/PCMR/PCCR，必须先调 |
| `BIST_METRICS_PRINT` 报 "Invalid metrics" | 未成对调用 BEGIN/END | 保证 END 在 BEGIN 之后 |
| 想同时测多个事件 | 硬件只有单个 PCCR | 串行测量，每次 INIT 一个事件 |
| `valid` 为 false | BEGIN 后未 END（如被测代码异常退出） | 确保被测路径正常返回 |
| 在中断里测量 | 中断打断会污染计数 | 仅在主流程、关中断窗口内测量 |
| Mode 解析错误 | 自行改写了 PCER 位布局 | 用 `BIST_METRICS_INIT(mode)` 设事件，让 PRINT 自动推导 |

## 参考

- `src/bist/include/bist_metrics.h`（CSR 定义、枚举、四个宏）
- `src/bist/Kconfig`（`CONFIG_ESP_BIST_METRICS`）
- `docs/en/module_design_and_coding.rst`（各测试的 cycles/instructions/code-size 参考表）
- `docs/en/software_architecture.rst`（Performance Metrics 模块描述）
