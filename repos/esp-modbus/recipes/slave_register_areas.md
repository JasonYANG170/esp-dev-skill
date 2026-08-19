# 从站寄存器区域映射

> **适用摘要**: 用 `mbc_slave_set_descriptor` 把用户存储结构映射到 Modbus 的 Holding / Input / Coil / Discrete 区域。覆盖 `mb_register_area_descriptor_t` 字段、`HOLD_OFFSET`/`INPUT_OFFSET` 宏、分段区域、字节 vs 位的区别、访问权限。

> Version: selected repo/component version; align with the user project and dependency manifest.
> Evidence: `repos/esp-modbus/resources/`, source/examples in `repos/esp-modbus/`, and this recipe path `repos/esp-modbus/recipes/slave_register_areas.md`.
> Validation: draft metadata added from repository routing; verify APIs, Kconfig symbols, and component dependencies against the selected version.

## 触发意图

- "从站怎么映射寄存器"
- "mbc_slave_set_descriptor 用法"
- "Holding 寄存器区域配置"
- "线圈区域字节数"
- "Modbus 从站存储结构"

## 前置条件

| 条件 | 要求 |
|---|---|
| 参考示例 | `espressif-repos/esp-modbus/examples/serial/mb_serial_slave/main/serial_slave.c`、`examples/tcp/mb_tcp_slave/main/tcp_slave.c` |
| 参数存储 | `holding_reg_params_t` 等（来自 `mb_example_common/include/modbus_params.h`） |
| 已创建从站对象 | `mbc_slave_create_serial` / `mbc_slave_create_tcp` 已成功 |

## 分步说明

### 1. 区域描述符字段（来自 `esp_modbus_slave.h`）

```c
typedef struct {
    uint16_t          start_offset;   // 该区域在 Modbus 协议中的相对寄存器偏移（0 基）
    mb_param_type_t   type;           // MB_PARAM_HOLDING / INPUT / COIL / DISCRETE
    mb_param_access_t access;         // MB_ACCESS_RW / MB_ACCESS_RO / MB_ACCESS_WO
    void             *address;        // 指向用户存储区
    size_t            size;           // 存储区大小（字节！）
} mb_register_area_descriptor_t;

typedef enum { MB_ACCESS_RW = 0x0000, MB_ACCESS_RO = 0x0001, MB_ACCESS_WO = 0x0002 } mb_param_access_t;
```

### 2. 偏移宏（从站侧用 `>>1`）

```c
#define HOLD_OFFSET(field) ((uint16_t)(offsetof(holding_reg_params_t, field) >> 1))
#define INPUT_OFFSET(field)((uint16_t)(offsetof(input_reg_params_t,   field) >> 1))

#define MB_REG_DISCRETE_INPUT_START (0x0000)
#define MB_REG_COILS_START          (0x0000)
#define MB_REG_HOLDING_START_AREA0  HOLD_OFFSET(holding_data0)
#define MB_REG_HOLDING_START_AREA1  HOLD_OFFSET(holding_data4)
```

### 3. 注册多个 Holding 分段

来自 `serial_slave.c`：

```c
mb_register_area_descriptor_t reg_area = {0};

// AREA0: holding_data0..holding_data3（4 个 float）
reg_area.type = MB_PARAM_HOLDING;
reg_area.start_offset = HOLD_OFFSET(holding_data0);
reg_area.address = (void *)&holding_reg_params.holding_data0;
reg_area.size = (HOLD_OFFSET(holding_data4) - HOLD_OFFSET(holding_data0)) << 1; // 寄存器差 << 1 = 字节
reg_area.access = MB_ACCESS_RW;
ESP_ERROR_CHECK(mbc_slave_set_descriptor(mbc_slave_handle, reg_area));

// AREA1: holding_data4..holding_data7
reg_area.type = MB_PARAM_HOLDING;
reg_area.start_offset = HOLD_OFFSET(holding_data4);
reg_area.address = (void *)&holding_reg_params.holding_data4;
reg_area.size = sizeof(float) << 2;     // 16 字节
reg_area.access = MB_ACCESS_RW;
ESP_ERROR_CHECK(mbc_slave_set_descriptor(mbc_slave_handle, reg_area));
```

> 一个从站可以为同一 `mb_param_type_t` 注册多个不同 `start_offset` 的区域。

### 4. 注册 Input Registers

```c
reg_area.type = MB_PARAM_INPUT;
reg_area.start_offset = INPUT_OFFSET(input_data0);
reg_area.address = (void *)&input_reg_params.input_data0;
reg_area.size = sizeof(float) << 2;     // 字节
ESP_ERROR_CHECK(mbc_slave_set_descriptor(mbc_slave_handle, reg_area));
```

### 5. 注册 Coils 与 Discrete Inputs

线圈/离散量按位打包，每字节 8 位。`size` 仍是字节数：

```c
reg_area.type = MB_PARAM_COIL;
reg_area.start_offset = MB_REG_COILS_START;
reg_area.address = (void *)&coil_reg_params;            // coil_reg_params_t
reg_area.size = sizeof(coil_reg_params);                // 字节
reg_area.access = MB_ACCESS_RW;
ESP_ERROR_CHECK(mbc_slave_set_descriptor(mbc_slave_handle, reg_area));

reg_area.type = MB_PARAM_DISCRETE;
reg_area.start_offset = MB_REG_DISCRETE_INPUT_START;
reg_area.address = (void *)&discrete_reg_params;
reg_area.size = sizeof(discrete_reg_params);
ESP_ERROR_CHECK(mbc_slave_set_descriptor(mbc_slave_handle, reg_area));
```

### 6. 直接读写映射存储要加锁

```c
mbc_slave_lock(mbc_slave_handle);
holding_reg_params.holding_data0 += 1.2f;
mbc_slave_unlock(mbc_slave_handle);
```

跨从站对象共享同一存储时，用 FreeRTOS 临界段：

```c
static portMUX_TYPE g_spinlock = portMUX_INITIALIZER_UNLOCKED;
portENTER_CRITICAL(&g_spinlock);
holding_reg_params.holding_data2 = 123.0f;
portEXIT_CRITICAL(&g_spinlock);
```

## 常见错误

| 错误 | 原因 | 解决方法 |
|---|---|---|
| 主机读到异常 0x02 | 该区域未注册 | 用 `mbc_slave_set_descriptor` 注册对应 `mb_param_type_t` |
| 主机能读但写失败 | `access = MB_ACCESS_RO` | 改为 `MB_ACCESS_RW` 或 `MB_ACCESS_WO` |
| 区域间错位 / 数据串 | `size` 写成寄存器数 | `size` 单位是 **字节**；float×4 = 16 字节 |
| 同类型多区域覆盖 | 两个区域 `start_offset` 重叠 | 区域之间不重叠；用 `HOLD_OFFSET` 宏算偏移 |
| 线圈位解析错位 | 当成字节处理 | Coils/Discrete 在协议层是按位；存储结构按字节打包（8 位/字节） |
| 修改存储导致数据竞争 | 协议栈活动时裸写 | 用 `mbc_slave_lock`/`unlock` 或 `portENTER_CRITICAL` |

## 参考

- `espressif-repos/esp-modbus/examples/serial/mb_serial_slave/main/serial_slave.c`
- `espressif-repos/esp-modbus/examples/tcp/mb_tcp_slave/main/tcp_slave.c`
- `espressif-repos/esp-modbus/modbus/mb_controller/common/include/esp_modbus_slave.h`
- `espressif-repos/esp-modbus/docs/en/slave_api_overview.rst`
- `recipes/serial_slave.md`、`recipes/tcp_slave.md`
