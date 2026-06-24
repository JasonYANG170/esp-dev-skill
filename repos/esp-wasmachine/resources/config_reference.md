# ESP-WASMachine Configuration Reference

Every symbol below is a real Kconfig option from a `components/*/Kconfig.wasmachine` (sourced by `components/wasmachine_core/Kconfig.projbuild`). Defaults and `depends on` come straight from those files. When in doubt, open `menuconfig` under "WASMachine Configuration".

## Top-level menu

The aggregate menu "WASMachine Configuration" (`Kconfig.projbuild`) `orsource`s each component's `Kconfig.wasmachine`:
- `wasmachine_core/Kconfig.wasmachine` → "Generic"
- `wasmachine_ext_wasm_native/Kconfig.wasmachine` → "WASM Extended Native" (+ `wasmachine_ext_wasm_native_rainmaker`)
- `wasmachine_ext_wasm_vfs/Kconfig.wasmachine` → "WASM Extended VFS"
- `wasmachine_shell/Kconfig.wasmachine` → "Shell"

---

## Generic (`wasmachine_core`)

| Symbol | Type | Default | Depends on | Meaning |
|---|---|---|---|---|
| `WASMACHINE_APP_MGR` | bool | n | `WAMR_ENABLE_APP_FRAMEWORK` | Enable WAMR App Management (resident applets, `install`/`uninstall`/`query`, `wm_wamr_app_mgr_init`) |
| `WASMACHINE_TCP_SERVER` | bool | y | `WASMACHINE_APP_MGR` | Run a TCP server for `host_tool` |
| `WASMACHINE_TCP_PORT` | int | 8080 | `WASMACHINE_TCP_SERVER` | TCP port for the App Manager host interface |
| `WASMACHINE_FILE_SYSTEM_BASE_PATH` | string | "/storage" | — | VFS mount point; consumed as `WM_FILE_SYSTEM_BASE_PATH` and as the WASI root dir |

## WASM Extended Native (`wasmachine_ext_wasm_native`)

| Symbol | Type | Default | Depends on | Meaning |
|---|---|---|---|---|
| `WASMACHINE_WASM_EXT_NATIVE` | bool | y | — | Master switch: export WASM extended native APIs |
| `WASMACHINE_WASM_EXT_NATIVE_LIBC` | bool | y | `_EXT_NATIVE` | libc native (`open`/`read`/`write`/`ioctl`/...) |
| `WASMACHINE_WASM_EXT_NATIVE_LIBMATH` | bool | y | `_EXT_NATIVE` | libm native (`sinf`/`cosf`/`pow`) |
| `WASMACHINE_WASM_EXT_NATIVE_HTTP_CLIENT` | bool | n | `_EXT_NATIVE` | HTTP client native (esp_http_client) |
| `WASMACHINE_WASM_EXT_NATIVE_LVGL` | bool | n | `_EXT_NATIVE` | LVGL native (needs BSP: esp-box / esp32_p4_function_ev_board) |
| `WASMACHINE_WASM_EXT_NATIVE_LVGL_USE_WASM_HEAP` | bool | n | `_EXT_NATIVE` + `_LVGL` | Allocate LVGL memory from WASM app heap (needs `configNUM_THREAD_LOCAL_STORAGE_POINTERS >= 3`) |
| `WASMACHINE_WASM_EXT_NATIVE_MQTT` | bool | n | `_EXT_NATIVE` + `WASMACHINE_APP_MGR` | MQTT native (esp-mqtt); **requires App Manager** |
| `WASMACHINE_WASM_EXT_NATIVE_WIFI_PROVISIONING` | bool | n | `_EXT_NATIVE` | Wi-Fi provisioning native |
| `WASMACHINE_WASM_EXT_NATIVE_RMAKER` | bool | n | `_EXT_NATIVE` | RainMaker native (off on ESP32-P4) |

## WASM Extended VFS (`wasmachine_ext_wasm_vfs`)

| Symbol | Type | Default | Depends on | Meaning |
|---|---|---|---|---|
| `WASMACHINE_EXT_VFS` | bool | y | — | Enable Extended VFS for WASMachine |
| `WASMACHINE_EXT_VFS_UART` | bool | y | `_EXT_VFS` + `!ESP_CONSOLE_UART_DEFAULT` + `!ESP_CONSOLE_UART_CUSTOM` | Register `/dev/uart/0|1|2` to VFS (UART1/2 not auto-initialized) |

Peripheral ioctls (GPIO/I2C/SPI/LEDC) come from esp-iot-solution's `extended_vfs` and are gated by `CONFIG_EXTENDED_VFS_GPIO`/`_I2C`/`_SPI`/`_LEDC`.

## Shell (`wasmachine_shell`)

| Symbol | Type | Default | Depends on | Meaning |
|---|---|---|---|---|
| `WASMACHINE_SHELL` | bool | n | — | Enable the WASMachine shell |
| `WASMACHINE_SHELL_WASM_APP_STACK_SIZE` | string | "16384" | `_SHELL` | Default WASM app stack size (bytes), used when `iwasm` omits `-s` |
| `WASMACHINE_SHELL_WASM_APP_HEAP_SIZE` | string | "16384" | `_SHELL` | Default WASM app heap size (bytes), used when `iwasm` omits `-h` |
| `WASMACHINE_SHELL_WASM_TASK_STACK_SIZE` | int | 8192 | `_SHELL` | Stack for the WASMachine-side task that loads WASM apps |
| `WASMACHINE_SHELL_PROMPT` | string | "WASMachine>" | `_SHELL` | Console prompt string |
| `WASMACHINE_SHELL_CMD_FREE` | bool | y | `_SHELL` | `free` command |
| `WASMACHINE_SHELL_CMD_IWASM` | bool | y | `_SHELL` | `iwasm` command |
| `WASMACHINE_SHELL_CMD_INSTALL` | bool | y | `_SHELL` + `WASMACHINE_APP_MGR` | `install` command |
| `WASMACHINE_SHELL_CMD_UNINSTALL` | bool | y | `_SHELL` + `WASMACHINE_APP_MGR` | `uninstall` command |
| `WASMACHINE_SHELL_CMD_QUERY` | bool | y | `_SHELL` + `WASMACHINE_APP_MGR` | `query` command |
| `WASMACHINE_SHELL_CMD_LS` | bool | y | `_SHELL` | `ls` command |
| `WASMACHINE_SHELL_CMD_WIFI` | bool | y | `_SHELL` | `sta` command |

---

## Reference `sdkconfig.defaults` (from `examples/wasmachine`)

`examples/wasmachine/sdkconfig.defaults` sets, by default:

```ini
CONFIG_BT_ENABLED=y
CONFIG_BT_NIMBLE_ENABLED=y
CONFIG_BT_NIMBLE_HOST_TASK_STACK_SIZE=8192

CONFIG_ESP_SYSTEM_EVENT_TASK_STACK_SIZE=4096
CONFIG_FREERTOS_TIMER_TASK_STACK_DEPTH=3072

CONFIG_EXTENDED_VFS_SPI=n
CONFIG_VFS_MAX_COUNT=10

CONFIG_LITTLEFS_FCNTL_GET_PATH=y
CONFIG_LITTLEFS_OPEN_DIR=y
CONFIG_LITTLEFS_SPIFFS_COMPAT=y

CONFIG_LWIP_SO_LINGER=y

CONFIG_WASMACHINE_APP_MGR=y
CONFIG_WASMACHINE_SHELL=y
CONFIG_WASMACHINE_WASM_EXT_NATIVE_HTTP_CLIENT=y
CONFIG_WASMACHINE_WASM_EXT_NATIVE_MQTT=y
CONFIG_WASMACHINE_WASM_EXT_NATIVE_RMAKER=y
CONFIG_WASMACHINE_WASM_EXT_NATIVE_WIFI_PROVISIONING=y
```

Board-specific files add board hardware config:

- `sdkconfig.defaults.esp32s3` (S3-BOX target): `CONFIG_ESPTOOLPY_FLASHMODE_QIO=y`, `CONFIG_ESPTOOLPY_FLASHSIZE_8MB=y`, `CONFIG_ESP_RMAKER_FACTORY_PARTITION_NAME="nvs"`, custom partition `partitions.8mb.csv`, `CONFIG_SPIRAM=y` + `CONFIG_SPIRAM_MODE_OCT=y` + `CONFIG_SPIRAM_TRY_ALLOCATE_WIFI_LWIP=y`, `CONFIG_MBEDTLS_EXTERNAL_MEM_ALLOC=y`.
- `sdkconfig.defaults.esp32` / `.esp32c6`: target the 4 MB single-app partition (`partitions.4mb.single_app.csv`).
- `sdkconfig.defaults.esp32p4` + `sdkconfig.esp32_p4_function_ev_board`: P4 BSP + HDMI/MIPI-DSI display, pulls `esp_wifi_remote` instead of `esp_wifi`.
- `sdkconfig.esp-box` / `sdkconfig.esp32_p4_function_ev_board`: selected via `-DSDKCONFIG_DEFAULTS="sdkconfig.defaults;sdkconfig.<board>"`.

## Managed dependencies (`examples/wasmachine/main/idf_component.yml`)

```yaml
dependencies:
  idf:
    version: ">=5.1"
  esp-box:
    version: "3.1.*"
    rules: [ if: "target in [esp32s3]" ]
  esp32_p4_function_ev_board:
    version: "5.0.*"
    rules: [ if: "target in [esp32p4]" ]
  lvgl/lvgl:
    version: "9.3.0"
    rules: [ if: "target in [esp32s3, esp32p4]" ]
  joltwallet/littlefs:
    version: "1.*"
  wasmachine_core:
    version: "0.*"
    override_path: ../../../components/wasmachine_core
  wasmachine_ext_wasm_native_rainmaker:
    version: "0.*"
    rules: [ if: "target not in [esp32p4]" ]
    override_path: ../../../components/wasmachine_ext_wasm_native_rainmaker
  esp_wifi_remote:
    version: "0.14.0"
    rules: [ if: "target in [esp32p4]" ]
```

`wasmachine_core` itself depends on `wasmachine_data_sequence`, `wasmachine_ext_wasm_native`, `wasmachine_ext_wasm_vfs`, `wasmachine_shell`, and the managed `wasm-micro-runtime` (`== 2.*`), plus `cmake_utilities` — see `components/wasmachine_core/idf_component.yml`.

## Partition tables (`examples/wasmachine`)

| File | Layout |
|---|---|
| `partitions.4mb.single_app.csv` | `nvs` 0x4000, `otadata` 0x2000, `phy_init` 0x1000, `ota_0` 3072K, `storage`(spiffs) 0xf0000 — single OTA slot, for 4 MB boards |
| `partitions.8mb.csv` | adds `ota_1` 3072K, `storage` 0x1F0000 — dual OTA, for 8 MB boards |
| `partitions.16mb.csv` | larger storage for 16 MB boards |

The littleFS image is created from `main/fs_image/` against the `storage` partition (see `main/CMakeLists.txt`: `littlefs_create_partition_image(storage fs_image)`), and is flashed with `idf.py storage-flash`.
