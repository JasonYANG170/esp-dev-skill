# AGENTS.md — Supplementary Agent Guide

> Core principles, recipe index, pitfalls, execution workflow and failure strategies live in `SKILL.md`.
> This file covers **only** conventions and tooling not present in `SKILL.md`. Do not duplicate content.

## Project Context

**Language**: C · **Target**: Espressif ESP32 family (ESP32 / S2 / S3 / C2 / C3 / C5 / C6 / C61 / H2 / P4) · **Framework**: ESP-IDF v5.0+ (CMake) · **Protocol library**: esp-modbus component v2.x.x (instance-based, handle-oriented).

## Code Generation Conventions

### File Naming

- Application main: `main/<role>_<transport>.c` (e.g. `serial_master.c`, `tcp_slave.c`) with a matching `main/CMakeLists.txt`.
- Shared parameter storage component: `mb_example_common/` (mirrors `examples/mb_example_common/`), exposing `modbus_params.h` / `modbus_params.c`.
- Per-example Kconfig: `main/Kconfig.projbuild` (defines `CONFIG_MB_UART_PORT_NUM`, `CONFIG_MB_UART_TXD/RXD/RTS`, `CONFIG_MB_UART_BAUD_RATE`, `CONFIG_MB_COMM_MODE_RTU` / `CONFIG_MB_COMM_MODE_ASCII`, `CONFIG_MB_SLAVE_ADDR`).
- Build defaults: `sdkconfig.defaults` (serial) or `sdkconfig.ci.*` variants for CI matrix.

### Include Pattern

```c
// Both master and slave, all transports
#include "mbcontroller.h"        // pulls in esp_modbus_common.h + esp_modbus_master.h + esp_modbus_slave.h

// For shared parameter structs (optional, used by the examples)
#include "modbus_params.h"       // holding_reg_params_t, input_reg_params_t, coil_reg_params_t, discrete_reg_params_t

// Extended endianness helpers — only when CONFIG_FMB_EXT_TYPE_SUPPORT=y
// (mbcontroller.h includes it automatically via esp_modbus_common.h when the flag is set;
//  you do not need to include mb_endianness_utils.h directly)

// Standard ESP-IDF helpers always used by the examples
#include "esp_err.h"
#include "esp_log.h"
#include "sdkconfig.h"
#include "driver/uart.h"         // uart_set_pin, uart_set_mode, UART_MODE_RS485_HALF_DUPLEX
```

### Standard Serial Project Structure

```
MyModbusSerial/
├── CMakeLists.txt              # project file; add set(EXCLUDE_COMPONENTS freemodbus) on old IDF
├── main/
│   ├── CMakeLists.txt
│   ├── Kconfig.projbuild       # CONFIG_MB_UART_*, CONFIG_MB_COMM_MODE_*, CONFIG_MB_SLAVE_ADDR
│   ├── idf_component.yml       # depends: espressif/esp-modbus: "^2.1.2"
│   ├── sdkconfig.defaults
│   └── serial_master.c         # app_main + master_init + master_operation_func
├── mb_example_common/          # optional shared component (structs + extern instances)
│   ├── CMakeLists.txt
│   ├── include/modbus_params.h
│   └── modbus_params.c
└── sdkconfig                   # generated
```

### Standard TCP Project Structure

Same as serial but `main/` contains `tcp_master.c` / `tcp_slave.c`, no UART Kconfig, and `app_main` first brings up the network with `nvs_flash_init` → `esp_netif_init` → `esp_event_loop_create_default` → `example_connect()` (Wi-Fi or Ethernet), then creates the Modbus TCP object with `.tcp_opts.ip_netif_ptr = get_example_netif()`.

### Canonical Init Patterns

**Serial master** (excerpt mirroring `examples/serial/mb_serial_master/main/serial_master.c::master_init`):

```c
static void *master_handle = NULL;

static esp_err_t master_init(void) {
    mb_communication_info_t comm = {
        .ser_opts.port = MB_PORT_NUM,
#if CONFIG_MB_COMM_MODE_ASCII
        .ser_opts.mode = MB_ASCII,
#elif CONFIG_MB_COMM_MODE_RTU
        .ser_opts.mode = MB_RTU,
#endif
        .ser_opts.baudrate = MB_DEV_SPEED,
        .ser_opts.parity = MB_PARITY_NONE,
        .ser_opts.uid = 0,                       // unused for master
        .ser_opts.response_tout_ms = 1000,
        .ser_opts.data_bits = UART_DATA_8_BITS,
        .ser_opts.stop_bits = UART_STOP_BITS_1,
    };
    ESP_ERROR_CHECK(mbc_master_create_serial(&comm, &master_handle));

    ESP_ERROR_CHECK(mbc_master_set_descriptor(master_handle,
                                              &device_parameters[0],
                                              num_device_parameters));

    ESP_ERROR_CHECK(uart_set_pin(MB_PORT_NUM, CONFIG_MB_UART_TXD, CONFIG_MB_UART_RXD,
                                 CONFIG_MB_UART_RTS, UART_PIN_NO_CHANGE));
    ESP_ERROR_CHECK(uart_set_mode(MB_PORT_NUM, UART_MODE_RS485_HALF_DUPLEX));

    return mbc_master_start(master_handle);
}
```

**Serial slave** (excerpt mirroring `examples/serial/mb_serial_slave/main/serial_slave.c::app_main`):

```c
static void *mbc_slave_handle = NULL;

mb_communication_info_t comm_config = {
    .ser_opts.port = MB_PORT_NUM,
    .ser_opts.mode = MB_RTU,                     // or MB_ASCII
    .ser_opts.baudrate = MB_DEV_SPEED,
    .ser_opts.parity = MB_PARITY_NONE,
    .ser_opts.uid = MB_SLAVE_ADDR,
    .ser_opts.data_bits = UART_DATA_8_BITS,
    .ser_opts.stop_bits = UART_STOP_BITS_1,
};
ESP_ERROR_CHECK(mbc_slave_create_serial(&comm_config, &mbc_slave_handle));

// register one mb_register_area_descriptor_t per type/area...
mb_register_area_descriptor_t reg_area = { .type = MB_PARAM_HOLDING,
                                           .start_offset = 0,
                                           .address = &holding_reg_params.holding_data0,
                                           .size = sizeof(float) * 4,
                                           .access = MB_ACCESS_RW };
ESP_ERROR_CHECK(mbc_slave_set_descriptor(mbc_slave_handle, reg_area));

ESP_ERROR_CHECK(uart_set_pin(MB_PORT_NUM, CONFIG_MB_UART_TXD, CONFIG_MB_UART_RXD,
                             CONFIG_MB_UART_RTS, UART_PIN_NO_CHANGE));
ESP_ERROR_CHECK(uart_set_mode(MB_PORT_NUM, UART_MODE_RS485_HALF_DUPLEX));

mbc_slave_start(mbc_slave_handle);
```

**TCP master** (excerpt mirroring `examples/tcp/mb_tcp_master/main/tcp_master.c::master_init`):

```c
mb_communication_info_t tcp_master_config = {
    .tcp_opts.port = CONFIG_FMB_TCP_PORT_DEFAULT,
    .tcp_opts.mode = MB_TCP,
    .tcp_opts.addr_type = MB_IPV4,               // or MB_IPV6
    .tcp_opts.ip_addr_table = (void *)slave_ip_address_table,
    .tcp_opts.uid = 0,
    .tcp_opts.start_disconnected = false,
    .tcp_opts.response_tout_ms = CONFIG_FMB_MASTER_TIMEOUT_MS_RESPOND,
    .tcp_opts.ip_netif_ptr = (void *)get_example_netif(),
};
mbc_master_create_tcp(&tcp_master_config, &master_handle);
mbc_master_set_descriptor(master_handle, &device_parameters[0], num_device_parameters);
mbc_master_start(master_handle);
```

### Common Macros (from the examples — copy verbatim)

```c
// Offset helpers. NOTE the +1 on master side, >>1 on slave side:
// Master (examples/serial/mb_serial_master/main/serial_master.c):
#define HOLD_OFFSET(field)  ((uint16_t)(offsetof(holding_reg_params_t, field) + 1))
#define INPUT_OFFSET(field) ((uint16_t)(offsetof(input_reg_params_t,   field) + 1))
#define COIL_OFFSET(field)  ((uint16_t)(offsetof(coil_reg_params_t,    field) + 1))
#define DISCR_OFFSET(field) ((uint16_t)(offsetof(discrete_reg_params_t,field) + 1))
#define TEST_HOLD_REG_START(field) (HOLD_OFFSET(field) >> 1)
#define TEST_HOLD_REG_SIZE(field)  (sizeof(((holding_reg_params_t *)0)->field) >> 1)
#define STR(fieldname) ((const char *)(fieldname))
#define OPTS(min_val, max_val, step_val) { .opt1 = min_val, .opt2 = max_val, .opt3 = step_val }

// Slave (examples/serial/mb_serial_slave/main/serial_slave.c):
#define HOLD_OFFSET(field) ((uint16_t)(offsetof(holding_reg_params_t, field) >> 1))
#define INPUT_OFFSET(field)((uint16_t)(offsetof(input_reg_params_t,   field) >> 1))
```

### Logging Convention

```c
static const char *TAG = "SLAVE_TEST";   // or "MASTER_TEST"
ESP_LOGI(TAG, "Modbus slave stack initialized.");
ESP_LOGE(TAG, "Characteristic #%d read fail, err = 0x%x (%s).",
         (int)cid, (int)err, (char *)esp_err_to_name(err));
```

The examples deliberately raise verbosity for the stack's own tags and silence the noisy `vfs_calls` tag at debug level:

```c
#if !CONFIG_LOG_DEFAULT_LEVEL_DEBUG
    esp_log_level_set("mbc_serial.slave", ESP_LOG_DEBUG);
    esp_log_level_set("mb_object.slave", ESP_LOG_DEBUG);
#else
    esp_log_level_set("vfs_calls", ESP_LOG_NONE);
#endif
```

## Build Workflow

1. `idf.py set-target <chip>` (e.g. `esp32`, `esp32s3`, `esp32c3`).
2. `idf.py menuconfig` → enable the right `CONFIG_FMB_COMM_MODE_*_EN`, optionally `CONFIG_FMB_EXT_TYPE_SUPPORT`, set `CONFIG_FMB_MASTER_TIMEOUT_MS_RESPOND`; for examples also set `CONFIG_MB_UART_*` pins / baud / mode and `CONFIG_MB_SLAVE_ADDR`.
3. `idf.py build`.
4. `idf.py -p <PORT> flash monitor`.
5. For TCP examples, also configure Wi-Fi/Ethernet (`Example Connection Configuration`) and pick the slave-IP source (`CONFIG_MB_SLAVE_IP_FROM_STDIN` vs `CONFIG_MB_MDNS_IP_RESOLVER`).
6. On older ESP-IDF that still ships freemodbus: add `set(EXCLUDE_COMPONENTS freemodbus)` to the top-level `CMakeLists.txt` before `project()`.

## esp-modbus Code Generation Checklist

- [ ] `mbc_master_create_*` / `mbc_slave_create_*` return value is checked; the `void *handle` is stored and passed as first arg to every later call
- [ ] Master: `mbc_master_set_descriptor()` called **before** `mbc_master_start()`
- [ ] Slave: at least one `mbc_slave_set_descriptor()` per register type the master will access
- [ ] Slave area `size` field is in **bytes**; master `mb_size` is in **registers** (Coil/Discrete in bits)
- [ ] `mb_param_request_t` for FC 0x17 fills `wr_rd_multi_reg_func.wr_reg_start` / `wr_reg_size`
- [ ] Serial: `uart_set_pin()` + `uart_set_mode(UART_MODE_RS485_HALF_DUPLEX)` called by the app
- [ ] Serial: master and slave baud/parity/data-bits/stop-bits match; mode (RTU/ASCII) matches
- [ ] TCP master: `slave_ip_address_table[]` ends with `NULL`; `.tcp_opts.ip_netif_ptr` non-NULL after `example_connect()`
- [ ] TCP master: `.tcp_opts.response_tout_ms` (or `CONFIG_FMB_MASTER_TIMEOUT_MS_RESPOND`) > worst-case slave RTT
- [ ] Extended types (`PARAM_TYPE_*_ABCD` …) only used when `CONFIG_FMB_EXT_TYPE_SUPPORT=y`
- [ ] Custom handlers registered via `mbc_set_handler()` are short and non-blocking
- [ ] Direct access to mapped slave storage wrapped in `mbc_slave_lock()` / `mbc_slave_unlock()`
- [ ] Teardown calls `mbc_master_delete()` / `mbc_slave_delete()`; TCP also tears down netif/services

## Do Not Modify

- The esp-modbus library source under `espressif-repos/esp-modbus/modbus/` — treat it as a read-only component; configure behaviour only through Kconfig and the documented `mbc_*` API.
- `SKILL.md` frontmatter (skill metadata).
