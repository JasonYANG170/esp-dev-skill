# Modbus 串行从站（RTU / ASCII）

> **适用摘要**: 在 ESP32 系列芯片上构建 Modbus 串行从站，覆盖 UART/RS485 初始化、寄存器区域描述符（Holding/Input/Coil/Discrete）、从站事件循环与销毁。适用于 RTU 与 ASCII 两种模式。

> Version: selected repo/component version; align with the user project and dependency manifest.
> Evidence: `repos/esp-modbus/resources/`, source/examples in `repos/esp-modbus/`, and this recipe path `repos/esp-modbus/recipes/serial_slave.md`.
> Validation: draft metadata added from repository routing; verify APIs, Kconfig symbols, and component dependencies against the selected version.

## 触发意图

- "做一个 Modbus 串行从机"
- "Modbus RTU slave"
- "Modbus ASCII 从站"
- "RS485 从机配置"
- "从站寄存器映射"

## 前置条件

| 条件 | 要求 |
|---|---|
| 参考示例 | `espressif-repos/esp-modbus/examples/serial/mb_serial_slave/` |
| 共享参数组件 | `espressif-repos/esp-modbus/examples/mb_example_common/`（`holding_reg_params_t` 等） |
| Kconfig | `CONFIG_FMB_COMM_MODE_RTU_EN=y`（或 `CONFIG_FMB_COMM_MODE_ASCII_EN=y`）；`CONFIG_MB_SLAVE_ADDR=1`；`CONFIG_MB_UART_*` |
| 硬件 | RS485 收发器，TX/RX/RTS 引脚按 `CONFIG_MB_UART_TXD/RXD/RTS` 接线 |

## 分步说明

### 1. 包含头文件与参数结构

```c
#include "mbcontroller.h"        // Modbus 控制器 API
#include "modbus_params.h"       // holding_reg_params_t 等结构 + extern 实例
#include "esp_err.h"
#include "esp_log.h"
#include "driver/uart.h"
#include "sdkconfig.h"
```

### 2. 构造 `mb_communication_info_t` 并创建从站对象

来自 `examples/serial/mb_serial_slave/main/serial_slave.c`：

```c
#define MB_PORT_NUM     (CONFIG_MB_UART_PORT_NUM)
#define MB_SLAVE_ADDR   (CONFIG_MB_SLAVE_ADDR)
#define MB_DEV_SPEED    (CONFIG_MB_UART_BAUD_RATE)

static void *mbc_slave_handle = NULL;

mb_communication_info_t comm_config = {
    .ser_opts.port = MB_PORT_NUM,
#if CONFIG_MB_COMM_MODE_ASCII
    .ser_opts.mode = MB_ASCII,
#elif CONFIG_MB_COMM_MODE_RTU
    .ser_opts.mode = MB_RTU,
#endif
    .ser_opts.baudrate = MB_DEV_SPEED,
    .ser_opts.parity = MB_PARITY_NONE,        // = UART_PARITY_DISABLE
    .ser_opts.uid = MB_SLAVE_ADDR,            // 从站 Unit Identifier
    .ser_opts.data_bits = UART_DATA_8_BITS,
    .ser_opts.stop_bits = UART_STOP_BITS_1
};
ESP_ERROR_CHECK(mbc_slave_create_serial(&comm_config, &mbc_slave_handle));
```

### 3. 注册寄存器区域描述符

每个被主机访问的区域都必须用 `mbc_slave_set_descriptor` 注册，否则协议栈返回 Modbus 异常。`size` 字段单位是 **字节**。

```c
// 偏移宏（从站示例用法：>>1 得到寄存器相对偏移）
#define HOLD_OFFSET(field) ((uint16_t)(offsetof(holding_reg_params_t, field) >> 1))
#define INPUT_OFFSET(field)((uint16_t)(offsetof(input_reg_params_t,   field) >> 1))

mb_register_area_descriptor_t reg_area = {0};

// Holding Registers（两个分段，对应示例的 AREA0 / AREA1）
reg_area.type = MB_PARAM_HOLDING;
reg_area.start_offset = HOLD_OFFSET(holding_data0);
reg_area.address = (void *)&holding_reg_params.holding_data0;
reg_area.size = (HOLD_OFFSET(holding_data4) - HOLD_OFFSET(holding_data0)) << 1; // 字节
reg_area.access = MB_ACCESS_RW;
ESP_ERROR_CHECK(mbc_slave_set_descriptor(mbc_slave_handle, reg_area));

reg_area.type = MB_PARAM_HOLDING;
reg_area.start_offset = HOLD_OFFSET(holding_data4);
reg_area.address = (void *)&holding_reg_params.holding_data4;
reg_area.size = sizeof(float) << 2;     // 4 个 float = 16 字节
reg_area.access = MB_ACCESS_RW;
ESP_ERROR_CHECK(mbc_slave_set_descriptor(mbc_slave_handle, reg_area));

// Input Registers
reg_area.type = MB_PARAM_INPUT;
reg_area.start_offset = INPUT_OFFSET(input_data0);
reg_area.address = (void *)&input_reg_params.input_data0;
reg_area.size = sizeof(float) << 2;
ESP_ERROR_CHECK(mbc_slave_set_descriptor(mbc_slave_handle, reg_area));

// Coils（1 字节对应 8 个线圈）
reg_area.type = MB_PARAM_COIL;
reg_area.start_offset = 0x0000;
reg_area.address = (void *)&coil_reg_params;
reg_area.size = sizeof(coil_reg_params);   // 字节
reg_area.access = MB_ACCESS_RW;
ESP_ERROR_CHECK(mbc_slave_set_descriptor(mbc_slave_handle, reg_area));

// Discrete Inputs
reg_area.type = MB_PARAM_DISCRETE;
reg_area.start_offset = 0x0000;
reg_area.address = (void *)&discrete_reg_params;
reg_area.size = sizeof(discrete_reg_params);
ESP_ERROR_CHECK(mbc_slave_set_descriptor(mbc_slave_handle, reg_area));
```

### 4. 设置 UART 引脚与 RS485 半双工模式（应用层职责）

```c
ESP_ERROR_CHECK(uart_set_pin(MB_PORT_NUM, CONFIG_MB_UART_TXD,
                             CONFIG_MB_UART_RXD, CONFIG_MB_UART_RTS,
                             UART_PIN_NO_CHANGE));
ESP_ERROR_CHECK(uart_set_mode(MB_PORT_NUM, UART_MODE_RS485_HALF_DUPLEX));
```

### 5. 启动协议栈并初始化寄存器值

```c
setup_reg_data();   // 用户函数：把 holding_reg_params 等填成已知值
esp_err_t err = mbc_slave_start(mbc_slave_handle);
```

### 6. 事件循环：阻塞等待主机访问

```c
#define MB_READ_MASK  (MB_EVENT_INPUT_REG_RD | MB_EVENT_HOLDING_REG_RD \
                       | MB_EVENT_DISCRETE_RD | MB_EVENT_COILS_RD)
#define MB_WRITE_MASK (MB_EVENT_HOLDING_REG_WR | MB_EVENT_COILS_WR)
#define MB_READ_WRITE_MASK (MB_READ_MASK | MB_WRITE_MASK)
#define MB_PAR_INFO_GET_TOUT  (10)

mb_param_info_t reg_info;

for (; holding_reg_params.holding_data0 < 6;) {
    (void)mbc_slave_check_event(mbc_slave_handle, MB_READ_WRITE_MASK);
    ESP_ERROR_CHECK(mbc_slave_get_param_info(mbc_slave_handle, &reg_info, MB_PAR_INFO_GET_TOUT));

    if (reg_info.type & (MB_EVENT_HOLDING_REG_WR | MB_EVENT_HOLDING_REG_RD)) {
        // 在修改映射存储时必须加锁
        if (reg_info.address == (uint8_t *)&holding_reg_params.holding_data0) {
            (void)mbc_slave_lock(mbc_slave_handle);
            holding_reg_params.holding_data0 += 1.2f;
            (void)mbc_slave_unlock(mbc_slave_handle);
        }
    }
}
```

### 7. 销毁

```c
vTaskDelay(100);
ESP_ERROR_CHECK(mbc_slave_delete(mbc_slave_handle));
```

## 常见错误

| 错误 | 原因 | 解决方法 |
|---|---|---|
| 主机读到 Modbus 异常 0x02（Illegal Data Address） | 该寄存器区域未用 `mbc_slave_set_descriptor` 注册 | 为每个被访问的 `MB_PARAM_*` 类型注册至少一个区域描述符 |
| 串口收发乱码 / 无响应 | 未调用 `uart_set_mode(UART_MODE_RS485_HALF_DUPLEX)` 或 RTS 引脚接错 | 应用层必须调用 `uart_set_pin` + `uart_set_mode`；核对 RS485 收发器 DE/RE 与 RTS 的连接 |
| `holding_data0` 累加错乱 | 直接修改映射存储未加锁 | 用 `mbc_slave_lock` / `mbc_slave_unlock` 包裹修改 |
| 主机能读但写不进 | 区域 `access` 设成了 `MB_ACCESS_RO` | 改为 `MB_ACCESS_RW` |
| `mbc_slave_create_serial` 返回 `ESP_ERR_NOT_SUPPORTED` | 对应通信模式 Kconfig 未启用 | 启用 `CONFIG_FMB_COMM_MODE_RTU_EN` 或 `CONFIG_FMB_COMM_MODE_ASCII_EN` |

## 参考

- `espressif-repos/esp-modbus/examples/serial/mb_serial_slave/main/serial_slave.c`
- `espressif-repos/esp-modbus/examples/serial/mb_serial_slave/main/Kconfig.projbuild`
- `espressif-repos/esp-modbus/docs/en/slave_api_overview.rst`
- `espressif-repos/esp-modbus/docs/en/port_initialization.rst`
- `resources/api_reference.md`、`resources/config_reference.md`
