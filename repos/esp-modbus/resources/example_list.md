# esp-modbus Example Index

Real example directories found under `espressif-repos/esp-modbus/examples/`. Each row lists the path and a one-line description drawn from the example's own README / source.

## Top-level

| Path | Description |
|---|---|
| `examples/README.md` | Index to the four master/slave examples. |
| `examples/mb_example_common/` | Shared component defining the parameter storage structs (`holding_reg_params_t`, `input_reg_params_t`, `coil_reg_params_t`, `discrete_reg_params_t`) and their extern instances; used by all four examples. |

## Serial (RTU / ASCII over RS485)

| Path | Description |
|---|---|
| `examples/serial/mb_serial_slave/` | Modbus serial slave (RTU or ASCII). Registers Holding/Input/Coil/Discrete areas, event loop, custom FC 0x41 echo handler, Report Slave ID. Entry: `main/serial_slave.c`. |
| `examples/serial/mb_serial_master/` | Modbus serial master (RTU or ASCII). Data Dictionary with CIDs, polling loop, custom FC 0x41 request, Report Slave ID read. Entry: `main/serial_master.c`. |
| `examples/serial/pytest_mb_master_slave.py` | Pytest harness running the master and slave examples against each other. |

Per-example config of note:
- `examples/serial/mb_serial_slave/sdkconfig.defaults` — RTU on, `CONFIG_MB_SLAVE_ADDR=1`, `CONFIG_FMB_EXT_TYPE_SUPPORT=y`, ISR dispatch on, master timeout 400 ms.
- `examples/serial/mb_serial_slave/sdkconfig.ci.ascii` / `sdkconfig.ci.rtu` — CI matrix variants for ASCII / RTU.
- `examples/serial/mb_serial_master/main/Kconfig.projbuild` — defines `CONFIG_MB_UART_PORT_NUM`, `CONFIG_MB_UART_TXD/RXD/RTS` (with per-target ranges/pins), `CONFIG_MB_UART_BAUD_RATE`, `CONFIG_MB_COMM_MODE_RTU` / `CONFIG_MB_COMM_MODE_ASCII`.

Report Slave ID (FC 0x11) — the serial slave example sets a vendor device-ID struct via the `INIT_DEV_ID` macro and `mbc_set_slave_id` (guarded by `CONFIG_FMB_CONTROLLER_SLAVE_ID_SUPPORT`, in `main/serial_slave.c`); the serial master example reads it back via `mbc_master_send_request` with `.command = 0x11` (in `main/serial_master.c`). How-to: `recipes/slave_device_id.md` (slave set side) and `recipes/serial_master.md` §6 (master read side).

## TCP (over Wi-Fi / Ethernet)

| Path | Description |
|---|---|
| `examples/tcp/mb_tcp_slave/` | Modbus TCP slave. Wi-Fi/Ethernet via `example_connect()`, multi-connection, keep-alive, register areas, event loop, custom FC 0x41. Entry: `main/tcp_slave.c`. |
| `examples/tcp/mb_tcp_master/` | Modbus TCP master. Slave IP address table (MDNS / static IP), Data Dictionary, polling loop, custom FC 0x41, Report Slave ID. Entry: `main/tcp_master.c`. |
| `examples/tcp/pytest_mb_tcp_master_slave.py` | Pytest: TCP master vs TCP slave. |
| `examples/tcp/pytest_mb_tcp_host_test_master.py` / `pytest_mb_tcp_host_test_slave.py` | Pytest against an external Modbus host test. |

Per-example config of note:
- `examples/tcp/mb_tcp_master/sdkconfig.ci.wifi` / `sdkconfig.ci.ethernet` / `sdkconfig.ci.defaults` — CI variants selecting the network backend and slave-IP source (`CONFIG_MB_SLAVE_IP_FROM_STDIN` vs `CONFIG_MB_MDNS_IP_RESOLVER`).

## Test apps (reference for additional patterns)

| Path | Description |
|---|---|
| `test_apps/adapter_tests/` | Port-layer adapter tests. |
| `test_apps/physical_tests/` | Physical-layer tests over real hardware. |
| `test_apps/tcp_instances_tests/mb_tcp_master_instances/` | Multi-instance TCP master test. |
| `test_apps/tcp_instances_tests/mb_tcp_slave_instances/` | Multi-instance TCP slave test. |
| `test_apps/unit_tests/mb_controller_common/` | Controller common unit tests. |
| `test_apps/unit_tests/mb_controller_mapping/` | Controller data-mapping unit tests. |
| `test_apps/unit_tests/mb_ext_types/` | Extended-types / endianness conversion unit tests. |
| `test_apps/unit_tests/test_config_parser/` | Config parser unit tests. |

> The test apps are not application templates; they exist for CI. Use `examples/serial/*` and `examples/tcp/*` as the starting point for new firmware.
