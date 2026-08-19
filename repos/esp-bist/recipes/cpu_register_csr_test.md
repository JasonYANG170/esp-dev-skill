# CPU 寄存器与 CSR 完整性测试

> **适用摘要**: 用 `bist_cpu_regs_test()` 校验 RISC-V 通用寄存器（X1–X31），用 `bist_cpu_csr_regs_test()` 校验 CSR（trap / PMP / PMA / mexstatus），并按 IEC 60730 组件 1.1 在运行时主循环中周期执行。

> Version: selected repo/component version; align with the user project and dependency manifest.
> Evidence: `repos/esp-bist/resources/`, source/examples in `repos/esp-bist/`, and this recipe path `repos/esp-bist/recipes/cpu_register_csr_test.md`.
> Validation: draft metadata added from repository routing; verify APIs, Kconfig symbols, and component dependencies against the selected version.

## 触发意图

- "CPU 寄存器自检"
- "CSR 测试 / PMP / PMA"
- "stuck-at 故障检测"
- "通用寄存器完整性"
- "IEC 60730 1.1"

## 前置条件

| 条件 | 要求 |
|---|---|
| 参考工程 | `tests/cpu_reg_test/main.c` |
| 头文件 | `bist_esp.h`（含 `bist_cpu_regs.h`、`bist_cpu_csr_regs.h`） |
| 配置 | `CONFIG_ESP_BIST_CPU_REG_TEST=y`、`CONFIG_ESP_BIST_CPU_CSR_REG_TEST=y` |

## 分步说明

### 1. 测试原理（来自 `src/bist/README.md`）

- **通用寄存器**：依次写入 `0xAAAAAAAA` 再 `0x55555555`，回读比对，覆盖所有位的 0/1；栈类寄存器先保存原值再恢复。
- **CSR**：对每个 CSR 施加写掩码后写 `0xAA...`/`0x55...` 并回读；需保留的 CSR 先入栈。掩码已排除写一次位（PMP Lock、PMA Lock）和保留位。

### 2. 各 SoC 覆盖的 CSR（来自 `bist_cpu_csr_regs.h`）

| 分组 | CSR | SoC | 写掩码 |
|---|---|---|---|
| Machine Trap Setup | `mtvec` | 全部 | `0xFFFFFF00` |
| Machine Trap Handling | `mscratch`、`mepc`、`mcause`、`mtval` | 全部 | 寄存器相关 |
| PMP Address | `pmpaddr0–15` | 全部 | C3/C6/H2 `0xFFFFFFFF`；C5 `0x3FFFFFE0`（25 位，128 字节粒度） |
| PMP Config | `pmpcfg0–3` | 全部 | C3/C6/H2 `0x1D1D1D1D`；C5 `0x0D0D0D0D`（额外排除 A[1]，因 G=5 时 NA4 不可选） |
| PMA Address | `pma_addr0–11`（`0xBD0–0xBDB`） | C6/H2/C5（`SOC_CPU_HAS_PMA`） | `0x3FFFFFE0`（25 位，32 字节粒度） |
| Machine Extension | `mexstatus`（`0x7E1`） | 仅 C5 | `0x00102C00`（PBEXE/PPBEXE/NMFT/CLIC_INHV） |
| Machine Hint | `mhint`（`0x7C5`） | 仅 C5 | `0x00100000`（SBE） |

CSR 数量：C3 = 25；C6/H2 = 37（含 12 个 pma_addr）；C5 = 39（再加 mexstatus + mhint）。

> PMA cfg 不测：`PMA_L`（位 29）为写一次，任何置位会永久锁定至下次上电复位。PMA 条目 12–15 不测：ROM bootloader 可能配置为活跃 NAPOT/TOR 区域。

### 3. Unity 单测（来自 `tests/cpu_reg_test/main.c`）

```c
#include "bist_esp.h"
#include "unity.h"
void setUp(void) {}
void tearDown(void) {}

void test_BIST_Cpu_Regs(void)
{
    TEST_ASSERT_EQUAL(BIST_ESP_OK, bist_cpu_regs_test());
}

void test_BIST_Cpu_Csr_Regs(void)
{
    TEST_ASSERT_EQUAL(BIST_ESP_OK, bist_cpu_csr_regs_test());
}

int main(void)
{
    UNITY_BEGIN();
    RUN_TEST(test_BIST_Cpu_Regs);
    RUN_TEST(test_BIST_Cpu_Csr_Regs);
    return UNITY_END();
}
```

### 4. 集成进运行时主循环

```c
bist_esp_err_t err = bist_cpu_regs_test();
if (err == BIST_ESP_CPU_TEST_ERR) {
    ESP_LOGE(TAG, "CPU register test failed");
    fail_safe_exit();
}

err = bist_cpu_csr_regs_test();
if (err == BIST_ESP_CPU_CSR_TEST_ERR) {
    ESP_LOGE(TAG, "CPU CSR register test failed");
    fail_safe_exit();
}
```

### 5. `bist.conf`（来自 `tests/cpu_reg_test/bist.conf`）

```
CONFIG_ESP_BIST_CPU_REG_TEST=y
CONFIG_ESP_BIST_CPU_CSR_REG_TEST=y
CONFIG_ESP_BIST_MEMORY_RAM_TEST=n
CONFIG_ESP_BIST_MEMORY_FLASH_TEST=n
CONFIG_ESP_BIST_STACK_TEST=n
CONFIG_ESP_BIST_CLOCK_TEST=n
CONFIG_ESP_BIST_GPIO_TEST=n
CONFIG_ESP_BIST_PROGRAM_COUNTER_TEST=n
CONFIG_ESP_BIST_WATCHDOG_TEST=n
```

## 常见错误

| 错误 | 原因 | 解决方法 |
|---|---|---|
| `bist_cpu_csr_regs_test` 在自定义代码后失败 | 应用改写了被测 CSR（如 `mtvec`） | 测试会保存/恢复原值；勿在测试运行期间改 CSR |
| 想增加 PMA cfg 测试 | `PMA_L` 写一次，会永久锁死 | 库有意不测；不要自行扩展掩码 |
| C5 上 mexstatus 测试失败 | 自行写了未掩码位 | 用库给定的 `0x00102C00` 掩码，勿改 |
| 误以为 C6/H2 也有 `mhint` | `mhint`（`0x7C5`）在 C6/H2 是非法指令 | 库已用 `SOC_TARGET_ESP32C5` 守卫；勿跨芯片调用 |
| CPU 寄存器测试误报 | 编译器优化破坏内联汇编 | 库已用 `volatile` 与 `BIST_ADD_LABEL`；保持 `-Os`/`-O0` 覆盖不变 |

## 参考

- `tests/cpu_reg_test/main.c`、`tests/cpu_reg_test/bist.conf`
- `src/bist/core/cpu/include/bist_cpu_regs.h`、`bist_cpu_csr_regs.h`
- `src/bist/README.md`（CSR 分组、掩码、覆盖说明）
- `docs/en/module_design_and_coding.rst`（`cpu-register-test` 节：24.8 µs / 992 cycles / 362 指令）
