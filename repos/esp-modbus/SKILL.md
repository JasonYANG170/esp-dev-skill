---
name: esp-modbus-skill
description: >-
  AI Skill for Espressif esp-modbus (Modbus RTU / ASCII / TCP) firmware development on ESP-IDF.
  Used when users need to create, modify, or debug Modbus master/slave applications, configure the
  Data Dictionary and register area descriptors, map CIDs to Modbus registers, register custom
  function-code handlers, or resolve Modbus communication errors on ESP32 family chips.
  Trigger words: "Modbus", "RTU", "ASCII", "Modbus TCP", "RS485", "esp-modbus", "freemodbus",
  "Modbus 主站", "Modbus 从站", "保持寄存器", "输入寄存器", "线圈", "离散输入", "Holding", "Coil",
  "mbc_master", "mbc_slave", "Data Dictionary", "数据字典"
license: Apache-2.0
metadata:
  author: Community
  version: "1.1.0"
---

# esp-modbus-skill

AI Skill for developing firmware with the Espressif **esp-modbus** library (component version v2.x.x). Covers Modbus RTU, ASCII and TCP, the v2 instance-based master/slave API (`mbc_*_create_*` returning a handle), the Data Dictionary (CID → register mapping), slave register-area descriptors, custom function-code handlers, the endianness/extended-types conversion API, and the Kconfig configuration keys. Every API name, struct, macro, and code snippet below is grounded in the repository's own `docs/`, headers under `modbus/mb_controller/common/include/`, and the examples under `examples/`.

## Core Principles

1. **v2 is instance-based** — Every master/slave object is created by `mbc_master_create_serial` / `mbc_master_create_tcp` / `mbc_slave_create_serial` / `mbc_slave_create_tcp`, which fill a `void *handle`. That handle is the **first argument of every later API call** for that object. The legacy single-instance global-handle API is gone.
2. **Never guess APIs** — All public functions live in `mbcontroller.h` → `esp_modbus_master.h` / `esp_modbus_slave.h`. If a function is not there, it does not exist; do not invent it.
3. **Communication mode is compile-time + per-instance** — The serial modes (RTU, ASCII) and TCP must be enabled at compile time (`CONFIG_FMB_COMM_MODE_RTU_EN`, `CONFIG_FMB_COMM_MODE_ASCII_EN`, `CONFIG_FMB_COMM_MODE_TCP_EN`). The per-instance mode is then set in `mb_communication_info_t` (`.ser_opts.mode = MB_RTU` / `MB_ASCII`, or `.tcp_opts.mode = MB_TCP`).
4. **Master needs a Data Dictionary** — The master does not read raw registers; it reads CIDs. A `mb_parameter_descriptor_t device_parameters[]` table must be registered via `mbc_master_set_descriptor()` before `mbc_master_start()`. Each CID maps to a slave UID + register type + start + size + data type.
5. **Slave needs register-area descriptors** — A slave must register at least one `mb_register_area_descriptor_t` per register type it wants to serve (Holding/Input/Coil/Discrete) via `mbc_slave_set_descriptor()`. Unmapped accesses return a Modbus exception.
6. **Initialization order is fixed** — create → (set descriptor / register handlers) → `uart_set_pin` + `uart_set_mode(UART_MODE_RS485_HALF_DUPLEX)` for serial → start. For master, `set_descriptor` must happen before `start`.
7. **RS485 UART setup is the app's job** — esp-modbus does not configure pins or half-duplex mode. The app must call `uart_set_pin()` and `uart_set_mode(MB_PORT_NUM, UART_MODE_RS485_HALF_DUPLEX)` itself (see serial examples).
8. **Master timeout governs reliability** — `CONFIG_FMB_MASTER_TIMEOUT_MS_RESPOND` (or per-instance `ser_opts.response_tout_ms` / `tcp_opts.response_tout_ms`) must be greater than the worst-case slave round-trip time, otherwise the master drops in-flight responses and the slave logs a race-condition warning.
9. **Extended data types need a Kconfig flag** — The big family of `PARAM_TYPE_U32_ABCD`, `PARAM_TYPE_FLOAT_CDAB`, `PARAM_TYPE_DOUBLE_HGFEDCBA`, etc., and the `mb_get_*` / `mb_set_*` endianness helpers, only exist when `CONFIG_FMB_EXT_TYPE_SUPPORT=y`. On the slave side they require `#include "mb_endianness_utils.h"` (pulled in automatically when the flag is set).
10. **Access shared slave data under lock** — When the stack is active, the app must guard direct reads/writes of the mapped register storage with `mbc_slave_lock()`/`mbc_slave_unlock()` (or a FreeRTOS critical section for cross-object sharing).
11. **Custom handlers run in the controller task** — `mbc_set_handler()` callbacks execute in the Modbus controller event task and must be short, non-blocking, and never log heavily; a slow handler makes the slave miss the master's response window.
12. **Destroy on teardown** — `mbc_master_delete()` / `mbc_slave_delete()` stop the stack and free all tasks/queues. For TCP, also tear down netif/services afterwards.

## When to Use

**Applicable:**
- Creating a Modbus RTU/ASCII master or slave over RS485 on an ESP32-family chip
- Creating a Modbus TCP master or slave over Wi-Fi/Ethernet
- Mapping physical parameters (temperature, humidity, relay state) to CIDs via the Data Dictionary
- Implementing vendor-specific (non-standard) Modbus function codes via custom handlers
- Reading/writing 32-bit / 64-bit / float / double values with specific byte order (ABCD/CDAB/…)
- Diagnosing Modbus errors (`ESP_ERR_TIMEOUT` 0x107, `ESP_ERR_NOT_SUPPORTED` 0x106, `ESP_ERR_INVALID_RESPONSE` 0x108, `ESP_ERR_INVALID_STATE` 0x103)
- Migrating from the legacy built-in freemodbus component to esp-modbus v2

**Not applicable:**
- Non-Modbus serial protocols (plain UART, DMX, proprietary)
- Bare-metal (non-ESP-IDF) projects — esp-modbus depends on FreeRTOS, UART and lwIP/esp_netif from ESP-IDF
- Other vendors' Modbus stacks (MicroPython `modbus`, Linux `libmodbus`, etc.)
- PCB / RS485 transceiver schematic design (only the firmware side)

---

## Scenario Quick Reference (Recipes)

When the user's intent matches a scenario below, **read the corresponding recipe first** — it has the full call chain, step-by-step instructions, real code, and a common-errors table.

### Serial (RTU / ASCII over RS485)

| recipe | scenario |
|---|---|
| `recipes/serial_slave.md` | Build a Modbus serial slave (RTU or ASCII): UART + RS485 setup, register-area descriptors, event loop |
| `recipes/serial_master.md` | Build a Modbus serial master: UART + RS485 setup, Data Dictionary, polling loop, teardown |

### TCP (over Wi-Fi / Ethernet)

| recipe | scenario |
|---|---|
| `recipes/tcp_slave.md` | Build a Modbus TCP slave: netif init, `mbc_slave_create_tcp`, multi-connection, keep-alive |
| `recipes/tcp_master.md` | Build a Modbus TCP master: slave IP address table, MDNS resolution, `mbc_master_create_tcp` |

### Data model & advanced

| recipe | scenario |
|---|---|
| `recipes/data_dictionary.md` | Author the master Data Dictionary (`mb_parameter_descriptor_t`), CID enum, `OPTS`/`STR` macros, offset macros |
| `recipes/slave_register_areas.md` | Map slave storage structs to Modbus areas via `mbc_slave_set_descriptor` + `HOLD_OFFSET` macros |
| `recipes/custom_handlers.md` | Register/override function-code handlers with `mbc_set_handler` / `mbc_get_handler` / `mbc_delete_handler`, including the FC 0x41 echo example |
| `recipes/extended_types.md` | Use `CONFIG_FMB_EXT_TYPE_SUPPORT`, the `PARAM_TYPE_*_ABCD` family, and `mb_set_float_abcd` / `mb_get_uint32_dcba` helpers |
| `recipes/slave_device_id.md` | Set vendor-specific slave device identification (short UID + running status + vendor data) via `mbc_set_slave_id` / `mbc_get_slave_id` for retrieval by master FC 0x11 Report Slave ID |

---

## Modbus Register Types & Data Areas

| Modbus area | `mb_param_type_t` | Function codes (master) | Direction | Size unit |
|---|---|---|---|---|
| Holding Registers | `MB_PARAM_HOLDING` | 0x03 Read, 0x06 Write Single, 0x10 Write Multiple, 0x17 R/W Multiple | read/write | 16-bit register |
| Input Registers | `MB_PARAM_INPUT` | 0x04 Read Input | read-only | 16-bit register |
| Coils | `MB_PARAM_COIL` | 0x01 Read, 0x05 Write Single, 0x0F Write Multiple | read/write | 1 bit |
| Discrete Inputs | `MB_PARAM_DISCRETE` | 0x02 Read Discrete | read-only | 1 bit |
| Custom | `MB_PARAM_CUSTOM` | vendor code (e.g. 0x41) | n/a | n/a |

> The master's `mb_size` field is in **registers (2 bytes)** for Holding/Input, and in **bits** for Coil/Discrete.

## Communication Mode / Config Structure Selection

| Mode | Compile flag | `mode` value | Config union member | Create call |
|---|---|---|---|---|
| Serial RTU | `CONFIG_FMB_COMM_MODE_RTU_EN` | `MB_RTU` | `mb_communication_info_t::ser_opts` | `mbc_master_create_serial` / `mbc_slave_create_serial` |
| Serial ASCII | `CONFIG_FMB_COMM_MODE_ASCII_EN` | `MB_ASCII` | `…::ser_opts` | `mbc_master_create_serial` / `mbc_slave_create_serial` |
| TCP/IP | `CONFIG_FMB_COMM_MODE_TCP_EN` (needs `LWIP_ENABLE`) | `MB_TCP` | `…::tcp_opts` | `mbc_master_create_tcp` / `mbc_slave_create_tcp` |

## Key Kconfig Quick Reference

| Symbol | Default | Purpose |
|---|---|---|
| `CONFIG_FMB_COMM_MODE_RTU_EN` | y | Enable RTU mode |
| `CONFIG_FMB_COMM_MODE_ASCII_EN` | y | Enable ASCII mode |
| `CONFIG_FMB_COMM_MODE_TCP_EN` | y (needs LWIP) | Enable TCP mode |
| `CONFIG_FMB_TCP_PORT_DEFAULT` | 502 | Default Modbus TCP port |
| `CONFIG_FMB_TCP_PORT_MAX_CONN` | 5 | Max simultaneous TCP slave connections |
| `CONFIG_FMB_TCP_CONNECTION_TOUT_SEC` | 2 | TCP connection (accept) timeout |
| `CONFIG_FMB_TCP_KEEP_ALIVE_TOUT_SEC` | 4 | TCP keep-alive probe interval |
| `CONFIG_FMB_MASTER_TIMEOUT_MS_RESPOND` | 10000 | Master wait for slave response (ms) |
| `CONFIG_FMB_MASTER_DELAY_MS_CONVERT` | 200 | Master broadcast convert delay (ms) |
| `CONFIG_FMB_QUEUE_LENGTH` | 50 | Event task queue length |
| `CONFIG_FMB_PORT_TASK_STACK_SIZE` | 4096 | Port rx/tx task stack |
| `CONFIG_FMB_PORT_TASK_PRIO` | 10 | Port task priority (controller = prio − 1) |
| `CONFIG_FMB_PORT_TASK_AFFINITY` | CPU0 | Core affinity (NO_AFFINITY / CPU0 / CPU1) |
| `CONFIG_FMB_BUFFER_SIZE` | 260 (TCP) / 256 (serial) | RX/TX buffer size |
| `CONFIG_FMB_CONTROLLER_SLAVE_ID_SUPPORT` | y | Enable Report Slave ID (FC 0x11) |
| `CONFIG_FMB_CONTROLLER_SLAVE_ID` | 0x00112233 | Default slave ID value |
| `CONFIG_FMB_CONTROLLER_NOTIFY_QUEUE_SIZE` | 20 | Slave parameter-notification queue size |
| `CONFIG_FMB_EXT_TYPE_SUPPORT` | n | Enable extended types + endianness API |
| `CONFIG_FMB_FUNC_HANDLERS_MAX` | 16 | Max registered function handlers per object |
| `CONFIG_FMB_TIMER_USE_ISR_DISPATCH_METHOD` | n | ISR dispatch for timer (selects `UART_ISR_IN_IRAM`) |
| `CONFIG_FMB_MDNS_INTEGRATION_ENABLE` | y | Resolve MDNS names in TCP master slave table |

## Modbus Error Code → esp_err_t Mapping

| Modbus error | esp_err_t | Hex | Meaning |
|---|---|---|---|
| `MB_ENOERR` | `ESP_OK` | 0x0 | Success |
| `MB_ENOREG` | `ESP_ERR_NOT_SUPPORTED` | 0x106 | Register not supported / exception from slave |
| `MB_ETIMEDOUT` | `ESP_ERR_TIMEOUT` | 0x107 | Slave did not respond in time |
| `MB_EILLFUNC` / `MB_ERECVDATA` | `ESP_ERR_INVALID_RESPONSE` | 0x108 | Unsupported/incorrect response from slave |
| `MB_EBUSY` / `MB_EILLSTATE` / `MB_ENOCONN` | `ESP_ERR_INVALID_STATE` | 0x103 | FSM busy / not connected / critical failure |

---

## Critical Pitfalls (Must Read)

These are the most common errors. Violating any of these produces non-working Modbus firmware.

### 1. Master handle is the first parameter of every call

```c
// ❌ WRONG — calling the v1-style no-argument API (does not exist in v2)
mbc_master_start();
mbc_master_get_parameter(CID_TEMP, temp_data, &type);

// ✅ CORRECT — pass the handle returned by mbc_master_create_serial/tcp
static void *master_handle = NULL;
mbc_master_create_serial(&comm, &master_handle);
mbc_master_start(master_handle);
mbc_master_get_parameter(master_handle, CID_TEMP, temp_data, &type);
```

### 2. Master: set the descriptor BEFORE start

```c
// ❌ WRONG — start before descriptor, reads return ESP_ERR_INVALID_STATE
mbc_master_start(master_handle);
mbc_master_set_descriptor(master_handle, device_parameters, num_device_parameters);

// ✅ CORRECT — descriptor first, then start
mbc_master_set_descriptor(master_handle, device_parameters, num_device_parameters);
uart_set_mode(MB_PORT_NUM, UART_MODE_RS485_HALF_DUPLEX);
mbc_master_start(master_handle);
```

### 3. Serial RS485: the app must set pins and half-duplex mode

```c
// ❌ WRONG — relying on esp-modbus to configure the UART pins/mode (it does not)
mbc_slave_create_serial(&comm, &slave_handle);
mbc_slave_start(slave_handle);   // no RS485 direction control -> garbage on the line

// ✅ CORRECT — app sets pins and RS485 half-duplex mode itself
mbc_slave_create_serial(&comm, &slave_handle);
ESP_ERROR_CHECK(uart_set_pin(MB_PORT_NUM, CONFIG_MB_UART_TXD,
                             CONFIG_MB_UART_RXD, CONFIG_MB_UART_RTS,
                             UART_PIN_NO_CHANGE));
ESP_ERROR_CHECK(uart_set_mode(MB_PORT_NUM, UART_MODE_RS485_HALF_DUPLEX));
mbc_slave_start(slave_handle);
```

### 4. Slave: register at least one area per type you want to serve

```c
// ❌ WRONG — master reads Holding but slave never called mbc_slave_set_descriptor for HOLDING
//           -> slave returns Modbus exception 0x02 (Illegal Data Address)

// ✅ CORRECT — register the area with correct byte size
mb_register_area_descriptor_t reg_area = {0};
reg_area.type = MB_PARAM_HOLDING;
reg_area.start_offset = 0;
reg_area.address = &holding_reg_params.holding_data0;
reg_area.size = sizeof(float) * 4;          // size is in BYTES
reg_area.access = MB_ACCESS_RW;
ESP_ERROR_CHECK(mbc_slave_set_descriptor(slave_handle, reg_area));
```

### 5. Data Dictionary `mb_size` is registers (2 bytes), `param_size` is bytes

```c
// ❌ WRONG — confusing register count with byte count
{ CID_TEMP, STR("Temp"), STR("C"), 1, MB_PARAM_HOLDING,
  0, 4 /*bytes?*/, 0, PARAM_TYPE_FLOAT, 4, OPTS(0,100,0), PAR_PERMS_READ },

// ✅ CORRECT — mb_size = registers (float = 2 registers), param_size = bytes (float = 4 bytes)
{ CID_TEMP, STR("Temp"), STR("C"), 1, MB_PARAM_HOLDING,
  0, 2 /*registers*/, 0, PARAM_TYPE_FLOAT, 4 /*bytes*/, OPTS(0,100,0),
  PAR_PERMS_READ_WRITE_TRIGGER },
```

### 6. Extended types require CONFIG_FMB_EXT_TYPE_SUPPORT=y

```c
// ❌ WRONG — using PARAM_TYPE_FLOAT_CDAB without enabling extended types
//   builds fail: 'PARAM_TYPE_FLOAT_CDAB' undeclared, mb_set_float_cdab missing
{ CID_F, STR("F"), STR("--"), 1, MB_PARAM_HOLDING, 0, 2,
  0, PARAM_TYPE_FLOAT_CDAB, 4, OPTS(0,0,0), PAR_PERMS_READ_WRITE_TRIGGER };

// ✅ CORRECT — enable in sdkconfig / menuconfig first
//   CONFIG_FMB_EXT_TYPE_SUPPORT=y
// then the PARAM_TYPE_*_ABCD/CDAB/... enumerators and mb_set_*/mb_get_* exist
```

### 7. Master response timeout must beat the slave's round-trip time

```c
// ❌ WRONG — 1000 ms timeout while the slave takes 1400 ms to answer
//   slave logs: "handling time [ms]: 1394, exceeds slave response time in master."
//   master reports ESP_ERR_TIMEOUT (0x107) for the first request
.ser_opts.response_tout_ms = 1000;

// ✅ CORRECT — timeout > worst-case RTT; raise CONFIG_FMB_MASTER_TIMEOUT_MS_RESPOND
//   or set per-instance response_tout_ms
.ser_opts.response_tout_ms = 0;     // 0 = use CONFIG_FMB_MASTER_TIMEOUT_MS_RESPOND (>= 1000 recommended)
// and in menuconfig: CONFIG_FMB_MASTER_TIMEOUT_MS_RESPOND=5000
```

### 8. Guard shared slave storage with lock/unlock

```c
// ❌ WRONG — modifying mapped storage while the stack may read/write it -> data race
holding_reg_params.holding_data0 += 1.0f;

// ✅ CORRECT — lock around the access while the stack is active
mbc_slave_lock(slave_handle);
holding_reg_params.holding_data0 += 1.0f;
mbc_slave_unlock(slave_handle);
```

### 9. Custom handler must stay short and non-blocking

```c
// ❌ WRONG — blocking/logging-heavy handler in the controller task
mb_exception_t my_handler(void *inst, uint8_t *frame, uint16_t *len) {
    vTaskDelay(pdMS_TO_TICKS(50));        // blocks the whole stack
    ESP_LOG_BUFFER_HEXDUMP("HUGE", buf, 4096, ESP_LOG_INFO);  // too verbose
    return MB_EX_NONE;
}

// ✅ CORRECT — minimal work, return quickly
mb_exception_t my_handler(void *inst, uint8_t *frame, uint16_t *len) {
    MB_RETURN_ON_FALSE((frame && len && *len < (MB_CUST_DATA_LEN - 1)),
                       MB_EX_ILLEGAL_DATA_VALUE, TAG, "bad frame");
    strncpy(my_buf, (char *)&frame[1], MB_CUST_DATA_LEN);
    return MB_EX_NONE;
}
```

### 10. TCP master slave table must end with NULL

```c
// ❌ WRONG — no NULL terminator, master reads past the table -> crash/garbage
char *slave_ip_address_table[] = {
    "01;192.168.1.5;502",
    "02;192.168.1.6;502",
};

// ✅ CORRECT — last entry is NULL
char *slave_ip_address_table[] = {
    "01;mb_slave_tcp_01;502",     // UID;host(or IP);port
    "02;192.168.1.6;1502",
    NULL
};
```

### 11. CID and param_key must be unique

```c
// ❌ WRONG — two entries share CID 0; mbc_master_get_cid_info returns the wrong one
{ 0, STR("Temp"),   ... }, { 0, STR("Humid"), ... },

// ✅ CORRECT — unique CID and unique param_key; use an enum
enum { CID_TEMP = 0, CID_HUMID, CID_COUNT };
const mb_parameter_descriptor_t device_parameters[] = {
    { CID_TEMP, STR("Temperature"), ... },
    { CID_HUMID, STR("Humidity"),   ... },
};
```

### 12. FC 0x17 (Read/Write Multiple) needs the wr_rd_multi_reg_func sub-fields

```c
// ❌ WRONG — leaving write fields zero for command 0x17, the write part is lost
mb_param_request_t req = { .slave_addr = 1, .command = 0x17,
                           .reg_start = HOLD_REG_START(holding_data2),
                           .reg_size = 2 };

// ✅ CORRECT — fill wr_reg_start / wr_reg_size (in registers)
mb_param_request_t req = {
    .slave_addr = 1,
    .command = 0x17,
    .reg_start = HOLD_REG_START(holding_data2),     // read start
    .reg_size  = 2,                                  // read length (registers)
    .wr_rd_multi_reg_func.wr_reg_start = HOLD_REG_START(holding_data0), // write start
    .wr_rd_multi_reg_func.wr_reg_size  = 6,          // write length (registers)
};
mbc_master_send_request(master_handle, &req, &read_write_buf);
```

### 13. ESP-IDF releases with built-in freemodbus must exclude it

```cmake
# ❌ WRONG on older ESP-IDF: link error / wrong API, because the built-in
#    freemodbus component (old single-instance API) shadows esp-modbus v2
# (no EXCLUDE_COMPONENTS line)

# ✅ CORRECT — in the project CMakeLists.txt, before project()
set(EXCLUDE_COMPONENTS freemodbus)
```

### 14. TCP master: netif must exist before create_tcp

```c
// ❌ WRONG — calling mbc_master_create_tcp before network is up; the
//    .tcp_opts.ip_netif_ptr is NULL -> connection failures
mbc_master_create_tcp(&cfg, &master_handle);   // ip_netif_ptr = NULL
example_connect();

// ✅ CORRECT — bring up netif first, then pass get_example_netif()
example_connect();
mb_communication_info_t cfg = {
    .tcp_opts.port = 502,
    .tcp_opts.mode = MB_TCP,
    .tcp_opts.addr_type = MB_IPV4,
    .tcp_opts.ip_addr_table = (void *)slave_ip_address_table,
    .tcp_opts.ip_netif_ptr = (void *)get_example_netif(),
    ...
};
mbc_master_create_tcp(&cfg, &master_handle);
```

---

## Execution Workflow

| Step | Name | Description |
|------|------|-------------|
| 1 | Plan | Identify role (master/slave), transport (serial RTU/ASCII or TCP), target chip, register map |
| 2 | Recipe | Read the matching recipe in `recipes/` and follow its call chain end-to-end |
| 3 | Config | Enable the right `CONFIG_FMB_COMM_MODE_*_EN` flags; set `CONFIG_FMB_EXT_TYPE_SUPPORT=y` if extended types are needed; pick UART pins / netif |
| 4 | Author | Write `app_main`: build `mb_communication_info_t`, create handle, (master) set descriptor / (slave) set area descriptors, register custom handlers if any, (serial) set UART pin + RS485 mode, start |
| 5 | Validate | Check every API signature in `resources/api_reference.md`; verify `mb_size` (registers) vs `param_size` (bytes); verify slave area byte sizes; verify slave IP table ends with NULL |
| 6 | Build | `idf.py set-target <chip>` then `idf.py build` (CMake) |
| 7 | Flash | `idf.py -p <PORT> flash monitor` |
| 8 | Debug | Watch for `ESP_ERR_TIMEOUT` (0x107), `ESP_ERR_NOT_SUPPORTED` (0x106), `ESP_ERR_INVALID_RESPONSE` (0x108); check `CONFIG_FMB_MASTER_TIMEOUT_MS_RESPOND`; confirm RS485 wiring and direction (RTS) pin |
| 9 | Teardown | Call `mbc_master_delete()` / `mbc_slave_delete()`; for TCP also disconnect netif/services |

### Step 4 Detail — Project Creation Strategy

**When the target directory has no existing project:**

1. Pick the closest example under `examples/` and copy it wholesale, then edit `main/*.c`:
   - Serial slave → `examples/serial/mb_serial_slave/`
   - Serial master → `examples/serial/mb_serial_master/`
   - TCP slave → `examples/tcp/mb_tcp_slave/`
   - TCP master → `examples/tcp/mb_tcp_master/`
2. Both master and slave examples depend on the shared `examples/mb_example_common/` component (defines `holding_reg_params_t`, `input_reg_params_t`, `coil_reg_params_t`, `discrete_reg_params_t` and their extern instances). Keep it as a component or inline the structs.
3. Adjust the Data Dictionary / register areas, the UART pins (or netif), and the slave UID to match the device's register map.
4. Explain what was copied and why.

**When the directory already contains a project:** edit files in place; do not overwrite unless asked.

---

## Failure Strategies

| Situation | Action |
|---|---|
| API not found in `resources/api_reference.md` | Stop; inform the user it does not exist in esp-modbus v2 |
| `ESP_ERR_TIMEOUT` (0x107) on master reads | Raise `CONFIG_FMB_MASTER_TIMEOUT_MS_RESPOND` / per-instance `response_tout_ms` above worst-case RTT; check RS485 wiring, RTS direction, baud/parity match |
| `ESP_ERR_NOT_SUPPORTED` (0x106) from slave | Slave returned an exception — the requested register is not in any registered area; check Data Dictionary / `mbc_slave_set_descriptor` coverage |
| `ESP_ERR_INVALID_RESPONSE` (0x108) | Wrong command for register type, or wrong `mb_size`; verify function-code ↔ `mb_param_type` pairing |
| `ESP_ERR_INVALID_STATE` (0x103) | Stack not started, FSM busy, or TCP slave not connected; check init order and connection state |
| Slave logs "handling time … exceeds slave response time in master" | Race condition: reduce slave request processing time or increase master timeout (see `docs/en/overview_messaging_and_mapping.rst` "The Race Condition") |
| Extended-type enum / `mb_set_float_abcd` missing at build | Set `CONFIG_FMB_EXT_TYPE_SUPPORT=y` in sdkconfig |
| TCP master cannot resolve `mb_slave_tcp_01` | Either keep `CONFIG_FMB_MDNS_INTEGRATION_ENABLE=y` and run an mDNS slave, or switch the IP table to literal IP addresses |
| Old ESP-IDF link error / wrong API | Add `set(EXCLUDE_COMPONENTS freemodbus)` to the project CMakeLists.txt |

## References

- Scenario recipes → `recipes/` directory
- API reference (real function signatures) → `resources/api_reference.md`
- Kconfig / configuration reference → `resources/config_reference.md`
- Consolidated pitfalls → `resources/pitfalls.md`
- Example project index → `resources/example_list.md`
- Source repo: `D:/esp-skill/espressif-repos/esp-modbus` (docs in `docs/en/`, headers in `modbus/mb_controller/common/include/`, examples in `examples/`)
