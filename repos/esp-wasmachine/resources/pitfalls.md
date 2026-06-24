# ESP-WASMachine Pitfalls (consolidated)

Every entry references a real behavior found in the repo source. See `SKILL.md` "Critical Pitfalls" for the WRONG/CORRECT code blocks.

## File system

- **First boot without `idf.py storage-flash` fails**: littleFS is mounted from the `storage` partition; an empty/unformatted partition means `iwasm`/`install` have nothing to read. Always burn the image (`main/fs_image/`) at least once. Re-burning wipes prior storage data (README §4.2).
- **Wrong partition table for flash size**: 4 MB boards must use `partitions.4mb.single_app.csv`; 8 MB → `partitions.8mb.csv`. A mismatch causes overlap / out-of-flash boot failures.
- **`storage` partition label must be `"storage"`**: `fs_init()` in `wm_main.c` hard-codes `.partition_label = "storage"`. Renaming the CSV row without updating code breaks the mount.

## init order (`app_main`)

- Order is fixed by `examples/wasmachine/main/wm_main.c`: `bsp_init → fs_init → nvs/netif/event → wm_wamr_init → (wm_wamr_app_mgr_init) → (wm_shell_init)`. WAMR full-init touches `WM_FILE_SYSTEM_BASE_PATH` (WASI root dir), so the file system must already be mounted.
- `wm_wamr_init()` calls `assert(wasm_runtime_full_init(&init_args))` — if WAMR init fails, the device reboots (assert). Typical cause: heap too small; enable PSRAM on S3/P4.

## Kconfig dependencies

- `WASMACHINE_WASM_EXT_NATIVE_MQTT` requires `WASMACHINE_APP_MGR` — without APP_MGR the option is hidden and the WASM app cannot import `wasm_mqtt_*`.
- `WASMACHINE_EXT_VFS_UART` requires `!ESP_CONSOLE_UART_DEFAULT && !ESP_CONSOLE_UART_CUSTOM`. If the console is on UART0, `/dev/uart/0` cannot be exposed; switch to USB/USB-serial-JTAG console or accept using UART1/2 only.
- `WASMACHINE_WASM_EXT_NATIVE_RMAKER` is gated off on ESP32-P4 (the example yml pulls `esp_wifi_remote` instead of `esp_wifi`, and RainMaker is excluded).
- `WASMACHINE_SHELL` defaults to n; enabling it alone does NOT give you `install`/`uninstall`/`query` unless `WASMACHINE_APP_MGR` is also on.
- `WASMACHINE_WASM_EXT_NATIVE_LVGL_USE_WASM_HEAP` needs `configNUM_THREAD_LOCAL_STORAGE_POINTERS >= 3` — otherwise thread-local lookups corrupt.

## Running WASM apps

- **`iwasm` default stack/heap = 16384 bytes** (`CONFIG_WASMACHINE_SHELL_WASM_APP_STACK_SIZE`/`_HEAP_SIZE`). Heavy apps crash; pass `-s` and `-h` in bytes (e.g. `-s 262144 -h 262144`).
- **`-e`/`-d`/`-a` (`--env`/`--dir`/`--addr-pool`) only exist when `CONFIG_WAMR_ENABLE_LIBC_WASI != 0`** — `shell_iwasm.c` registers them conditionally.
- **Only bytecode/AOT/XIP packages are accepted**: `get_package_type()` must return `Wasm_Module_Bytecode`, or (`CONFIG_WAMR_ENABLE_AOT` + `Wasm_Module_AoT`). Anything else logs `pkg_type=%d is not support`.
- **`iwasm` runs the app in a dedicated thread and `pthread_join`s it** — the shell blocks until the WASM app's `main` returns. Long-running applets should use the App Manager (`install`), not `iwasm`.

## App Manager / host_tool

- **Default cap of 3 installed applets** (README §4.5). `install` over an existing name fails: "App <name> is already installed" (`shell_install.c` calls `app_manager_lookup_module_data`). `uninstall` first.
- **`install` waits up to 2000 ms** (`INSTALL_TIMEOUT`) for the module to appear; `uninstall` waits up to 1000 ms. Network/flash slowness can time out.
- **`host_tool` builds only on Linux** (`wasm-micro-runtime/test-tools/host-tool`). PC and device must be on the same AP; the device joins via `sta -s <ssid> -p <password>`.
- **TCP port** is `CONFIG_WASMACHINE_TCP_PORT` (default 8080); `host_tool`'s `-P` must match. The `_tcp_server_thread` reads up to `TCP_TX_BUFFER_SIZE` (2048) per loop.
- **`host_tool` status codes** are byte codes from WAMR (e.g. `65`=install ok, `66`=uninstall ok, `69`=query response). A non-numeric/missing response usually means wrong IP/port or AP isolation.

## VFS / peripherals

- **UART1 and UART2 `/dev/uart/*` nodes are registered but the UART driver is NOT initialized** (Kconfig help text states this directly). Firmware must `uart_driver_install()` UART1/2 before the WASM app opens them.
- **`fstat` is unsupported** — the libc wrapper returns -1 and logs a warning. WASM apps that call `fstat` will not get file sizes this way.
- **`localtime_r` is manually registered** (NULL signature): the wrapper returns the WASM-side `tp` offset, not a native pointer — WAMR does not auto-convert it.

## LVGL / BSP

- `CONFIG_WASMACHINE_WASM_EXT_NATIVE_LVGL=y` is useless without `wm_ext_wasm_native_lvgl_register_ops(...)` in `bsp_init()` — the WASM LVGL calls would have no display. The example does this only when the macro is defined.
- LVGL is brought in only for `esp32s3` and `esp32p4` (see `idf_component.yml` rules). Other targets get a link error if you enable `_LVGL`.

## Memory

- WAMR allocations go through `wamr_malloc`, which uses `MALLOC_CAP_SPIRAM | MALLOC_CAP_8BIT` when `CONFIG_SPIRAM=y` else `MALLOC_CAP_8BIT` — verify with the `free` command (it reports DRAM and PSRAM separately when PSRAM is on).
- `CONFIG_SPIRAM_MALLOC_ALWAYSINTERNAL=0` (set in the S3 defaults) routes almost everything to PSRAM; tune if internal RAM is exhausted.
