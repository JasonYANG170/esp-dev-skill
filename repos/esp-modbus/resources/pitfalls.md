# esp-modbus Consolidated Pitfalls

This is the single place to look up the most common esp-modbus mistakes and their fixes. Each entry references the source of truth in the repository.

## 1. v2 API is handle-based; the handle is the first argument

Every `mbc_master_*` / `mbc_slave_*` call takes the `void *handle` returned by `mbc_master_create_serial` / `mbc_master_create_tcp` / `mbc_slave_create_serial` / `mbc_slave_create_tcp` as its first argument. The old no-argument single-instance API does not exist in v2.
Source: `modbus/mb_controller/common/include/esp_modbus_master.h`, `esp_modbus_slave.h`.

## 2. Master: set the descriptor before start

`mbc_master_set_descriptor()` must be called before `mbc_master_start()`. Calling start first leaves the controller with no Data Dictionary and parameter reads return `ESP_ERR_INVALID_STATE`.
Source: `examples/serial/mb_serial_master/main/serial_master.c::master_init`.

## 3. The app configures UART pins + RS485 half-duplex, not the library

esp-modbus does not touch UART pins or the RS485 direction mode. The application must call `uart_set_pin(MB_PORT_NUM, TXD, RXD, RTS, UART_PIN_NO_CHANGE)` and `uart_set_mode(MB_PORT_NUM, UART_MODE_RS485_HALF_DUPLEX)` itself. Omitting this produces garbage on the bus.
Source: `examples/serial/mb_serial_slave/main/serial_slave.c::app_main`, `examples/serial/mb_serial_master/main/serial_master.c::master_init`.

## 4. Slave: register at least one area per register type you serve

A master access to a Holding/Input/Coil/Discrete region with no matching `mbc_slave_set_descriptor()` returns a Modbus exception (`0x02` Illegal Data Address). Register one `mb_register_area_descriptor_t` per type/area. `size` is in **bytes**.
Source: `docs/en/slave_api_overview.rst`, `examples/serial/mb_serial_slave/main/serial_slave.c`.

## 5. Data Dictionary: `mb_size` = registers, `param_size` = bytes

In `mb_parameter_descriptor_t`, `mb_size` is the count of Modbus registers (2 bytes each) for Holding/Input, or bits for Coil/Discrete. `param_size` is the storage size in bytes. For a `float`: `mb_size = 2`, `param_size = 4`. Mixing them up yields corrupt values.
Source: `docs/en/master_api_overview.rst` Table 1, `esp_modbus_master.h`.

## 6. Extended types need `CONFIG_FMB_EXT_TYPE_SUPPORT=y`

The `PARAM_TYPE_*_ABCD/CDAB/BADC/DCBA` family, the 64-bit types, and all `mb_set_*`/`mb_get_*` endianness helpers only exist when this Kconfig flag is on. Without it, builds fail with undeclared identifiers.
Source: `Kconfig`, `modbus/mb_controller/common/include/mb_endianness_utils.h` (guarded by `CONFIG_FMB_EXT_TYPE_SUPPORT` via `esp_modbus_common.h`).

## 7. Master response timeout must beat slave RTT

If the slave's processing time exceeds the master's `CONFIG_FMB_MASTER_TIMEOUT_MS_RESPOND` (or per-instance `response_tout_ms`), the master times out the in-flight request and the slave logs:
`W ... mb_port.tcp.slave: ... handling time [ms]: NNNN, exceeds slave response time in master.`
Raise the master timeout above the worst-case round-trip time.
Source: `docs/en/overview_messaging_and_mapping.rst` ("The Race Condition").

## 8. Guard shared slave storage with lock/unlock

When the stack is active, direct application reads/writes of mapped register storage must be wrapped in `mbc_slave_lock()` / `mbc_slave_unlock()`. For storage shared across multiple slave objects use a FreeRTOS critical section (`portENTER_CRITICAL` / `portEXIT_CRITICAL`).
Source: `docs/en/slave_api_overview.rst`, `examples/tcp/mb_tcp_slave/main/tcp_slave.c`.

## 9. Custom handlers must be short and non-blocking

`mbc_set_handler()` callbacks execute in the Modbus controller event task. A blocking or heavy handler makes the slave miss the master's response window and triggers the race condition in #7. No `vTaskDelay`, no large hex dumps.
Source: `docs/en/master_api_overview.rst`, `docs/en/slave_api_overview.rst` (notes under Customize Function Handlers).

## 10. TCP master: slave IP table must end with NULL

`slave_ip_address_table[]` is walked until a `NULL` entry. Forgetting the terminator reads past the array. Each row is `UID;host_or_ip;port` (or `UID:ipv6:port`).
Source: `examples/tcp/mb_tcp_master/main/tcp_master.c`, `docs/en/port_initialization.rst`.

## 11. CID and `param_key` must be unique

Duplicate CIDs make `mbc_master_get_cid_info` return the wrong descriptor. Use an `enum` for CIDs and keep `param_key` strings unique (prefix similar parameters).
Source: `docs/en/master_api_overview.rst` note, `examples/serial/mb_serial_master/main/serial_master.c`.

## 12. FC 0x17 (Read/Write Multiple) needs the wr_rd sub-fields

For command `0x17`, fill `mb_param_request_t.wr_rd_multi_reg_func.wr_reg_start` and `wr_reg_size` (in registers), or the write half of the transaction is lost.
Source: `esp_modbus_master.h` (`mb_param_request_t`), `docs/en/master_api_overview.rst` example.

## 13. Old ESP-IDF: exclude the bundled freemodbus component

ESP-IDF releases that still ship `freemodbus` will shadow esp-modbus v2 and produce the wrong API / link errors. Add `set(EXCLUDE_COMPONENTS freemodbus)` to the top-level `CMakeLists.txt`. Not needed on ESP-IDF v5.x+.
Source: `README.md`.

## 14. TCP: bring up netif before `create_tcp`

`mbc_master_create_tcp` / `mbc_slave_create_tcp` need `.tcp_opts.ip_netif_ptr` to be a valid netif. Call `example_connect()` first and pass `get_example_netif()`.
Source: `examples/tcp/mb_tcp_master/main/tcp_master.c::app_main`, `examples/tcp/mb_tcp_slave/main/tcp_slave.c::init_services`.

## 15. Diagnose by error code

| esp_err_t | Hex | Meaning | First action |
|---|---|---|---|
| `ESP_ERR_TIMEOUT` | 0x107 | Slave did not respond in time | Raise `CONFIG_FMB_MASTER_TIMEOUT_MS_RESPOND`; check RS485 wiring/RTS, baud/parity match |
| `ESP_ERR_NOT_SUPPORTED` | 0x106 | Slave returned exception (register not mapped) | Check Data Dictionary / slave area descriptors vs register map |
| `ESP_ERR_INVALID_RESPONSE` | 0x108 | Wrong command for type, or bad `mb_size` | Verify FC ↔ `mb_param_type` pairing and register counts |
| `ESP_ERR_INVALID_STATE` | 0x103 | Not started / FSM busy / not connected | Check init order and (TCP) connection state |

Source: `docs/en/applications_and_references.rst` Table 5.
