# LP Core 调试（panic 定位与日志）

> **适用摘要**: 识别 LP Core 的 Breakpoint panic（空指针）与 Illegal Instruction panic（缓冲/栈溢出），用 `addr2line` + MEPC 定位代码行，依靠 `printf` 日志与编码实践排查。

## 触发意图

- "lowcode panic"
- "Guru Meditation Error Subcore"
- "addr2line 定位"
- "LP core 崩溃"
- "栈溢出 illegal instruction"

## 前置条件

| 条件 | 要求 |
|---|---|
| 产物 | 已构建的 `<product>.elf`（在产品 build 目录） |
| 工具 | `riscv32-esp-elf-addr2line`（ESP-IDF 工具链自带） |
| 参考文档 | `docs/debugging.md`、`docs/programmer_model.md` |

## 分步说明

### 1. Breakpoint Panic（空指针）

典型输出（取自 `docs/debugging.md`）：

```
Guru Meditation Error: Subcore panic'ed Breakpoint
Core 1 register dump:
MEPC    : 0x4086e922 RA      : 0x4086e91e SP      : 0x50003870 ...
MSTATUS : 0x00001800 MTVEC   : 0x50000001 MCAUSE  : 0x00000003 MTVAL   : 0x00000000
```

定位命令（在产品工作目录）：

```sh
riscv32-esp-elf-addr2line -e <path-to-elf-file> <mepc-address>
# 例：riscv32-esp-elf-addr2line -e build/light_cw_pwm.elf 0x4086e922
```

`.elf` 在产品的 build 目录中。注意：并非所有空指针访问都会触发 Breakpoint panic——编译器仅在**编译期可检测**时插入断点；运行时空指针可能不 panic。

### 2. Illegal Instruction Panic（缓冲/栈溢出）

典型特征：寄存器全 0，`MCAUSE` 为 Unhandled interrupt/Unknown cause：

```
Guru Meditation Error: Subcore panic'ed Unhandled interrupt/Unknown cause
Core 1 register dump:
MEPC : 0x00000000 ... MCAUSE : 0x00000000
```

此时栈可能已被破坏，**栈转储通常不可靠**。应转而依靠日志（见下）。

### 3. 依靠日志（LP Core 的 printf）

- LP Core 自带 `printf` 实现，输出到 console
- 单线程同步执行，日志顺序与执行顺序一致，便于追踪逻辑
- 在可疑函数前后加 `printf("%s: ...\n", TAG, ...)` 锁定最后执行点

```cpp
static const char *TAG = "app_driver";
printf("%s: before read, i2c=%d\n", TAG, I2C_PORT);
/* ...可疑操作... */
printf("%s: after read, temp=%d\n", TAG, (int)(temp*100));
```

### 4. 推荐编码实践（取自 `docs/debugging.md`）

- **显式定义并严守缓冲边界**：数组/缓冲绝不越界
- **不要用 malloc/calloc**：用静态缓冲，避免内存碎片与未定义行为
- **限制指针运算**：避免误访非法内存
- **避免深递归**：改迭代，恒定栈占用

> 注意：LP 固件非法内存访问不一定 panic；读非法内存**总是返回 0x0**，写可能静默——因此日志比依赖崩溃更重要。

## 常见错误

| 错误 | 原因 | 解决方法 |
|---|---|---|
| Breakpoint panic | 空指针访问 | addr2line + MEPC 定位，加判空 |
| Illegal Instruction | 缓冲/栈溢出 | 检查数组边界，去掉递归/malloc |
| 读数恒为 0 | 读了非法内存（返回 0） | 检查指针/句柄是否已初始化 |
| 栈转储无意义 | SP 被破坏 | 改用日志逐步定位 |
| 找不到 .elf | 路径不对 | 在产品 `build/` 目录下 `<product>.elf` |

## 参考

- `docs/debugging.md`、`docs/programmer_model.md`
- `recipes/getting_started.md`（console 命令）
