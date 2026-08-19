# 安全认证证据工作流：QEMU GDB 故障注入 + 追溯矩阵 + 覆盖率 + 工具鉴定

> **适用摘要**: 按 `docs/en/software_validation.rst` 的范式，用 QEMU（`qemu-system-riscv32 -icount 3`）+ GDB（`:1234`）对 BIST 测试做确定性故障注入，用 pytest 驱动断言 PASS/FAIL，产出 JUnit XML；再把结果串进 `test_traceability_matrix.rst` → `coverage_analysis.rst` → `tool_qualification.rst` → `safety_case_summary.rst` 的 IEC 60730 Class B 证据链。

> Version: selected repo/component version; align with the user project and dependency manifest.

## 触发意图

- "IEC 60730 证据包"
- "GDB 故障注入"
- "QEMU 跑 BIST 测试"
- "pytest_qemu_*"
- "追溯矩阵 / traceability"
- "safety case / 安全案例"
- "工具鉴定 / tool qualification"

## 前置条件

| 条件 | 要求 |
|---|---|
| 参考测试 | `tests/cpu_reg_test/pytest_qemu_cpu_reg_test.py`、`tests/ram_test/pytest_qemu_ram_test.py`、`tests/pc_test/pytest_qemu_pc_test.py`、`tests/wdt_test/pytest_qemu_wdt_test.py` |
| pytest 基础设施 | `conftest.py`（`QEMU_RISCV`、`GDB_RISCV` 类 + `qemu_instance`/`qemu_debug_instance`/`gdb_instance` fixture）、`tests/idf_targets.py`（4 目标参数化） |
| 工具 | `qemu-system-riscv32`、`riscv32-esp-elf-gdb`、pytest、Unity |
| 证据文档 | `docs/en/software_validation.rst`、`test_traceability_matrix.rst`、`coverage_analysis.rst`、`tool_qualification.rst`、`safety_case_summary.rst` |
| 调试脚本样例 | `samples/standalone/gdbinit`（OpenOCD `:3333` 范式；QEMU 用 `:1234`） |

## 分步说明

### 1. 双轨验证策略（来自 `software_validation.rst`）

| 轨道 | 环境 | 覆盖 | 用途 |
|---|---|---|---|
| QEMU 仿真 | `qemu-system-riscv32 -machine <target> -icount 3` + GDB `:1234` | CPU 寄存器、CSR、栈、RAM、Flash、PC、WDT、窗口 WDT、esp_timer | 确定性、可复现、CI 友好；GDB 注入故障走错误路径 |
| 硬件实测 | 真机串口 | 全部模块（时钟/GPIO 仅硬件） | 真实工况验证 |

> 时钟测试（32 kHz XT WDT / 40 MHz 漂移）与 GPIO 测试只能硬件：QEMU 无法建模晶振与物理 IO。WDT 是双启动序列（首次复位、二次校验复位原因）。

### 2. QEMU/GDB 环境约定（来自 `conftest.py`）

- 镜像：`build/<app_name>_qemu_image.bin`（带 MCUboot 头的原始二进制）
- 正常模式：`qemu-system-riscv32 -nographic -icount 3 -machine <target> -drive file=...,if=mtd,format=raw`
- 调试模式：加 `-s -S`（挂起等 GDB，开放 `:1234`）
- GDB：`riscv32-esp-elf-gdb build/<app>.elf --command=<script.gdb>`
- `-icount 3` = 指令计数模式，确保每次执行周期确定、可复现

pytest fixture（`conftest.py`）：

```python
@pytest.fixture
def qemu_instance(request, target):          # 正常模式
    ...
    qemu_process, output_queue = qemu.start()

@pytest.fixture
def qemu_debug_instance(request, target):    # 调试模式（-s -S，等 GDB）
    ...
    qemu_process, output_queue = qemu.start(debug=True)

@pytest.fixture
def gdb_instance(request):
    ...
    yield gdb
```

目标参数化（`tests/idf_targets.py`）：`("esp32c3", "esp32c5", "esp32c6", "esp32h2")`，与 CI 矩阵一致；可用 `--target=<soc>` 跑单变体。

### 3. 通用 GDB 故障注入模式（来自 `software_validation.rst`）

所有故障注入遵循同一脚本骨架：

```gdb
# 连到 QEMU 调试服务器
target remote :1234

# 在测试标签处下临时断点
tb <test_label>
continue

# 命中后改写寄存器/变量为错误值，再继续
commands
    set <reg_or_var>=<fault_value>
    continue
end
```

要点：
- 断点在测试**执行前**就下好
- 命中时**确定性地**改写目标
- 改完立即 `continue`，GDB 进程退出，留下 QEMU 跑被注入故障的测试
- pytest 捕获 QEMU stdout，在超时内断言输出

### 4. 端到端示例：CPU 寄存器故障注入（`tests/cpu_reg_test/pytest_qemu_cpu_reg_test.py`）

**正向测试（无 GDB，`test_cpu_reg_success`）：** QEMU 正常跑 `bist_cpu_regs_test()` + `bist_cpu_csr_regs_test()`，断言输出含：

```
test_BIST_Cpu_Regs:PASS
test_BIST_Cpu_Csr_Regs:PASS
```

**负向测试（参数化 32 个寄存器，`test_reg_error`）：** 对每个寄存器注入错误，期望 `FAIL`。pytest 把寄存器名注入 GDB 脚本模板：

```python
@pytest.mark.parametrize("reg_name", [
    "ra", "sp", "gp", "tp",
    "t0".."t6", "s0".."s11", "a0".."a7"
])
def test_reg_error(qemu_debug_instance, gdb_instance, reg_name, target):
    cpu_reg_error_test(qemu_debug_instance, gdb_instance, reg_name)
```

生成的 GDB 脚本（`reg_name=ra` 为例）：

```gdb
target remote :1234
tb testRegA_ra              # 库用 BIST_ADD_LABEL 打的标签
commands
    set $ra=0x55555555      # 测试期望 0xAAAAAAAA；注入相反模式
    continue
end
continue
```

机制：GDB 在标签 `testRegA_<reg>` 处把寄存器强改为 `0x55555555`，测试回读期望 `0xAAAAAAAA` → 不匹配 → 返回 `BIST_ESP_CPU_TEST_ERR` → 打印 `test_BIST_Cpu_Regs:FAIL`。pytest 断言输出含 `FAIL`，10 s 超时。

### 5. CSR 故障注入（同文件，`test_reg_csr_error`）

CSR 测试改的是临时寄存器 `$t0`（测试逻辑用它校验 CSR 内容），且按 SoC 跳过不适用的 CSR：

```python
PMA_ADDR_CSRS = ["0xBD0", ... "0xBDB"]      # 仅 C6/H2/C5
C5_ONLY_CSRS  = ["0x7E1", "0x7C5"]          # 仅 C5 (mexstatus/mhint)
PMA_TARGETS   = {"esp32c5", "esp32c6", "esp32h2"}

@pytest.mark.parametrize("csr_name", COMMON_CSRS + PMA_ADDR_CSRS + C5_ONLY_CSRS)
def test_reg_csr_error(qemu_debug_instance, gdb_instance, csr_name, target):
    if csr_name in PMA_ADDR_CSRS and target not in PMA_TARGETS:
        pytest.skip(...)                     # C3 无 PMA，跳过
    if csr_name in C5_ONLY_CSRS and target != "esp32c5":
        pytest.skip(...)                     # 非 C5 跳过 mexstatus/mhint
    script = f'''
    target remote :1234
    tb testRegA_{csr_name}
    commands
        set $t0=0x55555555
        continue
    end
    continue
    '''
```

覆盖：C3=25、C6/H2=37、C5=39 个 CSR，每个 CSR 一条 FAIL 用例。

### 6. RAM March 故障注入（`tests/ram_test/pytest_qemu_ram_test.py`）

关键细节：被测的局部指针 `start_addr` 在 `-Os` 下被优化进寄存器，GDB 访问不到，故改写的是**链接器符号** `_bist_ram_test_start`：

```gdb
target remote :1234
tb bist_ram_test_march_a_step2       # step2: Read 0, Write 1
commands
    set _bist_ram_test_start=0xFF     # 破坏测试区首字
    continue
end
continue
```

March X 用对称的 `bist_ram_test_march_x_step2`。两条用例分别期望 `test_BIST_ram_march_a:FAIL` / `test_BIST_ram_march_x:FAIL`。

### 7. PC 与 WDT 故障注入（带复位原因）

**PC 测试（`tests/pc_test/pytest_qemu_pc_test.py`）** 有两条负向用例，断点都在 `bist_verify_pc_test`（此刻 `$sp` 指向栈上的 `pcTestFunctions` 数组）：

| 用例 | 注入 | 期望 |
|---|---|---|
| `test_pc_error_no_wdt` | `set *(int*)$sp = *(int*)$sp ^ 4`（翻 PC bit 2） | `test_BIST_PC:FAIL`（跳进函数体偏移，返回值错，校验失败但不崩） |
| `test_pc_error_wdt` | `set *(int*)$sp = 0`（指针清零） | `Reset reason: 1` 与 `Reset reason: 7`（跳地址 0 → 取指故障 → WDT 复位） |

**WDT 测试（`tests/wdt_test/pytest_qemu_wdt_test.py`）** 在 `bist_test_wdt_timeout` 处改 `wdt_timeout_us`：

| 用例 | 注入 | 期望 |
|---|---|---|
| `test_wdt_success` | `set wdt_timeout_us=10000`（10 ms，测试内 50 ms 延时会超时 → 复位 → 二次启动校验复位原因） | `test_BIST_WDT:PASS` |
| `test_wdt_error` | `set wdt_timeout_us=1000000`（1 s，远大于 50 ms 延时 → 不复位） | `test_BIST_WDT:FAIL` |

### 8. 运行测试套件并产出 JUnit XML

```bash
# 单个测试目录（在 tests/cpu_reg_test/ 下，先 cmake -DSOC_TARGET=esp32c3 -B build -GNinja && ninja -C build）
pytest pytest_qemu_cpu_reg_test.py \
      --executable=cpu_reg_test --target=esp32c3 \
      --junitxml=build/tests/esp32c3_qemu_report.xml

# 全部 QEMU 套件
pytest pytest_qemu_* --executable=<app> --target=<soc> \
      --junitxml=build/tests/<soc>_qemu_report.xml

# 硬件套件
pytest pytest_device_* --junitxml=build/tests/<soc>_device_report.xml
```

`--executable` 是 `conftest.py` 的自定义选项（无扩展名的 elf 名）。`--target` 经 `indirect=True` 喂给 fixture 选 QEMU `-machine`。XML 落在 `build/tests/`，正是 `software_validation.rst` 里 `xml-junit-test-results` 指令消费的路径。

### 9. 串成 IEC 60730 证据链（4 份文档环环相扣）

| 文档 | 角色 | 关键产物 |
|---|---|---|
| `software_validation.rst` | 每个测试的 QEMU/硬件执行步骤、GDB 脚本、期望输出、CI JUnit 引用 | `build/tests/<soc>_{qemu,device}_report.xml` |
| `test_traceability_matrix.rst` | IEC 60730 组件 ID（1.1/1.3/3/4.1/4.2/6.3/7.1）→ 设计文件 → 测试脚本 → 结果 XML | 7 组件 / 10 模块 / 100+ 用例的四级追溯 |
| `coverage_analysis.rst` | 模块覆盖表（函数/测试用例数）+ 需求覆盖矩阵 + 已知限制（时钟/GPIO 仅硬件） | 100% 函数覆盖、100% 需求覆盖声明 |
| `tool_qualification.rst` | 工具分级 T1/T2/T3（IEC 61508-3 指导）+ GCC/ld/cppcheck/CMake/Ninja/QEMU/GDB/pytest 的鉴定方法与证据位置 | 工具链 pin 在 dev container、`-Os -ggdb -Wall -Wextra -Werror=all -std=gnu17` |
| `safety_case_summary.rst` | 6 条安全声明 × 证据来源 × 状态（全 Complete）+ 残余风险可接受性论证 | 顶层安全案例结论 |

证据链方向：`validation`（怎么测）→ `traceability`（测了哪些需求）→ `coverage`（覆盖够不够）→ `tool_qualification`（工具可信吗）→ `safety_case`（综合结论）。每条 IEC 60730 组件 ID 都能从需求一路追到 JUnit XML。

### 10. 关键覆盖数字（用于安全案例引用）

来自 `coverage_analysis.rst` 与 `test_traceability_matrix.rst`：

- **IEC 60730 组件**：7 个（1.1、1.3、3、4.1、4.2、6.3、7.1）
- **测试模块**：10 个
- **测试用例**：100+（CPU 寄存器 32×2、CSR 25–39×2、PC 4×2、RAM 2×2、Flash 2×2、栈 2、WDT 2、窗口 WDT 3、GPIO 3、时钟 2）
- **函数覆盖**：100% 公共 BIST API
- **SoC CSR 数**：C3=25、C6/H2=37、C5=39

## 常见错误

| 错误 | 原因 | 解决方法 |
|---|---|---|
| GDB 改 `$ra` 后测试仍 PASS | 断点标签拼错或没命中 | 标签由 `BIST_ADD_LABEL(testRegA_<reg>)` 生成；用 `tb testRegA_<reg>` 精确匹配，末尾 `continue` |
| RAM 注入无效（改局部变量） | `-Os` 把 `start_addr` 优化进寄存器 | 改链接器符号 `set _bist_ram_test_start=0xFF`（见 step 6） |
| CSR 测试在 C3 上跑 PMA 用例报错 | C3 无 PMA | pytest 已用 `PMA_TARGETS` 跳过；自定义脚本要照搬该 skip 逻辑 |
| `pytest` 找不到 fixture | 没用仓库根的 `conftest.py` | 从仓库根跑 pytest；`qemu_instance`/`gdb_instance` 定义在根 `conftest.py` |
| 漏传 `--executable` | fixture 取不到 elf 名 | 必须传 `--executable=<app名>`（无扩展名） |
| WDT 测试只跑一次就判失败 | WDT 是双启动序列 | 首次复位是预期行为；二次启动校验复位原因 == `RESET_REASON_CORE_MWDT0` 才 PASS |
| PC `test_pc_error_wdt` 找不到复位原因 | 期望 `Reset reason: 1` 和 `Reset reason: 7` 两条 | 两条都要断言（断点注入后 CPU 跳 0 → 取指故障 → WDT 复位） |
| QEMU 执行不确定 | 没加 `-icount 3` | 调试模式必须 `-s -S -icount 3`；`conftest.py` 已默认带 |
| 证据包缺 JUnit XML | 没生成 `--junitxml` | 每次跑测试都带 `--junitxml=build/tests/<soc>_<env>_report.xml`，`software_validation.rst` 按此路径引用 |

## 参考

- `docs/en/software_validation.rst`（双轨策略、每测试的 GDB 脚本、期望输出、CI XML 引用）
- `docs/en/test_traceability_matrix.rst`（组件 ID → 设计 → 测试 → 结果四级追溯）
- `docs/en/coverage_analysis.rst`（模块/需求覆盖表、已知限制）
- `docs/en/tool_qualification.rst`（T1/T2/T3 分级、工具链 flag、证据位置）
- `docs/en/safety_case_summary.rst`（6 条声明 × 证据 × 状态）
- `conftest.py`（`QEMU_RISCV`、`GDB_RISCV`、三个 fixture、`--executable` 选项）
- `tests/idf_targets.py`（4 目标参数化）
- `tests/cpu_reg_test/pytest_qemu_cpu_reg_test.py`、`tests/ram_test/pytest_qemu_ram_test.py`、`tests/pc_test/pytest_qemu_pc_test.py`、`tests/wdt_test/pytest_qemu_wdt_test.py`（真实 GDB 脚本模板）
- `samples/standalone/gdbinit`（OpenOCD `:3333` 调试范式；QEMU 用 `:1234`）
