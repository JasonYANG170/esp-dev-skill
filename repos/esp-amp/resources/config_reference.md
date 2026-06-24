# ESP-AMP 配置项速查（Kconfig）

> 所有符号来自 `espressif-repos/esp-amp/components/esp_amp/Kconfig`。默认值与取值范围因目标芯片不同（已标注）。

## 总开关与 subcore 类型

| 符号 | 类型 | 默认 | 说明 |
|---|---|---|---|
| `CONFIG_ESP_AMP_ENABLED` | bool | n | 启用 ESP-AMP。依赖 `IDF_TARGET_ESP32C6 / C5 / P4` |
| `CONFIG_ESP_AMP_SUBCORE_TYPE_LP_CORE` | bool | C5/C6 自动 | LP core 作为 subcore。依赖 `SOC_LP_CORE_SUPPORTED` |
| `CONFIG_ESP_AMP_SUBCORE_TYPE_HP_CORE` | bool | P4 自动 | HP core 作为 subcore。自动 `select FREERTOS_UNICORE` |

> subcore 类型由目标芯片自动选择，无需手动改。

## 内存管理（ESP-AMP Memory Management）

| 符号 | 类型 | 默认（C5/C6） | 默认（P4） | 范围（C5/C6） | 范围（P4） | 说明 |
|---|---|---|---|---|---|---|
| `CONFIG_ESP_AMP_HP_SHARED_MEM_SIZE` | int | 16384 | 16384 | 1024~20480 | 1024~65536 | HP RAM 共享内存大小。内部组件（virtqueue/event）从中分配，必须够大 |
| `CONFIG_ESP_AMP_RTC_SHARED_MEM_SIZE` | int | 256 | — | 2~1024 | — | RTC RAM 共享内存（仅 LP subcore）。virtqueue 标志位/sw_intr 位从中分配 |
| `CONFIG_ESP_AMP_SUBCORE_USE_HP_MEM_SIZE` | int | 16384 | 65536 | 16384~49152 | 65536~196608 | 加载 subcore 固件的最大 HP RAM（仅非 PURE_RTC_RAM_APP） |
| `CONFIG_ESP_AMP_SUBCORE_STACK_SIZE_MIN` | int | 1024 | 16384 | 256~32768 | 256~32768 | subcore 最小栈。C5/C6 栈在 RTC RAM，P4 在 HP RAM。不足则 build 失败 |
| `CONFIG_ESP_AMP_SUBCORE_ENABLE_HEAP` | bool | n | n | — | — | 启用 subcore 动态内存。不启用时 malloc 返回 -1 |
| `CONFIG_ESP_AMP_SUBCORE_HEAP_SIZE` | int | 见下 | 16384 | 0~16384 | 0~65536 | subcore heap 大小。PURE_RTC_RAM_APP 且 C5/C6 默认 512，否则 4096 |

## subcore 工具链设置（ESP-AMP Subcore Toolchain Settings）

| 符号 | 类型 | 默认 | 依赖 | 说明 |
|---|---|---|---|---|
| `CONFIG_ESP_AMP_SUBCORE_ENABLE_HW_FPU` | bool | y | 仅 P4 | 启用硬件 FPU。关闭则浮点用软件实现（慢） |
| `CONFIG_ESP_AMP_SUBCORE_ENABLE_HW_FPU_IN_ISR` | bool | n | `ENABLE_HW_FPU` | 允许 ISR 内用硬件 FPU（默认禁用，触发非法指令异常）。引入 ISR 上下文切换开销 |
| `CONFIG_ESP_AMP_SUBCORE_USE_ARITH64` | bool | n | — | 用 arith64 替代 libgcc 做 64 位整型运算（省约 1KB，慢 2~3 倍） |
| `CONFIG_ESP_AMP_SUBCORE_BUILD_TYPE_PURE_RTC_RAM_APP` | bool | n | 仅 LP subcore | 整个 subcore 应用放入 RTC RAM。LTO 仅在此模式下安全 |
| `CONFIG_ESP_AMP_SUBCORE_LTO_ENABLE` | bool | n | `PURE_RTC_RAM_APP` | 启用 LTO。注意：`idf.py size` 不兼容，需 `python3 -m esp_idf_size.ng --lto` |

> LP core 无硬件 FPU，浮点全软件实现，建议 LP subcore 仅用整数运算。HP/LP subcore 均不支持浮点数打印。

## 事件与软件中断表大小

| 符号 | 类型 | 默认 | 范围 | 说明 |
|---|---|---|---|---|
| `CONFIG_ESP_AMP_EVENT_TABLE_LEN` | int | 8 | 4~64 | ESP-AMP event 可绑定的 OS event handle 数（不含保留项） |
| `CONFIG_ESP_AMP_SW_INTR_HANDLER_TABLE_LEN` | int | 8 | 4~64 | 软件中断 handler 表大小。越大 ISR 延迟越大 |

## System 组件（ESP-AMP System）

| 符号 | 类型 | 默认 | 说明 |
|---|---|---|---|
| `CONFIG_ESP_AMP_SYSTEM_ENABLE_SUPPLICANT` | bool | n | 创建 maincore daemon 任务处理 subcore panic / printf 路由。+2KB flash / +2.5KB heap / +2KB 共享内存 |
| `CONFIG_ESP_AMP_ROUTE_SUBCORE_PRINT` | bool | n | 路由 subcore printf 到 maincore 控制台。依赖 supplicant；LP subcore 会 select `ULP_HP_UART_CONSOLE_PRINT` |
| `CONFIG_ESP_AMP_SYSTEM_AUTO_LIGHT_SLEEP_SUPPORT_ENABLE` | bool | n | 启用自动 light sleep 支持。依赖 LP subcore + `PM_ENABLE` |

## subcore 构建相关断言级别（IDF 通用，subcore 适用）

| 符号 | 用途 |
|---|---|
| `CONFIG_COMPILER_OPTIMIZATION_ASSERTIONS_ENABLE` | 开发期：打印断言内容与行号（体积大） |
| `CONFIG_COMPILER_OPTIMIZATION_ASSERTIONS_SILENT` | 生产期：仅打印 abort 地址（推荐） |
| `CONFIG_COMPILER_OPTIMIZATION_ASSERTIONS_DISABLE` | 极限省体积：禁用断言（可能未定义行为） |

## unified build 自定义选项（examples 中常见）

| 符号 | 来源 | 说明 |
|---|---|---|
| `CONFIG_SUBCORE_FIRMWARE_EMBEDDED` | example Kconfig | 切换 subcore 固件嵌入 / 分区存储 |
| `CONFIG_EXAMPLE_RPMSG_ENABLE_INTERRUPT_ON_MAINCORE` | example Kconfig | maincore RPMsg 中断/轮询模式 |
| `CONFIG_EXAMPLE_RPMSG_ENABLE_INTERRUPT_ON_SUBCORE` | example Kconfig | subcore RPMsg 中断/轮询模式 |

## 内存放置属性宏（light sleep / RTC RAM）

| 宏 | 作用 |
|---|---|
| `RTC_DATA_ATTR` | 数据放 RTC RAM |
| `RTC_RODATA_ATTR` | 常量放 RTC RAM |
| `RTC_IRAM_ATTR` | 函数放 RTC RAM（light sleep 时 ISR 必须用此而非 `IRAM_ATTR`） |
| `IRAM_ATTR` | 函数放 HP RAM 的 IRAM（maincore ISR 用；subcore light sleep 时不可用） |
| `DRAM_ATTR` | 数据放 DRAM（maincore ISR 中 TAG 字符串用） |

## 关键约定

- **separate build 下 maincore 与 subcore 的 ESP-AMP 相关项必须手动一致**（共享内存大小、SysInfo/Event 设置等）。unified build 下共享 sdkconfig 自动一致。
- subcore **不支持改名 main 组件**，必须保持名为 `main`。
- subcore component 在 unified build 下加 `sub_` 前缀避免与 maincore 同名组件冲突，并用 `if(NOT SUBCORE_BUILD) idf_component_register() return() endif()` 守卫。
- ESP32-P4 选 HP subcore 时自动 `select FREERTOS_UNICORE`。
