# Modbus 串行主站（RTU / ASCII）

> **适用摘要**: 在 ESP32 系列芯片上构建 Modbus 串行主站，覆盖 UART/RS485 初始化、数据字典（Data Dictionary）注册、按 CID 轮询从站参数、自定义命令、销毁。

> Version: selected repo/component version; align with the user project and dependency manifest.
> Evidence: `repos/esp-modbus/resources/`, source/examples in `repos/esp-modbus/`, and this recipe path `repos/esp-modbus/recipes/serial_master.md`.
> Validation: draft metadata added from repository routing; verify APIs, Kconfig symbols, and component dependencies against the selected version.

## 触发意图

- "做一个 Modbus 串行主机"
- "Modbus RTU master"
- "读取从站保持寄存器"
- "Modbus 主机轮询"
- "Data Dictionary 数据字典"

## 前置条件

| 条件 | 要求 |
|---|---|
| 参考示例 | `espressif-repos/esp-modbus/examples/serial/mb_serial_master/` |
| 共享参数组件 | `espressif-repos/esp-modbus/examples/mb_example_common/` |
| Kconfig | `CONFIG_FMB_COMM_MODE_RTU_EN=y`（或 ASCII）；`CONFIG_FMB_MASTER_TIMEOUT_MS_RESPOND` 大于从站最坏 RTT；`CONFIG_MB_UART_*` |
| 硬件 | 与从站共用同一 RS485 总线，波特率/校验/数据位一致，从站地址已知 |

## 分步说明

### 1. 定义 CID 枚举与数据字典

来自 `examples/serial/mb_serial_master/main/serial_master.c`。`mb_size` 单位是 **寄存器（2 字节）**，`param_size` 单位是 **字节**。

```c
#define HOLD_OFFSET(field) ((uint16_t)(offsetof(holding_reg_params_t, field) + 1))
#define INPUT_OFFSET(field)((uint16_t)(offsetof(input_reg_params_t,   field) + 1))
#define TEST_HOLD_REG_START(field) (HOLD_OFFSET(field) >> 1)
#define TEST_HOLD_REG_SIZE(field)  (sizeof(((holding_reg_params_t *)0)->field) >> 1)
#define TEST_INPUT_REG_START(field) (INPUT_OFFSET(field) >> 1)
#define TEST_INPUT_REG_SIZE(field)  (sizeof(((input_reg_params_t *)0)->field) >> 1)
#define STR(s)   ((const char *)(s))
#define OPTS(min_val, max_val, step_val) { .opt1 = min_val, .opt2 = max_val, .opt3 = step_val }

enum { MB_DEVICE_ADDR1 = 1 };
enum {
    CID_INP_DATA_0 = 0,
    CID_HOLD_DATA_0,
    CID_RELAY_P1,
    CID_DISCR_P1,
    CID_COUNT
};

const mb_parameter_descriptor_t device_parameters[] = {
    // { CID, Name, Units, Slave Addr, Reg Type, Reg Start, Reg Size(regs),
    //   Instance Offset, Data Type, Data Size(bytes), Options, Access }
    { CID_INP_DATA_0, STR("Data_channel_0"), STR("Volts"), MB_DEVICE_ADDR1, MB_PARAM_INPUT,
      TEST_INPUT_REG_START(input_data0), TEST_INPUT_REG_SIZE(input_data0),
      INPUT_OFFSET(input_data0), PARAM_TYPE_FLOAT, 4,
      OPTS(0, 100, 0), PAR_PERMS_READ_WRITE_TRIGGER },
    { CID_HOLD_DATA_0, STR("Humidity_1"), STR("%rH"), MB_DEVICE_ADDR1, MB_PARAM_HOLDING,
      TEST_HOLD_REG_START(holding_data0), TEST_HOLD_REG_SIZE(holding_data0),
      HOLD_OFFSET(holding_data0), PARAM_TYPE_FLOAT, 4,
      OPTS(-40, 50, 0), PAR_PERMS_READ_WRITE_TRIGGER },
    { CID_RELAY_P1, STR("RelayP1"), STR("on/off"), MB_DEVICE_ADDR1, MB_PARAM_COIL, 2, 6,
      0, PARAM_TYPE_U8, 1, OPTS(0xAA, 0x15, 0), PAR_PERMS_READ_WRITE_TRIGGER },
    { CID_DISCR_P1, STR("DiscreteInpP1"), STR("on/off"), MB_DEVICE_ADDR1, MB_PARAM_DISCRETE, 2, 7,
      0, PARAM_TYPE_U8, 1, OPTS(0xAA, 0x15, 0), PAR_PERMS_READ_WRITE_TRIGGER },
};
const uint16_t num_device_parameters =
    (sizeof(device_parameters) / sizeof(device_parameters[0]));
```

### 2. 创建主站对象

```c
#define MB_PORT_NUM   (CONFIG_MB_UART_PORT_NUM)
#define MB_DEV_SPEED  (CONFIG_MB_UART_BAUD_RATE)
static void *master_handle = NULL;

mb_communication_info_t comm = {
    .ser_opts.port = MB_PORT_NUM,
#if CONFIG_MB_COMM_MODE_ASCII
    .ser_opts.mode = MB_ASCII,
#elif CONFIG_MB_COMM_MODE_RTU
    .ser_opts.mode = MB_RTU,
#endif
    .ser_opts.baudrate = MB_DEV_SPEED,
    .ser_opts.parity = MB_PARITY_NONE,
    .ser_opts.uid = 0,                       // 主站不用
    .ser_opts.response_tout_ms = 1000,       // 0 表示用 CONFIG_FMB_MASTER_TIMEOUT_MS_RESPOND
    .ser_opts.data_bits = UART_DATA_8_BITS,
    .ser_opts.stop_bits = UART_STOP_BITS_1,
};
ESP_ERROR_CHECK(mbc_master_create_serial(&comm, &master_handle));
```

### 3. 设置 UART 引脚 + RS485 半双工 + 注册数据字典（顺序：start 之前）

```c
ESP_ERROR_CHECK(uart_set_pin(MB_PORT_NUM, CONFIG_MB_UART_TXD, CONFIG_MB_UART_RXD,
                             CONFIG_MB_UART_RTS, UART_PIN_NO_CHANGE));
ESP_ERROR_CHECK(mbc_master_set_descriptor(master_handle,
                                          &device_parameters[0],
                                          num_device_parameters));
ESP_ERROR_CHECK(uart_set_mode(MB_PORT_NUM, UART_MODE_RS485_HALF_DUPLEX));
ESP_ERROR_CHECK(mbc_master_start(master_handle));
```

### 4. 读取 / 写入参数（按 CID）

```c
const mb_parameter_descriptor_t *pdescr = NULL;
uint8_t temp_data[4] = {0};
uint8_t type = 0;

esp_err_t err = mbc_master_get_cid_info(master_handle, CID_HOLD_DATA_0, &pdescr);
if (err == ESP_OK && pdescr != NULL) {
    err = mbc_master_get_parameter(master_handle, pdescr->cid, temp_data, &type);
    if (err == ESP_OK) {
        float v = *(float *)temp_data;
        ESP_LOGI(TAG, "%s = %f", pdescr->param_key, v);
        // 回写：把目标值放进 temp_data 再调用 mbc_master_set_parameter
    } else {
        ESP_LOGE(TAG, "read fail err=0x%x (%s)", (int)err, esp_err_to_name(err));
    }
}
```

### 5. 发送自定义命令（FC 0x41 示例）

```c
char *pcustom_string = "Master";
mb_param_request_t req = {
    .slave_addr = MB_DEVICE_ADDR1,
    .command = 0x41,
    .reg_start = 0,
    .reg_size = (strlen(pcustom_string) >> 1)   // 寄存器数（偶数字节）
};
mbc_master_send_request(master_handle, &req, pcustom_string);
```

### 6. 读取从站 ID（FC 0x11，需 `CONFIG_FMB_CONTROLLER_SLAVE_ID_SUPPORT`）

```c
mb_param_request_t req = {
    .slave_addr = MB_DEVICE_ADDR1,
    .command = 0x11,
    .reg_start = 0,
    .reg_size = (CONFIG_FMB_CONTROLLER_SLAVE_ID_MAX_SIZE >> 1),
};
uint8_t info_buf[CONFIG_FMB_CONTROLLER_SLAVE_ID_MAX_SIZE] = {0};
mbc_master_send_request(master_handle, &req, &info_buf[0]);
```

### 7. 销毁

```c
ESP_ERROR_CHECK(mbc_master_delete(master_handle));
```

## 常见错误

| 错误 | 原因 | 解决方法 |
|---|---|---|
| `ESP_ERR_TIMEOUT` (0x107) | 从站未在超时内应答 | 调大 `CONFIG_FMB_MASTER_TIMEOUT_MS_RESPOND` / `ser_opts.response_tout_ms`；检查 RS485 接线、波特率/校验一致性、从站是否上电 |
| `ESP_ERR_NOT_SUPPORTED` (0x106) | 从站返回异常（请求的寄存器不存在） | 核对数据字典的 `mb_reg_start` / `mb_size` / `mb_param_type` 与从站寄存器表 |
| `ESP_ERR_INVALID_RESPONSE` (0x108) | 命令与寄存器类型不匹配，或 `mb_size` 错误 | Holding→0x03/0x10，Input→0x04，Coil→0x01/0x0F，Discrete→0x02 |
| `ESP_ERR_INVALID_STATE` (0x103) | 未 `start` 或协议栈 FSM 忙 | 检查初始化顺序；避免在前一请求未返回时再次调用 |
| 读出的 float 是乱值 | `mb_size` 写成了字节数而非寄存器数 | float = 2 个寄存器；`param_size` 才是 4 字节 |
| 广播无响应被当成错误 | 广播地址 0 不会有应答 | 区分单播与广播；广播后等 `CONFIG_FMB_MASTER_DELAY_MS_CONVERT` 再发下一帧 |

## 参考

- `espressif-repos/esp-modbus/examples/serial/mb_serial_master/main/serial_master.c`
- `espressif-repos/esp-modbus/examples/serial/mb_serial_master/main/Kconfig.projbuild`
- `espressif-repos/esp-modbus/docs/en/master_api_overview.rst`
- `espressif-repos/esp-modbus/docs/en/applications_and_references.rst`（错误码表）
- `recipes/data_dictionary.md`、`recipes/custom_handlers.md`
