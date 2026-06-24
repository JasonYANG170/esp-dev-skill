# AGENTS.md — Supplementary Agent Guide

> Core rules, Kconfig map, pitfalls, recipes index, and execution workflow are in `SKILL.md`.
> This file covers **only** conventions and tooling not present in `SKILL.md`. Do not duplicate content.

## Project Context

- **Language**: C (ESP-IDF application firmware) · the hosted payload is `.wasm` bytecode (built off-repo with a WASM toolchain)
- **Target**: Espressif ESP32 / ESP32-S3 / ESP32-C6 / ESP32-P4
- **Toolchain / Build**: ESP-IDF v5.1.x–v5.5.x/master (`idf.py`) + ESP-IDF component manager; runtime is wasm-micro-runtime (WAMR) 2.x pulled as a managed dependency

## Repository Layout (real)

```
esp-wasmachine/
├── components/
│   ├── wasmachine_core/                 WAMR adapter + App Manager host (wm_wamr_init, wm_wamr_app_mgr_init)
│   ├── wasmachine_ext_wasm_native/      WASM Native API surface (libc/libm/http/mqtt/lvgl/wifi_prov)
│   ├── wasmachine_ext_wasm_native_rainmaker/   RainMaker native API
│   ├── wasmachine_ext_wasm_vfs/         VFS bridge (UART + extended_vfs GPIO/I2C/SPI/LEDC)
│   ├── wasmachine_shell/                Console commands (iwasm/ls/free/sta/install/uninstall/query)
│   └── wasmachine_data_sequence/        data_seq serialization helper
├── examples/wasmachine/                 The single reference project
│   ├── CMakeLists.txt
│   ├── main/{wm_main.c, CMakeLists.txt, idf_component.yml, fs_image/wasm/hello_world.wasm}
│   ├── partitions.{4mb.single_app,8mb,16mb}.csv
│   └── sdkconfig.defaults{, .esp32, .esp32c6, .esp32s3, .esp32p4, .esp-box, .esp32_p4_function_ev_board}
├── docs/_static/                        block diagram only
├── README.md / README_CN.md
└── tools/
```

## File Naming & Conventions

- Component public headers live in `components/<component>/include/` and are named `wm_*.h` (host side, e.g. `wm_wamr.h`, `wm_ext_wasm_native.h`, `wm_ext_wasm_vfs.h`, `wm_shell.h`).
- Component-private headers live in `components/<component>/private_include/` (e.g. `shell_cmd.h`, `wm_ext_wasm_native_lvgl.h`).
- WASM-callable native symbols are declared with the `wasm_*` / `wm_ext_wasm_*` prefix and registered to the `"env"` module via `wasm_native_register_natives("env", ...)`.
- Native-exported initialization functions use the `WM_EXT_WASM_NATIVE_EXPORT_FN(name)` macro so `wm_ext_wasm_native_export()` discovers them automatically by walking the `.wm_ext_wasm_native_export_fn` linker section.
- Kconfig symbols are all prefixed `WASMACHINE_*`. The aggregate menu is `Kconfig.projbuild` ("WASMachine Configuration") which `orsource`s each component's `Kconfig.wasmachine`.

## Include Pattern

```c
/* Host application (examples/wasmachine/main/wm_main.c) */
#include "esp_event.h"
#include "esp_littlefs.h"
#include "esp_log.h"
#include "nvs_flash.h"
#include "protocol_examples_common.h"

#ifdef CONFIG_WASMACHINE_SHELL
#include "wm_shell.h"
#endif
#if CONFIG_WASMACHINE_WASM_EXT_NATIVE_LVGL
#include "wm_ext_wasm_native.h"
#include "bsp/esp-bsp.h"
#endif
#include "wm_wamr.h"

/* Inside WAMR glue / a new native module */
#include "wasm_export.h"
#include "wasm_native.h"
#include "wm_ext_wasm_native_macro.h"        /* REG_NATIVE_FUNC, validate_*, addr_* */
#include "wm_ext_wasm_native_export.h"       /* WM_EXT_WASM_NATIVE_EXPORT_FN */
```

## Canonical app_main Pattern

Straight from `examples/wasmachine/main/wm_main.c` — do not reorder:

```c
void app_main(void)
{
    bsp_init();                                              /* LVGL/BSP if CONFIG_WASMACHINE_WASM_EXT_NATIVE_LVGL */
    fs_init();                                               /* esp_vfs_littlefs_register at WM_FILE_SYSTEM_BASE_PATH, label "storage" */

    ESP_ERROR_CHECK(nvs_flash_init());
    ESP_ERROR_CHECK(esp_netif_init());
    ESP_ERROR_CHECK(esp_event_loop_create_default());

#ifndef CONFIG_WASMACHINE_SHELL_CMD_WIFI
#if defined(CONFIG_EXAMPLE_CONNECT_WIFI) || defined(CONFIG_EXAMPLE_CONNECT_ETHERNET)
    ESP_ERROR_CHECK(example_connect());
#endif
#endif

    wm_wamr_init();                                          /* wasm_runtime_full_init + native/vfs export */

#ifdef CONFIG_WASMACHINE_APP_MGR
    wm_wamr_app_mgr_init();                                  /* spawns _app_mgr_thread (+ TCP server thread) */
#endif

#ifdef CONFIG_WASMACHINE_SHELL
    wm_shell_init();                                         /* registers iwasm/ls/free/sta/(install/uninstall/query) */
#endif
}
```

## Adding a New WASM Native Module

To expose a new C function to WASM applets (real pattern from the source):

```c
#include "wasm_export.h"
#include "wasm_native.h"
#include "wm_ext_wasm_native_macro.h"
#include "wm_ext_wasm_native_export.h"

static int my_func_wrapper(wasm_exec_env_t exec_env, int a, int b) {
    /* use get_module_inst(exec_env), validate_app_addr/addr_app_to_native for pointer args */
    return a + b;
}

static NativeSymbol my_native_symbol[] = {
    REG_NATIVE_FUNC(my_func, "(ii)i"),                       /* name, wrapper, signature */
};

int wm_ext_wasm_native_my_export(void)
{
    return wasm_native_register_natives("env", my_native_symbol,
                                        sizeof(my_native_symbol) / sizeof(my_native_symbol[0]))
           ? 0 : -1;
}

/* auto-discovered by wm_ext_wasm_native_export() at boot — no manual call needed */
WM_EXT_WASM_NATIVE_EXPORT_FN(wm_ext_wasm_native_my_export)
{
    return wm_ext_wasm_native_my_export();
}
```

And in the component `CMakeLists.txt`, to stop the linker dropping the section:

```cmake
target_link_libraries(${COMPONENT_LIB} INTERFACE "-u wm_ext_wasm_native_my_export")
```

WAMR signature chars: `i`=int32, `I`=int64, `f`=float32, `F`=float64, `$`=string, `*`=pointer/buffer.

## Build Workflow

1. Set up ESP-IDF: `. ./export.sh` (or `export.bat` / `export.ps1` on Windows) from the IDF root.
2. `cd` into the project (copy of `examples/wasmachine/`).
3. `idf.py set-target <esp32|esp32s3|esp32c6|esp32p4>`.
   - For S3-BOX: `idf.py -DSDKCONFIG_DEFAULTS="sdkconfig.defaults;sdkconfig.esp-box" set-target esp32s3`
   - For P4-Func-EV: `idf.py -DSDKCONFIG_DEFAULTS="sdkconfig.defaults;sdkconfig.esp32_p4_function_ev_board" set-target esp32p4`
4. `idf.py build`
5. `idf.py storage-flash` — burn the littleFS image built from `main/fs_image/` into the `storage` partition. **Do this before/at least once with the firmware flash.**
6. `idf.py flash monitor` — flash firmware and open the console. Wait for the `WASMachine>` prompt.
7. Run an app: `ls wasm` then `iwasm wasm/hello_world.wasm`.
8. (Optional) Build `host_tool` on Linux for remote install/uninstall/query over TCP (default port 8080).

## Codegen Checklist

- [ ] Target chip set via `idf.py set-target`; board `sdkconfig.defaults.<board>` passed in `SDKCONFIG_DEFAULTS` if using S3-BOX / P4-Func-EV
- [ ] Partition table file matches flash size (`partitions.4mb.single_app.csv` for 4 MB boards)
- [ ] `app_main` calls `bsp_init → fs_init → nvs/netif/event → wm_wamr_init → (app_mgr_init) → shell_init` in that order
- [ ] `storage` partition label `"storage"` matches `fs_init()`; `WM_FILE_SYSTEM_BASE_PATH` = "/storage" by default
- [ ] `main/fs_image/` contains the `.wasm` applets you want available; `idf.py storage-flash` re-burns it (wipes prior storage data)
- [ ] Every native extension enabled in `sdkconfig.defaults` has its `depends on` satisfied (MQTT needs APP_MGR; VFS UART needs console off UART; RMAKER off on P4)
- [ ] Any new native module uses `WM_EXT_WASM_NATIVE_EXPORT_FN(...)` + CMakeLists `-u` link flag
- [ ] `host_tool` (if used) built only on Linux; device on same AP as PC; `CONFIG_WASMACHINE_TCP_SERVER=y` and `CONFIG_WASMACHINE_TCP_PORT` matches `-P`
- [ ] No fabricated symbol/Kconfig: every API referenced appears in `resources/api_reference.md`; every Kconfig in `resources/config_reference.md`

## Do Not Modify

- `components/` — managed components; treat as upstream. Edit your project's `sdkconfig.defaults`, `main/wm_main.c`, and `main/fs_image/` instead.
- `SKILL.md` frontmatter — skill metadata.
- The pinned WAMR version (managed dependency `wasm-micro-runtime == 2.*`); the host_thread/CoAP framing in `wm_wamr_app_mgr.c` assumes it.
