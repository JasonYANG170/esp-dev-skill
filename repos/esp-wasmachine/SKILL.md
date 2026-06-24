---
name: esp-wasmachine-skill
description: >-
  AI Skill for ESP-WASMachine WebAssembly virtual-machine firmware development on ESP32 chips.
  Used when users need to create, configure, build, or debug an ESP-WASMachine project, including
  running WASM apps via the WAMR runtime, enabling WASM Native extensions (libc/libm/HTTP/MQTT/LVGL/
  RainMaker/Wi-Fi provisioning), exposing hardware through Extended VFS, and managing applications
  through the shell or the host_tool.
  Trigger words: "ESP-WASMachine", "WASMachine", "WASM", "WebAssembly", "WAMR", "iwasm", "wasm-micro-runtime", "虚拟机", "ESP32", "Espressif", "host_tool", "install wasm", "esp32s3", "esp32c6", "esp32p4"
tags:
  - embedded
  - esp32
  - espressif
  - webassembly
  - wasm
  - wamr
  - virtual-machine
  - iot
  - firmware
  - esp-idf
license: Apache-2.0
compatibility: ESP32 / ESP32-S3 / ESP32-C6 / ESP32-P4 ; build with ESP-IDF v5.1.x–v5.5.x/master and wasm-micro-runtime 2.x
metadata:
  author: Community
  version: "1.0.0"
---

# esp-wasmachine-skill

AI Skill for ESP-WASMachine firmware development. ESP-WASMachine is Espressif's WebAssembly virtual-machine (WASM-VM) development framework: it bundles the wasm-micro-runtime (WAMR) into an ESP-IDF application so an ESP32-class chip can load, instantiate, and run `.wasm`/`.wasm`-AOT applets out of a littleFS file system. This skill provides scenario-driven recipes, a real API reference, the Kconfig map, and the concrete pitfalls that come from reading the actual source under `components/` and `examples/wasmachine/`.

## Core Principles

1. **ESP-WASMachine is a host framework, not a driver library** — it runs on ESP-IDF and hosts WAMR; your "app" is usually a `.wasm` file placed in `examples/wasmachine/main/fs_image/`. The firmware is the VM.
2. **Never invent native APIs** — every WASM-callable native symbol must exist in `resources/api_reference.md`; if it is not listed (i.e. not registered via `wasm_native_register_natives("env", ...)` in the source), the WASM app cannot import it.
3. **Init order in `app_main` is fixed** — `bsp_init()` → `fs_init()` → `nvs/netif/event_loop` → `wm_wamr_init()` → (`wm_wamr_app_mgr_init()` if `CONFIG_WASMACHINE_APP_MGR`) → `wm_shell_init()`. See `examples/wasmachine/main/wm_main.c`.
4. **The file system MUST be flashed before first boot** — littleFS is mounted from the `storage` partition; an unformatted/empty partition means `iwasm`/`install` have nothing to load. Burn it with `idf.py storage-flash`.
5. **Two execution models** — (a) `iwasm <file>` runs a WASM app once to completion in its own thread; (b) `install` + App Manager (`CONFIG_WASMACHINE_APP_MGR=y`) hosts resident applets managed over TCP (`host_tool`). Pick the model before writing code.
6. **WAMR heap is the ESP heap** — `wm_wamr_init()` registers `wamr_malloc`/`wamr_realloc`/`wamr_free` over `heap_caps_aligned_alloc`. With `CONFIG_SPIRAM=y`, WAMR allocations go to PSRAM; plan stack/heap via `-s`/`-h` on `iwasm` or `--heap` on `install`.
7. **Native extensions are opt-in via Kconfig** — libc/libm default on; HTTP client, MQTT, LVGL, RainMaker, Wi-Fi provisioning default off and each `depends on WASMACHINE_WASM_EXT_NATIVE`. Enable in `sdkconfig.defaults` (e.g. `CONFIG_WASMACHINE_WASM_EXT_NATIVE_MQTT=y`).
8. **Extended VFS maps peripherals into `/dev/...`** — UART (`/dev/uart/0|1|2`), and via esp-iot-solution's `extended_vfs`: GPIO/I2C/SPI/LEDC. WASM apps reach them with `open/read/write/ioctl`, not direct registers.
9. **`install`/`uninstall` use the App Manager CoAP-style request path** — they build a `request_t` with `init_request(...)` and `COAP_PUT`/`COAP_DELETE`, sent through `wm_wamr_app_send_request()`. Up to 3 applets by default.
10. **Shell is off by default** — `CONFIG_WASMACHINE_SHELL=n`; enabling it pulls in `iwasm`, `ls`, `free`, `sta`, and (with APP_MGR) `install`/`uninstall`/`query`. Each command is a separate Kconfig toggle.
11. **LVGL native extension needs a BSP** — `wm_ext_wasm_native_lvgl_register_ops()` wires BSP display callbacks; requires `esp-box` (S3) or `esp32_p4_function_ev_board` (P4) per the example `idf_component.yml`.
12. **Target/board is chosen with `idf.py set-target` + optional `sdkconfig.defaults.<board>`** — e.g. S3-BOX needs `-DSDKCONFIG_DEFAULTS="sdkconfig.defaults;sdkconfig.esp-box"`.

## When to Use

**Applicable:**
- Creating or configuring an ESP-WASMachine firmware project (the VM host)
- Enabling/selecting WAMR runtime features and WASM Native extensions (libc, libm, HTTP, MQTT, LVGL, RainMaker, Wi-Fi provisioning)
- Wiring hardware access for WASM apps through Extended VFS (UART/GPIO/I2C/SPI/LEDC)
- Running, installing, uninstalling, or querying WASM applets via the shell or `host_tool`
- Adapting the partition table / littleFS image / `fs_image` contents
- Porting to a supported board (ESP32, ESP32-S3, ESP32-C6, ESP32-P4) and choosing the right `sdkconfig.defaults`

**Not applicable:**
- Writing/compiling the WASM applet itself (that is the esp-wdf / wasi-sdk side; this skill covers the host/VM firmware only)
- General ESP-IDF questions unrelated to the WASMachine runtime/components
- Non-Espressif WebAssembly runtimes (wasmtime, wasmer, WasmEdge)
- PCB/hardware schematic design

---

## Scenario Quick Reference (Recipes)

When user intent matches a scenario below, **read the corresponding recipe first** — it contains the real call chain, Kconfig toggles, commands, and common errors.

### Project Setup & Build

| recipe | scenario |
|---|---|
| `recipes/new_project.md` | Create/configure a new WASMachine firmware for a target chip and board |
| `recipes/build_flash.md` | Configure, build, flash firmware + littleFS image, and open the monitor |
| `recipes/target_board.md` | Select the right board `sdkconfig.defaults` (ESP32-DevKitC / S3-BOX / S3-DevKitC / C6 / P4) |

### Running WASM Apps

| recipe | scenario |
|---|---|
| `recipes/run_iwasm.md` | Load and run a WASM app from the file system with `iwasm` (stack/heap, WASI env/dir/addr-pool) |
| `recipes/app_manager.md` | Enable App Manager + TCP server; install/uninstall/query resident applets via shell and `host_tool` |
| `recipes/filesystem.md` | Add `.wasm` files to the littleFS image and burn the `storage` partition |

### WASM Native Extensions

| recipe | scenario |
|---|---|
| `recipes/native_libc.md` | libc native API (open/read/write/ioctl/close/sleep/time) — basis for all file I/O in WASM apps |
| `recipes/native_http_mqtt.md` | HTTP client and MQTT native APIs (enable + usage notes from source signatures) |
| `recipes/native_lvgl.md` | LVGL native API + BSP display ops for S3-BOX / P4 |
| `recipes/native_rainmaker_prov.md` | RainMaker and Wi-Fi provisioning native APIs |

### Hardware via Extended VFS

| recipe | scenario |
|---|---|
| `recipes/ext_vfs.md` | Extended VFS: UART `/dev/uart/x` plus GPIO/I2C/SPI/LEDC ioctl commands |

### System / Shell

| recipe | scenario |
|---|---|
| `recipes/shell_wifi.md` | `sta` command: connect the device to an AP for remote app management |
| `recipes/data_sequence.md` | `data_seq` component for serializing parameters between VM and WASM app |

---

## Supported Targets & Boards

Source: `README.md` §2, §4.1 and `examples/wasmachine/sdkconfig.defaults.*`.

| Target | `idf.py set-target` | Board-specific defaults file | Notes |
|---|---|---|---|
| ESP32 | `esp32` | `sdkconfig.defaults.esp32` | 4 MB flash uses `partitions.4mb.single_app.csv` |
| ESP32-S3 (BOX) | `esp32s3` | `sdkconfig.defaults` + `sdkconfig.esp-box` | LVGL/BSP; pass `-DSDKCONFIG_DEFAULTS="sdkconfig.defaults;sdkconfig.esp-box"` |
| ESP32-S3 (DevKitC) | `esp32s3` | `sdkconfig.defaults` | — |
| ESP32-C6 | `esp32c6` | `sdkconfig.defaults.esp32c6` | 4 MB flash uses single-app partition |
| ESP32-P4 (Function EV Board) | `esp32p4` | `sdkconfig.defaults` + `sdkconfig.esp32_p4_function_ev_board` | LVGL/BSP; uses `esp_wifi_remote` |

ESP-IDF supported versions: v5.1.x, v5.2.x, v5.3.x, v5.4.x, v5.5.x, master (pinned: v5.1.6, v5.2.5, v5.3.3, v5.4.2, v5.5-rc1).

---

## Component / Kconfig Map

Real components under `components/` and the Kconfig symbols each exposes (`Kconfig.wasmachine`).

| Component | Purpose | Key Kconfig symbols |
|---|---|---|
| `wasmachine_core` | WAMR platform adapter, app-manager host | `WASMACHINE_APP_MGR`, `WASMACHINE_TCP_SERVER`, `WASMACHINE_TCP_PORT` (8080), `WASMACHINE_FILE_SYSTEM_BASE_PATH` ("/storage") |
| `wasmachine_ext_wasm_native` | WASM Native API surface (env module) | `WASMACHINE_WASM_EXT_NATIVE`, `_LIBC`, `_LIBMATH`, `_HTTP_CLIENT`, `_LVGL`, `_LVGL_USE_WASM_HEAP`, `_MQTT` (needs APP_MGR), `_WIFI_PROVISIONING` |
| `wasmachine_ext_wasm_native_rainmaker` | RainMaker native API | `WASMACHINE_WASM_EXT_NATIVE_RMAKER` |
| `wasmachine_ext_wasm_vfs` | VFS for WASM (UART + extended_vfs bridge) | `WASMACHINE_EXT_VFS`, `WASMACHINE_EXT_VFS_UART` |
| `wasmachine_shell` | Console commands | `WASMACHINE_SHELL`, `WASMACHINE_SHELL_WASM_APP_STACK_SIZE` (16384), `WASMACHINE_SHELL_WASM_APP_HEAP_SIZE` (16384), `WASMACHINE_SHELL_WASM_TASK_STACK_SIZE` (8192), `WASMACHINE_SHELL_PROMPT` ("WASMachine>"), `WASMACHINE_SHELL_CMD_FREE/IWASM/INSTALL/UNINSTALL/QUERY/LS/WIFI` |

> Note: `WASMACHINE_WASM_EXT_NATIVE_MQTT` depends on both `WASMACHINE_WASM_EXT_NATIVE` and `WASMACHINE_APP_MGR`. `WASMACHINE_EXT_VFS_UART` depends on `!ESP_CONSOLE_UART_DEFAULT && !ESP_CONSOLE_UART_CUSTOM`.

---

## Critical Pitfalls (Must Read)

These come directly from the source. Violating any of them breaks the build or the runtime.

### 1. Flash the file system before first boot

```text
# ❌ WRONG — only flash firmware, never the storage partition
idf.py build flash
# → boot log: littleFS mount fails, iwasm has no wasm/ dir, "storage" empty

# ✅ CORRECT — burn the littleFS image generated from main/fs_image
idf.py storage-flash
# then: idf.py build flash monitor
```

### 2. app_main init order is fixed (wm_main.c)

```c
// ❌ WRONG — call wm_wamr_init() before the file system is mounted
void app_main(void) {
    wm_wamr_init();        // WAMR needs WM_FILE_SYSTEM_BASE_PATH ready
    fs_init();
}

// ✅ CORRECT — order from examples/wasmachine/main/wm_main.c
void app_main(void) {
    bsp_init();                                 // LVGL/BSP (if enabled)
    fs_init();                                  // mount littleFS at WM_FILE_SYSTEM_BASE_PATH
    ESP_ERROR_CHECK(nvs_flash_init());
    ESP_ERROR_CHECK(esp_netif_init());
    ESP_ERROR_CHECK(esp_event_loop_create_default());
    wm_wamr_init();                             // full_init + native/vfs export
#ifdef CONFIG_WASMACHINE_APP_MGR
    wm_wamr_app_mgr_init();                     // App Manager thread + TCP server
#endif
#ifdef CONFIG_WASMACHINE_SHELL
    wm_shell_init();                            // register console commands
#endif
}
```

### 3. MQTT native requires App Manager

```text
# ❌ WRONG — enable MQTT native but leave APP_MGR off
CONFIG_WASMACHINE_WASM_EXT_NATIVE_MQTT=y
CONFIG_WASMACHINE_APP_MGR=n
# → Kconfig: MQTT depends on WASMACHINE_APP_MGR, option never appears

# ✅ CORRECT
CONFIG_WASMACHINE_APP_MGR=y
CONFIG_WASMACHINE_WASM_EXT_NATIVE_MQTT=y
```

### 4. WASM app heap/stack default too small for real apps

```text
# ❌ WRONG — rely on defaults (16384) for a heavy WASM app
iwasm wasm/big_app.wasm

# ✅ CORRECT — set explicitly via -s (stack) and -h (heap), in bytes
iwasm wasm/big_app.wasm -s 262144 -h 262144
```

### 5. install requires the app to NOT already exist

```text
# ❌ WRONG — installing the same name twice
install wasm/hello.wasm -i app0
install wasm/other.wasm -i app0   # → "App app0 is already installed"

# ✅ CORRECT — uninstall first, or pick a new name
uninstall -u app0
install wasm/other.wasm -i app0
```

### 6. host_tool only builds on Linux

```text
# ❌ WRONG — try to build host_tool on Windows/macOS native
cmake ..  # fails: depends on a Linux toolchain/posix assumptions

# ✅ CORRECT — build in Linux
git clone -b WAMR-2.2.0 https://github.com/espressif/wasm-micro-runtime.git
cd wasm-micro-runtime/test-tools/host-tool && mkdir build && cd build && cmake .. && make
```

### 7. LVGL native needs BSP ops registered

```c
// ❌ WRONG — enable CONFIG_WASMACHINE_WASM_EXT_NATIVE_LVGL but never call register_ops
// → WASM LVGL calls have no display to draw to; BSP never starts

// ✅ CORRECT — in app_main bsp_init (from wm_main.c)
bsp_display_config();
wm_ext_wasm_native_lvgl_ops_t lvgl_ops = {
    .backlight_on  = bsp_display_backlight_on,
    .backlight_off = bsp_display_backlight_off,
    .lock          = bsp_display_lock,
    .unlock        = bsp_display_unlock,
};
ESP_ERROR_CHECK(wm_ext_wasm_native_lvgl_register_ops(&lvgl_ops));
```

### 8. Extended VFS UART conflicts with console UART

```text
# ❌ WRONG — request /dev/uart/0 while console owns UART0
CONFIG_ESP_CONSOLE_UART_DEFAULT=y
CONFIG_WASMACHINE_EXT_VFS_UART=y   # Kconfig: depends on !ESP_CONSOLE_UART_DEFAULT → hidden

# ✅ CORRECT — use USB/USB-serial-JTAG console, or use /dev/uart/1 (note: UART1/2 are NOT auto-initialized)
CONFIG_ESP_CONSOLE_USB_SERIAL_JTAG=y
```

### 9. UART1/UART2 VFS nodes are not initialized for you

```text
# ❌ WRONG — open "/dev/uart/1" and expect it to just work
int fd = open("/dev/uart/1", O_RDWR);   # registered but UART driver not installed

# ✅ CORRECT — UART1/2 need explicit driver init by firmware before the WASM app uses them
# (the comment in Kconfig.wasmachine states this directly)
```

### 10. Partition table must match flash size

```text
# ❌ WRONG — 4 MB board using the 8 MB / 16 MB partition table
# → overlap or storage partition past flash end; boot failure

# ✅ CORRECT — 4 MB boards use partitions.4mb.single_app.csv
# In sdkconfig.defaults.esp32:
#   CONFIG_PARTITION_TABLE_CUSTOM_FILENAME="partitions.4mb.single_app.csv"
# 8 MB boards use partitions.8mb.csv, etc.
```

### 11. WASI env/dir/addr-pool only with libc WASI enabled

```text
# ❌ WRONG — pass --env/--dir/--addr-pool when WAMR_ENABLE_LIBC_WASI is off
iwasm wasm/app.wasm -e "k=v" --dir=data   # silently ignored / error

# ✅ CORRECT — these options exist only when CONFIG_WAMR_ENABLE_LIBC_WASI != 0
# (shell_iwasm.c wraps them in #if CONFIG_WAMR_ENABLE_LIBC_WASI != 0)
```

### 12. WASM file must be valid bytecode (or AOT/XIP)

```c
// shell_iwasm.c rejects unknown package types:
// pkg_type must be Wasm_Module_Bytecode, or (with CONFIG_WAMR_ENABLE_AOT)
// Wasm_Module_AoT (XIP allowed). Anything else: "pkg_type=%d is not support"
// → compile with a WASM toolchain (wasi-sdk) before flashing.
```

---

## Execution Workflow

| Step | Name | Description |
|------|------|-------------|
| 1 | Plan | Identify target chip/board, execution model (iwasm one-shot vs App Manager resident), which native extensions and VFS peripherals are needed |
| 2 | Recipe | Match a recipe in `recipes/`; follow its Kconfig toggles and call chain |
| 3 | Query | For APIs/Kconfig not in recipes, consult `resources/api_reference.md` and `resources/config_reference.md` |
| 4 | Validate | Confirm every native symbol, Kconfig symbol, file path against `resources/` — never invent |
| 5 | Confirm | Present plan: target, sdkconfig.defaults, partition table, fs_image contents, enabled components |
| 6 | Execute | New project: copy `examples/wasmachine/` and edit `sdkconfig.defaults` + `main/fs_image/`. Existing project: edit in place |
| 7 | Check | Verify init order in `app_main`, dependencies between Kconfig symbols, partition vs flash size |
| 8 | Build | `idf.py set-target <chip>` (+ `-DSDKCONFIG_DEFAULTS=...` for board), `idf.py build` |
| 9 | Flash | `idf.py storage-flash` (file system) then `idf.py flash monitor` |
| 10 | Run | At the `WASMachine>` prompt: `ls wasm`, `iwasm wasm/hello_world.wasm`, or `sta` + `host_tool -i/-q/-u` |

### Step 6 Detail — Project Creation Strategy

**New project (no existing WASMachine project):**
1. Copy `examples/wasmachine/` (the only reference project in this repo) into the working directory — it already wires `wasmachine_core`, the native/vfs components, and the littleFS image.
2. Pick the target with `idf.py set-target <chip>`; if the board is S3-BOX or P4-Func-EV, also pass `-DSDKCONFIG_DEFAULTS="sdkconfig.defaults;sdkconfig.<board>"`.
3. Edit `sdkconfig.defaults` to toggle native extensions / shell / app manager. Add lines like `CONFIG_WASMACHINE_WASM_EXT_NATIVE_MQTT=y` (plus `CONFIG_WASMACHINE_APP_MGR=y`).
4. Drop your `.wasm` applets into `main/fs_image/wasm/` and re-run `idf.py storage-flash`.
5. Explain what was copied and why.

**Existing project:** edit `sdkconfig.defaults`, `main/wm_main.c` (init order), and `main/fs_image/` in place. Do not rewrite the components under `components/`.

---

## Failure Strategies

| Situation | Action |
|---|---|
| Native symbol not in `resources/api_reference.md` | It is not registered; do not emit it. Tell the user to enable the matching Kconfig or it does not exist |
| `iwasm` reports "pkg_type=… is not support" | The file is not valid WASM bytecode/AOT; recompile with a WASM toolchain |
| littleFS mount fails on boot | Run `idf.py storage-flash`; verify the `storage` partition exists in the partition CSV |
| Kconfig option missing in menuconfig | Check `depends on`: e.g. MQTT needs APP_MGR, VFS UART needs console not on UART, RMAKER hidden on P4 |
| `host_tool` install returns no response / timeout | Ensure device joined AP (`sta -s … -p …`), same network as PC, and `CONFIG_WASMACHINE_TCP_SERVER=y` with port matching `-P` |
| Already at 3 installed applets | `uninstall -u <name>` first; default applet cap is 3 |
| Build fails on LVGL/BSP for a non-display board | LVGL native needs `esp-box` (S3) or `esp32_p4_function_ev_board` (P4); disable `_LVGL` otherwise |

## References

- Scenario recipes → `recipes/` directory
- Native + host API reference → `resources/api_reference.md`
- Kconfig option reference → `resources/config_reference.md`
- Consolidated pitfalls → `resources/pitfalls.md`
- Real example index → `resources/example_list.md`
- Runtime lifecycle → `resources/lifecycle.md`
