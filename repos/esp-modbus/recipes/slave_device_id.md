# 从站设备识别（Report Slave ID / FC 0x11 从站侧）

> **适用摘要**: 在 ESP32 Modbus 从站上设置厂商自定义的设备识别信息（短 UID、运行状态字节、厂商扩展数据），供主站通过标准命令 0x11 Report Slave ID 读回。覆盖 `mbc_set_slave_id` / `mbc_get_slave_id` 的调用时机、`INIT_DEV_ID` 结构体的构造方式，以及配套的 Kconfig 三元组。

## 触发意图

- "设置从站设备 ID"
- "Report Slave ID"
- "从站厂商信息"
- "mbc_set_slave_id"
- "mbc_get_slave_id"
- "FC 0x11 从站侧"
- "Modbus 设备识别"

## 前置条件

| 条件 | 要求 |
|---|---|
| 参考示例 | `espressif-repos/esp-modbus/examples/serial/mb_serial_slave/`（从站侧设置）；`espressif-repos/esp-modbus/examples/serial/mb_serial_master/`（主站侧读回） |
| Kconfig | `CONFIG_FMB_CONTROLLER_SLAVE_ID_SUPPORT=y`（启用 FC 0x11 支持，默认开启）；`CONFIG_FMB_CONTROLLER_SLAVE_ID=0x00112233`（默认短 ID，可被覆盖）；`CONFIG_FMB_CONTROLLER_SLAVE_ID_MAX_SIZE`（厂商数据最大字节数） |
| 协议栈状态 | 调用 `mbc_set_slave_id` 前，从站对象必须已通过 `mbc_slave_create_serial` / `mbc_slave_create_tcp` 创建；建议在 `mbc_slave_start` 之后调用，以便同时上报真实的运行状态 |
| 互配主站 | 主站通过 `mbc_master_send_request` 以 `.command = 0x11` 读取；主站侧用法见 `recipes/serial_master.md` 第 6 节 |

## 分步说明

### 1. 启用 Kconfig 支持

FC 0x11 的支持由 `CONFIG_FMB_CONTROLLER_SLAVE_ID_SUPPORT` 控制（在 `modbus/mb_objects/include/mb_config.h` 中映射为 `MB_FUNC_OTHER_REP_SLAVEID_ENABLED`，缓冲区大小由 `MB_FUNC_OTHER_REP_SLAVEID_BUF` 即 `CONFIG_FMB_CONTROLLER_SLAVE_ID_MAX_SIZE` 决定）。默认开启，但如果被关闭，`mbc_set_slave_id` / `mbc_get_slave_id` 的声明会被头文件 `#if` 条件编译掉，调用会编译失败。

```
# sdkconfig.defaults
CONFIG_FMB_CONTROLLER_SLAVE_ID_SUPPORT=y
CONFIG_FMB_CONTROLLER_SLAVE_ID=0x00112233
```

> 若不调用 `mbc_set_slave_id`，从站启动后默认使用 `CONFIG_FMB_CONTROLLER_SLAVE_ID` 指定的值，文档给出的默认字节序列为 `01 ff 33 22 11`（即 UID=01、运行状态=ff、厂商数据=33 22 11）。

### 2. 构造厂商设备 ID 结构体（INIT_DEV_ID 宏）

来自 `examples/serial/mb_serial_slave/main/serial_slave.c`。该宏定义一个包含短 UID、运行状态字节、长度、标记字节、序列号与设备名称的紧凑结构体，并按设备名称字符串的实际长度计算 `length` 字段（协议要求 `length` 反映厂商数据的真实长度）。

```c
#define MB_SLAVE_NAME_MAX_LEN 32

#define INIT_DEV_ID(struct_name, uid, running, serial, name) static struct {    \
        uint8_t slave_uid;                                                      \
        uint8_t is_running;                                                     \
        uint8_t length;                                                         \
        uint8_t marker;                                                         \
        uint32_t serial_number;                                                 \
        char dev_name[MB_SLAVE_NAME_MAX_LEN];                                   \
    } struct_name = {                                                           \
        .slave_uid = (uid),                                                     \
        .is_running = (running),                                                \
        .marker = 0x55,                                                         \
        .length = (sizeof(struct_name) - sizeof(struct_name.dev_name)           \
                            + strlen((name)) - 2),                              \
        .serial_number =(serial),                                               \
        .dev_name = name,                                                       \
    };
```

实例化（`uid` 与 `running` 在第 4 步由 `mbc_set_slave_id` 单独传入，这里结构体中对应字段仅用于厂商数据缓冲区的内部记录）：

```c
INIT_DEV_ID(new_id_struct, 0x00, 0x00, 0x11223344, "esp_modbus_serial_slave");
```

### 3. 创建从站并启动协议栈

`mbc_set_slave_id` 期望对象已经创建；示例是在 `mbc_slave_start` 之后调用，以便把 `start` 的返回值映射成 `is_running` 字节。

```c
static void *mbc_slave_handle = NULL;

mb_communication_info_t comm_config = {
    .ser_opts.port = MB_PORT_NUM,
    .ser_opts.mode = MB_RTU,
    .ser_opts.baudrate = MB_DEV_SPEED,
    .ser_opts.parity = MB_PARITY_NONE,
    .ser_opts.uid = MB_SLAVE_ADDR,
    .ser_opts.data_bits = UART_DATA_8_BITS,
    .ser_opts.stop_bits = UART_STOP_BITS_1
};
ESP_ERROR_CHECK(mbc_slave_create_serial(&comm_config, &mbc_slave_handle));

// ... mbc_slave_set_descriptor、uart_set_pin、uart_set_mode(RS485_HALF_DUPLEX) ...

esp_err_t err = mbc_slave_start(mbc_slave_handle);
```

### 4. 调用 mbc_set_slave_id 写入设备识别信息

签名（`esp_modbus_slave.h`，受 `CONFIG_FMB_CONTROLLER_SLAVE_ID_SUPPORT` 保护）：

```c
esp_err_t mbc_set_slave_id(void *ctx, uint8_t uid, bool is_running,
                           uint8_t const *data_ptr, uint8_t data_len);
```

- `uid`：短从站地址（通常等于 `comm_config.ser_opts.uid`）
- `is_running`：运行状态字节，会原样回传给主站
- `data_ptr`：指向厂商扩展数据缓冲区的指针
- `data_len`：厂商数据字节数（不可超过 `CONFIG_FMB_CONTROLLER_SLAVE_ID_MAX_SIZE`）

示例的调用方式（从 `new_id_struct.length` 字段起作为厂商数据，长度即 `new_id_struct.length`）：

```c
uint8_t is_running = (bool)(err == ESP_OK);   // start 成功 = 在运行

// The way to set Slave ID fields to retrieve it by master using report slave ID command.
err = mbc_set_slave_id(mbc_slave_handle,
                       comm_config.ser_opts.uid,
                       is_running,
                       &new_id_struct.length,
                       new_id_struct.length);
if (err == ESP_OK) {
    ESP_LOGW("SET_SLAVE_ID", "dev_name: %s", (char *)new_id_struct.dev_name);
    ESP_LOG_BUFFER_HEX_LEVEL("SET_SLAVE_ID",
                             (void *)&new_id_struct.length,
                             new_id_struct.length, ESP_LOG_WARN);
} else {
    ESP_LOGE("SET_SLAVE_ID", "Set slave ID fail, err=%d.", err);
}
```

文档（`slave_api_overview.rst`）给出的简化用法是直接传一个字符串作为厂商数据：

```c
const char *pdevice_name = "my_slave_device_description";
bool is_started = (bool)(err == ESP_OK);
mbc_set_slave_id(mbc_slave_handle, comm_config.ser_opts.uid, is_started,
                 (uint8_t *)pdevice_name, strlen(pdevice_name));
```

### 5. 用 mbc_get_slave_id 回读校验

签名（`esp_modbus_slave.h`）：

```c
esp_err_t mbc_get_slave_id(void *ctx, uint8_t const *data_ptr, uint8_t *data_len);
```

`data_len` 是入参/出参：传入分配缓冲区大小，返回实际写入字节数。缓冲区至少要 `CONFIG_FMB_CONTROLLER_SLAVE_ID_MAX_SIZE` 字节。

```c
uint8_t current_slave_id[CONFIG_FMB_CONTROLLER_SLAVE_ID_MAX_SIZE] = {0};
uint8_t length = sizeof(current_slave_id);

esp_err_t r = mbc_get_slave_id(mbc_slave_handle, &current_slave_id[0], &length);
if (r == ESP_OK) {
    ESP_LOGW("GET_SLAVE_ID", "Get slave ID, length=%u.", length);
    ESP_LOG_BUFFER_HEX_LEVEL("GET_SLAVE_ID",
                             (void *)current_slave_id, length, ESP_LOG_WARN);
} else {
    ESP_LOGE("GET_SLAVE_ID", "Get slave ID fail, err=%d.", r);
}
```

### 6. 主站侧读回（FC 0x11）

供对照——主站通过 `mbc_master_send_request` 以 `.command = 0x11` 读取从站写入的识别信息（来自 `examples/serial/mb_serial_master/main/serial_master.c`，完整说明见 `recipes/serial_master.md` 第 6 节）：

```c
mb_param_request_t req = {
    .slave_addr = MB_DEVICE_ADDR1,                              // 目标从站 UID
    .command = 0x11,                                            // Report Slave ID
    .reg_start = 0,                                             // 必须为 0
    .reg_size = (CONFIG_FMB_CONTROLLER_SLAVE_ID_MAX_SIZE >> 1), // 期望长度（寄存器数）
};
uint8_t info_buf[CONFIG_FMB_CONTROLLER_SLAVE_ID_MAX_SIZE] = {0};
mbc_master_send_request(master_handle, &req, &info_buf[0]);
```

返回的 `info_buf` 中第一字节是从站 UID，第二字节是运行状态，其后是厂商扩展数据（即从站 `mbc_set_slave_id` 传入的 `data_ptr` 内容）。

## 常见错误

| 错误 | 原因 | 解决方法 |
|---|---|---|
| `mbc_set_slave_id` / `mbc_get_slave_id` 未声明（隐式声明警告或链接失败） | `CONFIG_FMB_CONTROLLER_SLAVE_ID_SUPPORT` 未启用，头文件中这两个函数被 `#if` 条件编译掉 | 在 sdkconfig 中设 `CONFIG_FMB_CONTROLLER_SLAVE_ID_SUPPORT=y` |
| `mbc_set_slave_id` 返回 `ESP_ERR_INVALID_ARG` (0x102) | `data_len` 超过 `CONFIG_FMB_CONTROLLER_SLAVE_ID_MAX_SIZE`，或 `data_ptr` 为 NULL | 用 `CONFIG_FMB_CONTROLLER_SLAVE_ID_MAX_SIZE` 约束厂商数据长度；确保缓冲区非空 |
| 主站 FC 0x11 读取失败或返回空 | 从站在 `mbc_slave_start` 前调用 `mbc_set_slave_id`，或根本未调用（使用默认 ID 但主站期望厂商数据） | 在 `mbc_slave_start` 之后调用 `mbc_set_slave_id`；核对主站 `reg_size` 与从站厂商数据长度 |
| 主站读回的厂商数据被截断 | 主站 `req.reg_size`（寄存器数）小于实际厂商数据长度的一半 | 主站 `reg_size` 至少为 `(CONFIG_FMB_CONTROLLER_SLAVE_ID_MAX_SIZE >> 1)` |
| `mbc_get_slave_id` 返回 `ESP_ERR_INVALID_STATE` (0x103) | 传入的 `*data_len` 小于实际 ID 长度，缓冲区装不下 | 传入 `*data_len` 至少为 `CONFIG_FMB_CONTROLLER_SLAVE_ID_MAX_SIZE` |
| TCP 从站上报 ID 不生效 | 误以为 FC 0x11 只适用于串行 | 头文件注释明确指出 `0x11 Report Slave ID` 对 TCP 从站同样支持，调用方式完全相同 |

## 参考

- `espressif-repos/esp-modbus/examples/serial/mb_serial_slave/main/serial_slave.c`（`INIT_DEV_ID` 宏与 `mbc_set_slave_id` 调用）
- `espressif-repos/esp-modbus/examples/serial/mb_serial_master/main/serial_master.c`（主站 FC 0x11 读回）
- `espressif-repos/esp-modbus/docs/en/slave_api_overview.rst`（`mbc_set_slave_id` / `mbc_get_slave_id` 文档与示例）
- `espressif-repos/esp-modbus/docs/en/master_api_overview.rst`
- `espressif-repos/esp-modbus/modbus/mb_controller/common/include/esp_modbus_slave.h`（函数签名）
- `espressif-repos/esp-modbus/modbus/mb_objects/include/mb_config.h`（`MB_FUNC_OTHER_REP_SLAVEID_ENABLED` / `MB_FUNC_OTHER_REP_SLAVEID_BUF`）
- `recipes/serial_slave.md`、`recipes/serial_master.md`
