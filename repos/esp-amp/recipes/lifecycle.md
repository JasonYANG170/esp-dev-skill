# subcore 生命周期管理

> **适用摘要**: 在 maincore 管理 subcore 的完整生命周期：加载固件、启动、停止、检测 panic、自定义 panic 处理、路由 subcore printf 到 maincore 控制台。

> Version: selected repo/component version; align with the user project and dependency manifest.
> Evidence: `repos/esp-amp/resources/`, source/examples in `repos/esp-amp/`, and this recipe path `repos/esp-amp/recipes/lifecycle.md`.
> Validation: draft metadata added from repository routing; verify APIs, Kconfig symbols, and component dependencies against the selected version.

## 触发意图

- "启动/停止 subcore"
- "subcore panic 处理"
- "路由 subcore printf 到 maincore"
- "重新加载 subcore 固件"
- "subcore 崩溃恢复"

## 前置条件

| 条件 | 要求 |
|---|---|
| 参考工程 | 所有 `examples/*/maincore/main/app_main.c`（含加载启动模式） |
| Kconfig | 路由 printf / panic 处理需 `CONFIG_ESP_AMP_SYSTEM_ENABLE_SUPPLICANT=y`；路由打印还需 `CONFIG_ESP_AMP_ROUTE_SUBCORE_PRINT=y` |

## 分步说明

### 1. 加载并启动 subcore

```c
#include "esp_amp.h"
#include "esp_partition.h"

/* 方式 A：从 flash 分区加载 */
const esp_partition_t *sub_partition = esp_partition_find_first(ESP_PARTITION_TYPE_DATA, 0x40, NULL);
ESP_ERROR_CHECK(esp_amp_load_sub_from_partition(sub_partition));

/* 方式 B：从嵌入的二进制加载（仅 unified build + EMBED） */
extern const uint8_t subcore_my_app_bin_start[] asm("_binary_subcore_my_app_bin_start");
ESP_ERROR_CHECK(esp_amp_load_sub(subcore_my_app_bin_start));

/* 启动 subcore（返回 0 成功，-1 失败） */
ESP_ERROR_CHECK(esp_amp_start_subcore() == 0 ? ESP_OK : ESP_FAIL);
```

### 2. 停止 subcore

```c
esp_amp_stop_subcore();
```

### 3. 检测 subcore panic

```c
if (esp_amp_subcore_panic()) {
    printf("subcore has panicked!\n");
}
```
> panic 时 subcore 把栈与寄存器转储到专用内存区并触发对 maincore 的软件中断；maincore 停止 subcore 后由 supplicant 在任务上下文调用 panic handler。

### 4. 覆盖默认 panic handler（弱函数）

```c
/* 在 maincore 工程中定义同名函数即可覆盖弱符号 */
void esp_amp_subcore_panic_handler(void)
{
    printf("custom: subcore panic detected\n");
    esp_amp_subcore_panic_handler_default();   // 仍可调用默认实现打印信息
    // 例：重新加载并重启 subcore，或复位整个系统
}
```

### 5. 路由 subcore printf 到 maincore 控制台

无需应用代码——subcore 调用 `printf()` 即自动路由。需在 maincore sdkconfig 开启：
```
CONFIG_ESP_AMP_SYSTEM_ENABLE_SUPPLICANT=y    # 创建 maincore daemon 任务（+2KB flash / +2.5KB heap / +2KB 共享内存）
CONFIG_ESP_AMP_ROUTE_SUBCORE_PRINT=y         # 路由打印到 maincore 控制台
```
> 未开启路由时：LP subcore 打印到 LP UART，HP subcore 打印到 UART1，需要额外硬件查看。virtqueue 未初始化时（panic 早期）使用 `esp_amp_early_printf()` 直接写 maincore UART tx fifo。

### 6. 完整典型流程（maincore app_main）

```c
void app_main(void)
{
    assert(esp_amp_init() == 0);

    const esp_partition_t *p = esp_partition_find_first(ESP_PARTITION_TYPE_DATA, 0x40, NULL);
    ESP_ERROR_CHECK(esp_amp_load_sub_from_partition(p));
    ESP_ERROR_CHECK(esp_amp_start_subcore());

    /* 握手 */
    assert((esp_amp_event_wait(EVENT_SUBCORE_READY, true, true, 10000)
            & EVENT_SUBCORE_READY) == EVENT_SUBCORE_READY);

    /* 业务 ... */
}
```

## 常见错误

| 错误 | 原因 | 解决方法 |
|---|---|---|
| `esp_amp_start_subcore()` 返回 -1 | subcore 固件未加载 / 加载失败 | 先调用 `esp_amp_load_sub()` 或 `esp_amp_load_sub_from_partition()` |
| subcore panic 无输出 | 未启用 supplicant | 设 `CONFIG_ESP_AMP_SYSTEM_ENABLE_SUPPLICANT=y`；panic 经软件中断到 maincore |
| subcore printf 与 maincore 交错 | 未启用路由，双方直写同一 UART | 启用 `CONFIG_ESP_AMP_ROUTE_SUBCORE_PRINT=y`（经 supplicant 任务串行输出） |
| panic handler 内调用阻塞 API 失败 | handler 在任务上下文但仍应避免阻塞 | handler 由 supplicant 任务调用，避免耗时/阻塞操作 |
| 自定义 handler 不生效 | 函数签名不匹配 | 必须为 `void esp_amp_subcore_panic_handler(void)`（覆盖弱符号） |

## 参考

- `examples/rpmsg_send_recv/maincore/main/app_main.c` — 加载/启动/握手完整流程
- `espressif-repos/esp-amp/docs/system.md`
- `espressif-repos/esp-amp/components/esp_amp/system/include/esp_amp_system.h`
