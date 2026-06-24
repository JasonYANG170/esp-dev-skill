# esp-modbus API Quick Reference

All signatures below are taken verbatim from the repository headers under `modbus/mb_controller/common/include/`. The single public include for applications is `mbcontroller.h`, which pulls in `esp_modbus_common.h`, `esp_modbus_master.h`, `esp_modbus_slave.h`, and (when `CONFIG_FMB_EXT_TYPE_SUPPORT=y`) `mb_endianness_utils.h`.

Every master/slave object is identified by a `void *ctx` handle returned by its `create_*` constructor; that handle is the **first parameter** of every later call.

## Common (master + slave)

```c
// Header: esp_modbus_common.h
esp_err_t mbc_set_handler(void *ctx, uint8_t func_code, mb_fn_handler_fp handler);
esp_err_t mbc_get_handler(void *ctx, uint8_t func_code, mb_fn_handler_fp *handler);
esp_err_t mbc_delete_handler(void *ctx, uint8_t func_code);
esp_err_t mbc_get_handler_count(void *ctx, uint16_t *count);
```

Function-handler prototype (`mb_types.h`):

```c
typedef mb_exception_t (*mb_fn_handler_fp)(void *pinst, uint8_t *frame_ptr, uint16_t *len_buf);
```

## Master API (`esp_modbus_master.h`)

### Lifecycle

```c
esp_err_t mbc_master_create_serial(mb_communication_info_t *config, void **ctx);
esp_err_t mbc_master_create_tcp(mb_communication_info_t *config, void **ctx);
esp_err_t mbc_master_start(void *ctx);
esp_err_t mbc_master_stop(void *ctx);
esp_err_t mbc_master_delete(void *ctx);
esp_err_t mbc_master_lock(void *ctx);
esp_err_t mbc_master_unlock(void *ctx);
```

### Data Dictionary & parameter access

```c
esp_err_t mbc_master_set_descriptor(void *ctx,
                                    const mb_parameter_descriptor_t *descriptor,
                                    const uint16_t num_elements);

esp_err_t mbc_master_send_request(void *ctx, mb_param_request_t *request, void *data_ptr);

esp_err_t mbc_master_get_cid_info(void *ctx, uint16_t cid,
                                  const mb_parameter_descriptor_t **param_info);

esp_err_t mbc_master_get_parameter(void *ctx, uint16_t cid, uint8_t *value, uint8_t *type);
esp_err_t mbc_master_get_parameter_with(void *ctx, uint16_t cid, uint8_t uid,
                                        uint8_t *value, uint8_t *type);
esp_err_t mbc_master_set_parameter(void *ctx, uint16_t cid, uint8_t *value, uint8_t *type);
esp_err_t mbc_master_set_parameter_with(void *ctx, uint16_t cid, uint8_t uid,
                                        uint8_t *value, uint8_t *type);

esp_err_t mbc_master_set_param_data(void *dest, void *src,
                                    mb_descr_type_t param_type, size_t param_size);
uint8_t   mbc_master_get_command(const mb_parameter_descriptor_t *descr, mb_param_mode_t mode);
```

### Optional weak register callbacks (legacy mapping helpers)

```c
mb_err_enum_t mbc_reg_holding_master_cb(mb_base_t *inst, uint8_t *reg_buffer,
                                        uint16_t reg_address, uint16_t num_regs,
                                        mb_reg_mode_enum_t mode);
mb_err_enum_t mbc_reg_input_master_cb(mb_base_t *inst, uint8_t *reg_buffer,
                                      uint16_t reg_address, uint16_t num_regs);
mb_err_enum_t mbc_reg_discrete_master_cb(mb_base_t *inst, uint8_t *reg_buffer,
                                         uint16_t reg_address, uint16_t n_discrete);
mb_err_enum_t mbc_reg_coils_master_cb(mb_base_t *inst, uint8_t *reg_buffer,
                                      uint16_t reg_addr, uint16_t ncoils,
                                      mb_reg_mode_enum_t mode);
```

## Slave API (`esp_modbus_slave.h`)

### Lifecycle

```c
esp_err_t mbc_slave_create_serial(mb_communication_info_t *config, void **ctx);
esp_err_t mbc_slave_create_tcp(mb_communication_info_t *config, void **ctx);
void      mbc_slave_init_iface(void *ctx);
esp_err_t mbc_slave_start(void *ctx);
esp_err_t mbc_slave_stop(void *ctx);
esp_err_t mbc_slave_delete(void *ctx);
esp_err_t mbc_slave_lock(void *ctx);
esp_err_t mbc_slave_unlock(void *ctx);
```

### Register areas & events

```c
esp_err_t        mbc_slave_set_descriptor(void *ctx, mb_register_area_descriptor_t descr_data);
mb_event_group_t mbc_slave_check_event(void *ctx, mb_event_group_t group);
esp_err_t        mbc_slave_get_param_info(void *ctx, mb_param_info_t *reg_info, uint32_t timeout);
```

### Slave ID (FC 0x11, needs `CONFIG_FMB_CONTROLLER_SLAVE_ID_SUPPORT`)

```c
esp_err_t mbc_set_slave_id(void *ctx, uint8_t uid, bool is_running,
                           uint8_t const *data_ptr, uint8_t data_len);
esp_err_t mbc_get_slave_id(void *ctx, uint8_t const *data_ptr, uint8_t *data_len);
```

`mbc_set_slave_id`: `uid` = short slave address, `is_running` = running-status byte reported to master, `data_ptr`/`data_len` = vendor-specific extension (`data_len` <= `CONFIG_FMB_CONTROLLER_SLAVE_ID_MAX_SIZE`). Returns `ESP_OK` or `ESP_ERR_INVALID_ARG`.

`mbc_get_slave_id`: `data_ptr` = buffer to receive the ID bytes, `data_len` is in/out (in: allocated buffer size; out: actual length). Returns `ESP_OK`, `ESP_ERR_INVALID_RESPONSE` (ID not set), or `ESP_ERR_INVALID_STATE` (buffer too small).

Header note (`esp_modbus_slave.h`): Report Slave ID support is intentionally included for the TCP slave as well, not only serial.

Internal macros (`mb_config.h`) that wire the feature to Kconfig:

```c
#define MB_FUNC_OTHER_REP_SLAVEID_ENABLED  (CONFIG_FMB_CONTROLLER_SLAVE_ID_SUPPORT)
#define MB_FUNC_OTHER_REP_SLAVEID_BUF      (CONFIG_FMB_CONTROLLER_SLAVE_ID_MAX_SIZE)
```

Master-side retrieval uses the generic request API with `.command = 0x11`; see `recipes/slave_device_id.md` (slave set side) and `recipes/serial_master.md` §6 (master read side).

### Optional weak register callbacks

```c
mb_err_enum_t mbc_reg_holding_slave_cb(mb_base_t *inst, uint8_t *reg_buffer,
                                       uint16_t address, uint16_t n_regs,
                                       mb_reg_mode_enum_t mode);
mb_err_enum_t mbc_reg_input_slave_cb(mb_base_t *inst, uint8_t *reg_buffer,
                                     uint16_t address, uint16_t n_regs);
mb_err_enum_t mbc_reg_discrete_slave_cb(mb_base_t *inst, uint8_t *reg_buffer,
                                        uint16_t address, uint16_t n_discrete);
mb_err_enum_t mbc_reg_coils_slave_cb(mb_base_t *inst, uint8_t *reg_buffer,
                                     uint16_t address, uint16_t n_coils,
                                     mb_reg_mode_enum_t mode);
```

## Key types

### `mb_communication_info_t` (union, `esp_modbus_common.h`)

```c
typedef union {
    mb_comm_mode_t       mode;          // MB_RTU / MB_ASCII / MB_TCP / MB_UDP
    mb_common_opts_t     common_opts;
    mb_tcp_opts_t        tcp_opts;      // when CONFIG_FMB_COMM_MODE_TCP_EN
    mb_serial_opts_t     ser_opts;      // when RTU or ASCII enabled
} mb_communication_info_t;
```

`mb_serial_opts_t` (`mb_port_types.h`): `mode`, `port` (uart_port_t), `uid`, `response_tout_ms`, `test_tout_us`, `baudrate`, `data_bits`, `stop_bits`, `parity`.

`mb_tcp_opts_t` (`mb_port_types.h`): `mode`, `port`, `uid`, `response_tout_ms`, `test_tout_us`, `addr_type` (`MB_IPV4`/`MB_IPV6`), `ip_addr_table`, `ip_netif_ptr`, `dns_name`, `start_disconnected`.

### `mb_parameter_descriptor_t` (master Data Dictionary entry, `esp_modbus_master.h`)

```c
typedef struct {
    uint16_t            cid;
    const char         *param_key;
    const char         *param_units;
    uint8_t             mb_slave_addr;
    mb_param_type_t     mb_param_type;     // MB_PARAM_HOLDING / INPUT / COIL / DISCRETE / CUSTOM
    uint16_t            mb_reg_start;      // 0-based
    uint16_t            mb_size;           // Holding/Input: registers; Coil/Discrete: bits
    uint32_t            param_offset;      // instance offset in storage struct (bytes)
    mb_descr_type_t     param_type;        // PARAM_TYPE_FLOAT / U16 / *_ABCD ... (esp_modbus_master.h)
    size_t              param_size;        // bytes
    mb_parameter_opt_t  param_opts;        // {opt1,opt2,opt3} = {min,max,step} or cust cmds
    mb_param_perms_t    access;            // PAR_PERMS_READ_WRITE_TRIGGER ...
} mb_parameter_descriptor_t;
```

### `mb_param_request_t` (for `mbc_master_send_request`)

```c
typedef struct {
    uint8_t  slave_addr;
    uint8_t  command;                       // Modbus function code
    uint16_t reg_start;
    uint16_t reg_size;
    struct {
        uint16_t wr_reg_start;              // only used by FC 0x17
        uint16_t wr_reg_size;
    } wr_rd_multi_reg_func;
} mb_param_request_t;
```

### `mb_register_area_descriptor_t` (slave, `esp_modbus_slave.h`)

```c
typedef struct {
    uint16_t          start_offset;
    mb_param_type_t   type;
    mb_param_access_t access;               // MB_ACCESS_RW / RO / WO
    void             *address;
    size_t            size;                 // BYTES
} mb_register_area_descriptor_t;
```

### `mb_param_info_t` (slave event info, `esp_modbus_slave.h`)

```c
typedef struct {
    uint32_t          time_stamp;           // us
    uint16_t          mb_offset;
    mb_event_group_t  type;
    uint8_t          *address;
    size_t            size;                 // registers
} mb_param_info_t;
```

## Enums

### `mb_comm_mode_t` (`mb_types.h`)
`MB_RTU`, `MB_ASCII`, `MB_TCP`, `MB_UDP`

### `mb_param_type_t` (`esp_modbus_common.h`)
`MB_PARAM_HOLDING = 0`, `MB_PARAM_INPUT`, `MB_PARAM_COIL`, `MB_PARAM_DISCRETE`, `MB_PARAM_COUNT`, `MB_PARAM_CUSTOM`, `MB_PARAM_UNKNOWN = 0xFF`

### `mb_event_group_t` (`esp_modbus_common.h`)
`MB_EVENT_HOLDING_REG_WR = BIT0`, `MB_EVENT_HOLDING_REG_RD = BIT1`, `MB_EVENT_INPUT_REG_RD = BIT3`, `MB_EVENT_COILS_WR = BIT4`, `MB_EVENT_COILS_RD = BIT5`, `MB_EVENT_DISCRETE_RD = BIT6`, `MB_EVENT_STACK_STARTED = BIT7`, `MB_EVENT_STACK_CONNECTED = BIT8`

### `mb_param_access_t` (`esp_modbus_slave.h`)
`MB_ACCESS_RW = 0x0000`, `MB_ACCESS_RO = 0x0001`, `MB_ACCESS_WO = 0x0002`

### `mb_addr_type_t` (`mb_port_types.h`)
`MB_NOIP = 0`, `MB_IPV4 = 1`, `MB_IPV6 = 2`

### `mb_exception_t` (`mb_types.h`)
`MB_EX_NONE`, `MB_EX_ILLEGAL_FUNCTION`, `MB_EX_ILLEGAL_DATA_ADDRESS`, `MB_EX_ILLEGAL_DATA_VALUE`, `MB_EX_SLAVE_DEVICE_FAILURE`, `MB_EX_ACKNOWLEDGE`, `MB_EX_SLAVE_BUSY`, `MB_EX_MEMORY_PARITY_ERROR`, `MB_EX_GATEWAY_PATH_FAILED`, `MB_EX_GATEWAY_TGT_FAILED`, `MB_EX_CRITICAL`

### `mb_err_enum_t` (`mb_types.h`) → esp_err_t
`MB_ENOERR`→`ESP_OK`; `MB_ENOREG`→`ESP_ERR_NOT_SUPPORTED` (0x106); `MB_ETIMEDOUT`→`ESP_ERR_TIMEOUT` (0x107); `MB_EILLFUNC`/`MB_ERECVDATA`→`ESP_ERR_INVALID_RESPONSE` (0x108); `MB_EBUSY`/`MB_EILLSTATE`/`MB_ENOCONN`→`ESP_ERR_INVALID_STATE` (0x103)

### `mb_param_perms_t` (`esp_modbus_master.h`)
`PAR_PERMS_READ`, `PAR_PERMS_WRITE`, `PAR_PERMS_TRIGGER`, `PAR_PERMS_CUST_CMD`, plus combinations `PAR_PERMS_READ_WRITE`, `PAR_PERMS_READ_TRIGGER`, `PAR_PERMS_WRITE_TRIGGER`, `PAR_PERMS_READ_WRITE_TRIGGER`, `PAR_PERMS_READ_WRITE_CUST_CMD`.

### `mb_descr_type_t` (extended types, `esp_modbus_master.h`)
Compatibility: `PARAM_TYPE_U8`, `PARAM_TYPE_U16`, `PARAM_TYPE_U32`, `PARAM_TYPE_FLOAT`, `PARAM_TYPE_ASCII`, `PARAM_TYPE_BIN`.
Extended (need `CONFIG_FMB_EXT_TYPE_SUPPORT`): `PARAM_TYPE_I8_A`, `PARAM_TYPE_I8_B`, `PARAM_TYPE_U8_A`, `PARAM_TYPE_U8_B`, `PARAM_TYPE_I16_AB`, `PARAM_TYPE_I16_BA`, `PARAM_TYPE_U16_AB`, `PARAM_TYPE_U16_BA`, `PARAM_TYPE_I32_ABCD/CDAB/BADC/DCBA`, `PARAM_TYPE_U32_ABCD/CDAB/BADC/DCBA`, `PARAM_TYPE_FLOAT_ABCD/CDAB/BADC/DCBA`, `PARAM_TYPE_I64_ABCDEFGH/HGFEDCBA/GHEFCDAB/BADCFEHG`, `PARAM_TYPE_U64_ABCDEFGH/HGFEDCBA/GHEFCDAB/BADCFEHG`, `PARAM_TYPE_DOUBLE_ABCDEFGH/HGFEDCBA/GHEFCDAB/BADCFEHG`.

## Endianness conversion API (`mb_endianness_utils.h`, needs `CONFIG_FMB_EXT_TYPE_SUPPORT`)

Sized array types:

```c
typedef uint8_t val_16_arr[2];
typedef uint8_t val_32_arr[4];
typedef uint8_t val_64_arr[8];
```

Each type has paired `mb_set_<type>` and `mb_get_<type>` (signatures verbatim from the header):

```c
// 8-bit (val_16_arr)
int8_t   mb_get_int8_a(val_16_arr *);  uint16_t mb_set_int8_a(val_16_arr *, int8_t);
int8_t   mb_get_int8_b(val_16_arr *);  uint16_t mb_set_int8_b(val_16_arr *, int8_t);
uint8_t  mb_get_uint8_a(val_16_arr *); uint16_t mb_set_uint8_a(val_16_arr *, uint8_t);
uint8_t  mb_get_uint8_b(val_16_arr *); uint16_t mb_set_uint8_b(val_16_arr *, uint8_t);

// 16-bit (val_16_arr)
uint16_t mb_set_int16_ab(val_16_arr *, int16_t);
uint16_t mb_set_int16_ba(val_16_arr *, int16_t);
uint16_t mb_set_uint16_ab(val_16_arr *, uint16_t);
uint16_t mb_set_uint16_ba(val_16_arr *, uint16_t);

// 32-bit (val_32_arr)
int32_t  mb_get_int32_abcd/cdab/badc/dcba(val_32_arr *);
uint32_t mb_get_uint32_abcd/cdab/badc/dcba(val_32_arr *);
float    mb_get_float_abcd/badc/cdab/dcba(val_32_arr *);

// 64-bit (val_64_arr)
double  mb_get_double_abcdefgh/hgfedcba/ghefcdab/badcfehg(val_64_arr *);
int64_t mb_get_int64_abcdefgh/hgfedcba/ghefcdab/badcfehg(val_64_arr *);
uint64_t mb_get_uint64_abcdefgh/hgfedcba/ghefcdab/badcfehg(val_64_arr *);
```

> The matching `mb_set_*` counterparts for the 32/64-bit types follow the same naming (`mb_set_uint32_abcd`, `mb_set_float_cdab`, `mb_set_double_hgfedcba`, `mb_set_uint64_ghefcdab`, …) and are declared in the same header.

## Important macros (`esp_modbus_common.h`)

```c
#define MB_SLAVE_ADDR_PLACEHOLDER   (0xFF)    // for mbc_master_get/set_parameter_with
#define MB_CONTROLLER_STACK_SIZE    (CONFIG_FMB_CONTROLLER_STACK_SIZE)
#define MB_CONTROLLER_PRIORITY      (CONFIG_FMB_PORT_TASK_PRIO - 1)
#define MB_PORT_TASK_AFFINITY       (CONFIG_FMB_PORT_TASK_AFFINITY)
#define MB_PAR_INFO_TOUT            (10)
#define MB_PARITY_NONE              (UART_PARITY_DISABLE)
```
