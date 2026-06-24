# 主站数据字典（Data Dictionary）

> **适用摘要**: 为 Modbus 主站编写 `mb_parameter_descriptor_t` 数据字典表，把物理量（CID）映射到从站的 Modbus 寄存器。覆盖 CID 枚举、`STR`/`OPTS`/`HOLD_OFFSET` 宏、寄存器类型与数据类型的搭配、`param_offset` 的含义、自定义命令权限。

## 触发意图

- "怎么定义 Modbus 数据字典"
- "CID 映射寄存器"
- "mb_parameter_descriptor_t 怎么填"
- "Modbus 主站参数表"
- "PARAM_TYPE_FLOAT 怎么配"

## 前置条件

| 条件 | 要求 |
|---|---|
| 参考示例 | `espressif-repos/esp-modbus/examples/serial/mb_serial_master/main/serial_master.c`、`examples/tcp/mb_tcp_master/main/tcp_master.c` |
| 参数存储 | `holding_reg_params_t` 等结构（来自 `mb_example_common/include/modbus_params.h`） |
| Kconfig | 用扩展类型（`PARAM_TYPE_*_ABCD` 等）需 `CONFIG_FMB_EXT_TYPE_SUPPORT=y` |

## 分步说明

### 1. 结构字段总览（来自 `esp_modbus_master.h`）

```c
typedef struct {
    uint16_t            cid;            // 特征 ID（必须唯一）
    const char         *param_key;      // 名称字符串（必须唯一）
    const char         *param_units;    // 物理单位
    uint8_t             mb_slave_addr;  // 从站短地址（UID）
    mb_param_type_t     mb_param_type;  // MB_PARAM_HOLDING / INPUT / COIL / DISCRETE
    uint16_t            mb_reg_start;   // Modbus 寄存器起始（0 基）
    uint16_t            mb_size;        // 长度（Holding/Input=寄存器数；Coil/Discrete=位数）
    uint32_t            param_offset;   // 在存储结构中的实例偏移（字节）
    mb_descr_type_t     param_type;     // PARAM_TYPE_FLOAT / U16 / ASCII / *_ABCD ...
    size_t              param_size;     // 实例存储大小（字节）
    mb_parameter_opt_t  param_opts;     // 选项：min/max/step 或 cust_cmd_read/write/other
    mb_param_perms_t    access;         // PAR_PERMS_READ_WRITE_TRIGGER 等
} mb_parameter_descriptor_t;
```

### 2. 辅助宏（直接抄自主站示例）

```c
#define STR(fieldname) ((const char *)(fieldname))
#define OPTS(min_val, max_val, step_val) { .opt1 = min_val, .opt2 = max_val, .opt3 = step_val }

// 主站侧用 +1（示例约定，用于区分“未设置”）
#define HOLD_OFFSET(field) ((uint16_t)(offsetof(holding_reg_params_t, field) + 1))
#define INPUT_OFFSET(field)((uint16_t)(offsetof(input_reg_params_t,   field) + 1))
#define COIL_OFFSET(field) ((uint16_t)(offsetof(coil_reg_params_t,    field) + 1))
#define DISCR_OFFSET(field)((uint16_t)(offsetof(discrete_reg_params_t,field) + 1))

#define TEST_HOLD_REG_START(field) (HOLD_OFFSET(field) >> 1)
#define TEST_HOLD_REG_SIZE(field)  (sizeof(((holding_reg_params_t *)0)->field) >> 1)
#define TEST_INPUT_REG_START(field) (INPUT_OFFSET(field) >> 1)
#define TEST_INPUT_REG_SIZE(field)  (sizeof(((input_reg_params_t *)0)->field) >> 1)
```

### 3. 一个完整的最小字典

```c
enum { MB_DEVICE_ADDR1 = 1 };
enum { CID_TEMP = 0, CID_HUMID, CID_RELAY, CID_COUNT };

const mb_parameter_descriptor_t device_parameters[] = {
    // float 温度，Input Register，2 个寄存器 = 4 字节
    { CID_TEMP, STR("Temperature"), STR("C"), MB_DEVICE_ADDR1, MB_PARAM_INPUT,
      TEST_INPUT_REG_START(input_data0), TEST_INPUT_REG_SIZE(input_data0),
      INPUT_OFFSET(input_data0), PARAM_TYPE_FLOAT, 4,
      OPTS(0, 100, 0), PAR_PERMS_READ_WRITE_TRIGGER },

    // float 湿度，Holding Register
    { CID_HUMID, STR("Humidity"), STR("%rH"), MB_DEVICE_ADDR1, MB_PARAM_HOLDING,
      TEST_HOLD_REG_START(holding_data0), TEST_HOLD_REG_SIZE(holding_data0),
      HOLD_OFFSET(holding_data0), PARAM_TYPE_FLOAT, 4,
      OPTS(-40, 50, 0), PAR_PERMS_READ_WRITE_TRIGGER },

    // 1 字节线圈状态，Coil，6 位
    { CID_RELAY, STR("RelayP1"), STR("on/off"), MB_DEVICE_ADDR1, MB_PARAM_COIL, 2, 6,
      COIL_OFFSET(coils_port0), PARAM_TYPE_U8, 1,
      OPTS(0xAA, 0x15, 0), PAR_PERMS_READ_WRITE_TRIGGER },
};
const uint16_t num_device_parameters =
    (sizeof(device_parameters) / sizeof(device_parameters[0]));
```

### 4. 自定义命令权限（`PAR_PERMS_READ_WRITE_CUST_CMD`）

当 `access` 含 `PAR_PERMS_CUST_CMD` 位时，`param_opts` 的前两个 int 被解释为读/写功能码（如 0x03 / 0x06），覆盖默认命令：

```c
{ CID_HOLD_CUSTOM1, STR("CustomHoldReg"), STR("__"), MB_DEVICE_ADDR1, MB_PARAM_HOLDING,
  TEST_HOLD_REG_START(holding_area1_end), 1,
  HOLD_OFFSET(holding_area1_end), PARAM_TYPE_U16, 2,
  OPTS(0x03, 0x06, 0x5555), PAR_PERMS_READ_WRITE_CUST_CMD },
```

随后 `mbc_master_get_parameter` / `mbc_master_set_parameter` 会用这两个功能码读写该 CID。

### 5. 注册字典

```c
mbc_master_set_descriptor(master_handle,
                          &device_parameters[0],
                          num_device_parameters);
```

### 6. 寄存器类型 ↔ 数据类型 ↔ 功能码对照

| `mb_param_type_t` | 默认读 FC | 默认写 FC | 常用 `param_type` |
|---|---|---|---|
| `MB_PARAM_HOLDING` | 0x03 | 0x06 / 0x10 | `PARAM_TYPE_FLOAT`、`PARAM_TYPE_U16`、`PARAM_TYPE_ASCII`、`PARAM_TYPE_U32_ABCD`… |
| `MB_PARAM_INPUT` | 0x04 | （只读） | `PARAM_TYPE_FLOAT`、`PARAM_TYPE_U16`、`PARAM_TYPE_U32` |
| `MB_PARAM_COIL` | 0x01 | 0x05 / 0x0F | `PARAM_TYPE_U8` |
| `MB_PARAM_DISCRETE` | 0x02 | （只读） | `PARAM_TYPE_U8` |

## 常见错误

| 错误 | 原因 | 解决方法 |
|---|---|---|
| 读到的 float 全错 | `mb_size` 当成字节数（应寄存器数） | float = 2 个寄存器，`param_size` 才是 4 字节 |
| 两个 CID 行为串扰 | CID 或 `param_key` 重复 | CID 唯一、`param_key` 唯一，用枚举管理 |
| `param_offset = 0` 导致取实例失败 | 0 被主站示例当作“未设置” | 主站侧宏用 `offsetof(...) + 1`，取实例时再 `−1` |
| 编译报 `PARAM_TYPE_FLOAT_CDAB` 未定义 | 未启用扩展类型 | `CONFIG_FMB_EXT_TYPE_SUPPORT=y` |
| ASCII / 二进制读不全 | `mb_size`（寄存器数）小于实际长度 | 每寄存器 2 字节；`param_size` = 字节数 |
| 线圈状态位对不齐 | 把 `mb_size` 当字节数 | Coil/Discrete 的 `mb_size` 单位是 **位** |

## 参考

- `espressif-repos/esp-modbus/examples/serial/mb_serial_master/main/serial_master.c`
- `espressif-repos/esp-modbus/modbus/mb_controller/common/include/esp_modbus_master.h`
- `espressif-repos/esp-modbus/docs/en/master_api_overview.rst`（Table 1/2/3 映射示例）
- `recipes/extended_types.md`、`recipes/custom_handlers.md`
