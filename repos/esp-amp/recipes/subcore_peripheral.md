# subcore 外设驱动开发

> **适用摘要**: 在 subcore 上开发外设驱动。HP subcore 使用 ESP-IDF hal 组件的 `_ll.h` 低层驱动；LP subcore 使用 IDF ulp 组件已实现的 LP 外设驱动。强调避免双核并发访问同一外设。

> Evidence: `repos/esp-amp/resources/`, source/examples in `repos/esp-amp/`, and this recipe path `repos/esp-amp/recipes/subcore_peripheral.md`.
> Validation: draft metadata added from repository routing; verify APIs, Kconfig symbols, and component dependencies against the selected version.

## 触发意图

- "subcore 控制 GPIO/UART/I2C"
- "LP subcore 外设"
- "HP subcore ll driver"
- "subcore 中断处理"

## 前置条件

| 条件 | 要求 |
|---|---|
| 参考文档 | `espressif-repos/esp-amp/docs/peripheral.md` |
| 外设能力 | 见下表（来自官方文档） |

## 分步说明

### 1. 外设 × subcore 类型 能力矩阵（官方）

| 外设类型 | subcore 类型 | 能力 |
|---|---|---|
| HP | HP（P4） | `_ll.h` 可用；支持中断处理 |
| HP | LP（C5/C6） | `_ll.h` 可用；**暂不支持中断处理** |
| LP | HP（P4） | 不支持 |
| LP | LP（C5/C6） | ulp 组件可用；支持中断处理 |

### 2. HP subcore 使用 `_ll.h` 低层驱动

```c
/* HP subcore（ESP32-P4）：IDF driver 组件不能用于 bare-metal，
   改用 components/hal/<target>/include/hal/<peripheral>_ll.h */
#include "hal/uart_ll.h"     /* 例：UART 低层驱动，OS-agnostic */

void uart_send_ll(uart_dev_t *dev, const uint8_t *data, int len)
{
    for (int i = 0; i < len; i++) {
        /* ll 驱动 API 跨 SoC 一致，寄存器级操作 */
        uart_ll_write_txfifo(dev, &data[i], 1, NULL);
    }
}
```

### 3. LP subcore 使用 ulp 组件（LP 外设）

```c
/* LP subcore（C5/C6）：IDF ulp 组件已实现 LP 外设驱动，直接复用 */
/* 参考官方：ESP32-C6 / ESP32-C5 ULP LP Core 文档 */
#include "ulp_lp_core.h"
#include "ulp_lp_core_gpio.h"   /* LP GPIO */
#include "ulp_lp_core_uart.h"   /* LP UART */

void lp_gpio_toggle(void)
{
    ulp_lp_core_gpio_set_level(LP_GPIO_NUM, 1);
    /* LP 外设完全由 LP core 控制，含中断注册与处理，不受 HP core 影响 */
}
```

> LP 外设支持（C5/C6）：LP IO（均支持）、LP I2C（IDF v5.4+）、LP UART（均支持）、LP SPI（不支持）。

### 4. 关键原则：单次操作周期内同一外设只由一个核访问

```c
/* ❌ WRONG — maincore 调 driver_init() 复位外设寄存器，破坏 subcore 正在使用的外设 */
// maincore: uart_driver_install(UART_NUM_1, ...);   // 复位 UART1 寄存器
// subcore 同时在用 _ll.h 操作 UART1 → 非预期

/* ✅ CORRECT — 同一 HP 外设在 init→deinit 周期内只由一个核访问 */
// maincore 用 UART0 做控制台；subcore 用 UART1（互不冲突）
```

### 5. HP subcore 中断处理（仅 P4）

```c
/* HP subcore 支持 HP 外设中断；注册与处理与 IDF ll 驱动配合 */
/* LP subcore 上 HP 外设暂不支持中断（仅轮询） */
```

### 6. 共享状态建议用 ESP-AMP IPC

外设若需跨核同步状态（如 subcore 采集、maincore 消费），用 RPMsg/Event/Virtqueue 传递，而非直接共享外设寄存器。

## 常见错误

| 错误 | 原因 | 解决方法 |
|---|---|---|
| LP subcore 上 HP 外设中断不触发 | 当前不支持 | LP subcore 上 HP 外设仅轮询；中断需 HP subcore（P4） |
| subcore 调 `uart_driver_install` 失败 | IDF driver 组件假设 FreeRTOS | subcore 用 `_ll.h` 或 ulp 组件 |
| 双核同时操作同一外设异常 | `driver_init()` 复位寄存器 | 同一外设单次周期内只由一个核访问 |
| LP SPI 不可用 | 硬件不支持 | LP subcore 无 LP SPI；改用 LP I2C 或 HP SPI |
| maincore deinit 后 subcore 仍访问 | 未协调生命周期 | 协调 init/deinit 时机，避免交叉 |

## 参考

- `espressif-repos/esp-amp/docs/peripheral.md`
- ESP-IDF ULP LP Core 文档（C6/C5）
- ESP-IDF hal 组件 `components/hal/<target>/include/hal/<peripheral>_ll.h`
