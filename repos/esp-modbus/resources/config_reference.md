# esp-modbus Configuration Reference

All symbols below are defined in `espressif-repos/esp-modbus/Kconfig` under the menu **Modbus configuration**, set via `idf.py menuconfig` or `sdkconfig.defaults`. Example-level symbols (prefixed `CONFIG_MB_*`) come from each example's `main/Kconfig.projbuild`.

## Communication mode enable

| Symbol | Default | Depends on | Purpose |
|---|---|---|---|
| `CONFIG_FMB_COMM_MODE_RTU_EN` | y | — | Enable serial RTU mode |
| `CONFIG_FMB_COMM_MODE_ASCII_EN` | y | — | Enable serial ASCII mode |
| `CONFIG_FMB_COMM_MODE_TCP_EN` | y | `LWIP_ENABLE` | Enable TCP mode |

## TCP options

| Symbol | Default | Range | Purpose |
|---|---|---|---|
| `CONFIG_FMB_TCP_PORT_DEFAULT` | 502 | 0–65535 | Default Modbus TCP port |
| `CONFIG_FMB_TCP_PORT_MAX_CONN` | 5 | 1–`LWIP_MAX_SOCKETS` | Max simultaneous slave connections |
| `CONFIG_FMB_TCP_CONNECTION_TOUT_SEC` | 2 | 1–7200 | TCP connection (accept) timeout |
| `CONFIG_FMB_TCP_KEEP_ALIVE_TOUT_SEC` | 4 | 1–7200 | Keep-alive probe interval; should exceed master timeout |
| `CONFIG_FMB_TCP_UID_ENABLED` | n | — | Use UID field in MBAP frame |

## Master timing

| Symbol | Default | Range | Purpose |
|---|---|---|---|
| `CONFIG_FMB_MASTER_TIMEOUT_MS_RESPOND` | 10000 | 150–30000 | Master wait for slave response (ms); must be > worst-case RTT |
| `CONFIG_FMB_MASTER_DELAY_MS_CONVERT` | 200 | 150–2000 | Delay after broadcast before next frame (ms) |

## Stack / task sizing

| Symbol | Default | Range | Purpose |
|---|---|---|---|
| `CONFIG_FMB_QUEUE_LENGTH` | 50 | 10–500 | Event task queue length |
| `CONFIG_FMB_PORT_TASK_STACK_SIZE` | 4096 | 2048–16384 | Port rx/tx task stack |
| `CONFIG_FMB_PORT_TASK_PRIO` | 10 | 3–23 | Port task priority (controller task = prio − 1) |
| `CONFIG_FMB_PORT_TASK_AFFINITY` | CPU0 | NO_AFFINITY / CPU0 / CPU1 | Core affinity (`FMB_PORT_TASK_AFFINITY_*`) |
| `CONFIG_FMB_CONTROLLER_STACK_SIZE` | 4096 | 2048–32768 | Modbus controller task stack |
| `CONFIG_FMB_EVENT_QUEUE_TIMEOUT` | 20 | 10–500 | Event queue wait timeout (ms) |
| `CONFIG_FMB_BUFFER_SIZE` | 260 (TCP) / 256 (serial) | 256–2048 | RX/TX buffer size (Modbus max frame = 260 B) |

## ASCII-specific

| Symbol | Default | Range | Purpose |
|---|---|---|---|
| `CONFIG_FMB_SERIAL_ASCII_BITS_PER_SYMB` | 8 | 7–8 | Data bits per ASCII character |
| `CONFIG_FMB_SERIAL_ASCII_TIMEOUT_RESPOND_MS` | 1000 | 200–5000 | Slave response timeout for ASCII mode (ms) |

## Slave ID (FC 0x11)

| Symbol | Default | Range | Purpose |
|---|---|---|---|
| `CONFIG_FMB_CONTROLLER_SLAVE_ID_SUPPORT` | y | — | Enable Report Slave ID command |
| `CONFIG_FMB_CONTROLLER_SLAVE_ID` | 0x00112233 | 0–0xFFFFFFFF | Default slave ID value (MSB = short ID) |
| `CONFIG_FMB_CONTROLLER_SLAVE_ID_MAX_SIZE` | 32 | 4–255 | Max slave ID buffer size (bytes) |

## Notifications & dispatch

| Symbol | Default | Range | Purpose |
|---|---|---|---|
| `CONFIG_FMB_CONTROLLER_NOTIFY_TIMEOUT` | 20 | 0–200 | Notification send timeout (ms) |
| `CONFIG_FMB_CONTROLLER_NOTIFY_QUEUE_SIZE` | 20 | 0–200 | Slave parameter-notification queue size |
| `CONFIG_FMB_TIMER_USE_ISR_DISPATCH_METHOD` | n | — | ISR dispatch for timer (selects `ESP_TIMER_SUPPORTS_ISR_DISPATCH_METHOD` + `UART_ISR_IN_IRAM`) |

## Types & handlers

| Symbol | Default | Range | Purpose |
|---|---|---|---|
| `CONFIG_FMB_EXT_TYPE_SUPPORT` | n | — | Enable extended types + `mb_set_*`/`mb_get_*` endianness API |
| `CONFIG_FMB_FUNC_HANDLERS_MAX` | 16 | 16–255 | Max function handlers per object |

## Integration / build

| Symbol | Default | Purpose |
|---|---|---|
| `CONFIG_FMB_MDNS_INTEGRATION_ENABLE` | y | Resolve MDNS names in the TCP master slave IP table |
| `CONFIG_FMB_COMPILER_STATIC_ANALYZER_ENABLE` | n | Enable GCC static analyzer for the library (needs `IDF_TOOLCHAIN_GCC`) |

## Example-level Kconfig (`examples/*/main/Kconfig.projbuild`)

These are defined per-example, not by the library itself:

| Symbol | Default | Purpose |
|---|---|---|
| `CONFIG_MB_UART_PORT_NUM` | 1 or 2 (chip-dependent) | UART port number for Modbus |
| `CONFIG_MB_UART_BAUD_RATE` | 115200 | UART baud (1200–115200) |
| `CONFIG_MB_UART_TXD` / `RXD` / `RTS` | chip-dependent | RS485 UART pins (RTS drives DE/RE) |
| `CONFIG_MB_COMM_MODE_RTU` / `CONFIG_MB_COMM_MODE_ASCII` | RTU | Per-example serial mode choice |
| `CONFIG_MB_SLAVE_ADDR` | 1 | Slave Unit Identifier (slave example) |

## Example `sdkconfig.defaults` (from `examples/serial/mb_serial_slave/sdkconfig.defaults`)

```ini
CONFIG_MB_COMM_MODE_ASCII=n
CONFIG_MB_COMM_MODE_RTU=y
CONFIG_MB_SLAVE_ADDR=1
CONFIG_MB_UART_BAUD_RATE=115200
CONFIG_FMB_TIMER_USE_ISR_DISPATCH_METHOD=y
CONFIG_FMB_MASTER_DELAY_MS_CONVERT=200
CONFIG_FMB_MASTER_TIMEOUT_MS_RESPOND=400
CONFIG_FMB_EXT_TYPE_SUPPORT=y
CONFIG_LOG_DEFAULT_LEVEL_DEBUG=n
CONFIG_LOG_MAXIMUM_LEVEL_DEBUG=y
```

## Excluding the legacy freemodbus component

On ESP-IDF releases that still bundle the old single-instance `freemodbus` component, add to the **top-level** `CMakeLists.txt` (before `project()`):

```cmake
set(EXCLUDE_COMPONENTS freemodbus)
```

ESP-IDF v5.x and later do not ship freemodbus, so this is unnecessary there.
