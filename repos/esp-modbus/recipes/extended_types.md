# 扩展数据类型与字节序转换

> **适用摘要**: 启用 `CONFIG_FMB_EXT_TYPE_SUPPORT`，使用 `PARAM_TYPE_U32_ABCD` / `PARAM_TYPE_FLOAT_CDAB` / `PARAM_TYPE_DOUBLE_HGFEDCBA` 等扩展类型，以及 `mb_set_float_abcd` / `mb_get_uint32_dcba` 等字节序转换助手，正确处理第三方设备的 32/64 位值。

## 触发意图

- "Modbus float ABCD 字节序"
- "32 位寄存器解析"
- "第三方设备 double"
- "mb_set_float_cdab"
- "扩展数据类型 CONFIG_FMB_EXT_TYPE_SUPPORT"

## 前置条件

| 条件 | 要求 |
|---|---|
| Kconfig | `CONFIG_FMB_EXT_TYPE_SUPPORT=y`（默认关） |
| 头文件 | `mbcontroller.h`（启用后会自动包含 `mb_endianness_utils.h`） |
| 参考示例 | `examples/serial/mb_serial_slave/main/serial_slave.c`（`setup_reg_data`）、`examples/serial/mb_serial_master/main/serial_master.c`（字典） |
| 存储类型 | `val_16_arr[2]`、`val_32_arr[4]`、`val_64_arr[8]`（来自 `mb_endianness_utils.h`） |

## 分步说明

### 1. 启用扩展类型

`sdkconfig.defaults`：

```
CONFIG_FMB_EXT_TYPE_SUPPORT=y
```

或 `idf.py menuconfig` → Modbus configuration → "Modbus uses extended types to support third party devices"。

### 2. 字节序命名约定（来自 `docs/en/overview_messaging_and_mapping.rst`）

| 后缀 | 含义 |
|---|---|
| `ABCD` | 大端，高字节在前 |
| `CDAB` | 大端，寄存器顺序反转（小端 + 字节交换） |
| `BADC` | 小端，寄存器顺序反转 |
| `DCBA` | 小端，低字节在前 |

> 单个 16 位寄存器本身按 Modbus 规范是大端。`ABCD` 等后缀描述的是 **多个寄存器组合** 的字节顺序。

### 3. 主站：在数据字典里声明扩展类型

来自 `serial_master.c`（节选）：

```c
#if CONFIG_FMB_EXT_TYPE_SUPPORT
    { CID_HOLD_UINT32_ABCD, STR("UINT32_ABCD"), STR("__"), MB_DEVICE_ADDR1, MB_PARAM_HOLDING,
      TEST_HOLD_REG_START(holding_uint32_abcd), TEST_HOLD_REG_SIZE(holding_uint32_abcd),
      HOLD_OFFSET(holding_uint32_abcd), PARAM_TYPE_U32_ABCD,
      (TEST_HOLD_REG_SIZE(holding_uint32_abcd) << 1),
      OPTS(0, TEST_VALUE, TEST_VALUE), PAR_PERMS_READ_WRITE_TRIGGER },

    { CID_HOLD_FLOAT_CDAB, STR("FLOAT_CDAB"), STR("__"), MB_DEVICE_ADDR1, MB_PARAM_HOLDING,
      TEST_HOLD_REG_START(holding_float_cdab), TEST_HOLD_REG_SIZE(holding_float_cdab),
      HOLD_OFFSET(holding_float_cdab), PARAM_TYPE_FLOAT_CDAB,
      (TEST_HOLD_REG_SIZE(holding_float_cdab) << 1),
      OPTS(0, TEST_VALUE, TEST_VALUE), PAR_PERMS_READ_WRITE_TRIGGER },

    { CID_HOLD_DOUBLE_HGFEDCBA, STR("DOUBLE_HGFEDCBA"), STR("__"), MB_DEVICE_ADDR1, MB_PARAM_HOLDING,
      TEST_HOLD_REG_START(holding_double_hgfedcba), TEST_HOLD_REG_SIZE(holding_double_hgfedcba),
      HOLD_OFFSET(holding_double_hgfedcba), PARAM_TYPE_DOUBLE_HGFEDCBA,
      (TEST_HOLD_REG_SIZE(holding_double_hgfedcba) << 1),
      OPTS(0, TEST_VALUE, TEST_VALUE), PAR_PERMS_READ_WRITE_TRIGGER },
#endif
```

读到的 `temp_data` 直接就是编译器原生表示（栈负责转换），可直接当 `uint32_t` / `float` / `double` 解释。

### 4. 从站：用 `mb_set_*` 把值写入存储

来自 `serial_slave.c::setup_reg_data`：

```c
#if CONFIG_FMB_EXT_TYPE_SUPPORT
    mb_set_uint8_a((val_16_arr *)&holding_reg_params.holding_u8_a[0], (uint8_t)0x55);
    mb_set_uint16_ab((val_16_arr *)&holding_reg_params.holding_u16_ab[0], (uint16_t)12345);
    mb_set_float_abcd((val_32_arr *)&holding_reg_params.holding_float_abcd[0], (float)12345.0);
    mb_set_float_cdab((val_32_arr *)&holding_reg_params.holding_float_cdab[0], (float)12345.0);
    mb_set_uint32_dcba((val_32_arr *)&holding_reg_params.holding_uint32_dcba[0], (uint32_t)12345);
    mb_set_double_abcdefgh((val_64_arr *)&holding_reg_params.holding_double_abcdefgh[0], (double)12345.0);
    mb_set_double_hgfedcba((val_64_arr *)&holding_reg_params.holding_double_hgfedcba[0], (double)12345.0);
#endif
```

### 5. 从站：用 `mb_get_*` 读回实际值

```c
float f = mb_get_float_abcd((val_32_arr *)&holding_reg_params.holding_float_abcd[0]);
double d = mb_get_double_ghefcdab((val_64_arr *)&holding_reg_params.holding_double_ghefcdab[0]);
ESP_LOGI("TEST", "abcd: %f, ghefcdab: %lf", f, d);
```

### 6. 写入时加锁

```c
portENTER_CRITICAL(&param_lock);
mb_set_float_abcd(&holding_float_abcd[0], (float)12345.0);
portEXIT_CRITICAL(&param_lock);
```

## 助手函数族（节选自 `mb_endianness_utils.h`）

每个类型都有 `mb_set_<type>` 与 `mb_get_<type>` 配对，类型族包括：

- 8 位：`mb_set_uint8_a` / `mb_get_uint8_a`、`_b`；`mb_set_int8_a` / `_b`
- 16 位：`mb_set_uint16_ab` / `_ba`；`mb_set_int16_ab` / `_ba`
- 32 位：`mb_set_uint32_abcd` / `_cdab` / `_badc` / `_dcba`；`mb_set_int32_*`；`mb_set_float_abcd` …
- 64 位：`mb_set_uint64_abcdefgh` / `_hgfedcba` / `_ghefcdab` / `_badcfehg`；`mb_set_int64_*`；`mb_set_double_*`

## 常见错误

| 错误 | 原因 | 解决方法 |
|---|---|---|
| 编译报 `PARAM_TYPE_FLOAT_CDAB` 未声明 | 未启用扩展类型 | `CONFIG_FMB_EXT_TYPE_SUPPORT=y` |
| 读到的 float 数值与设备显示不符 | 字节序选错（如该用 CDAB 却用了 ABCD） | 查设备手册的寄存器映射表，选对应后缀 |
| `mb_set_float_abcd` 未定义链接错误 | 头文件没被包含（应自动包含） | 确认启用扩展类型；必要时显式 `#include "mb_endianness_utils.h"` |
| 数组类型不匹配编译警告 | 直接传 `float*` 给助手 | 用 `val_32_arr *` / `val_64_arr *` 强转 |
| 多线程读写扩展值错乱 | 未加锁 | 用 `portENTER_CRITICAL` 或 `mbc_slave_lock` 包裹 |

## 参考

- `espressif-repos/esp-modbus/modbus/mb_controller/common/include/mb_endianness_utils.h`
- `espressif-repos/esp-modbus/modbus/mb_controller/common/include/esp_modbus_master.h`（`mb_descr_type_t` 枚举）
- `espressif-repos/esp-modbus/docs/en/overview_messaging_and_mapping.rst`（Table 1/2/3 类型与字节序）
- `espressif-repos/esp-modbus/examples/serial/mb_serial_slave/main/serial_slave.c`（`setup_reg_data`）
- `recipes/data_dictionary.md`
