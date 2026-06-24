# ESP-WASMachine Example Index

The repository ships exactly one reference project under `examples/`. Every path below is real.

## Reference project: `examples/wasmachine/`

The canonical ESP-WASMachine firmware application. Copy this to start any new WASMachine project.

| Path | Description |
|---|---|
| `examples/wasmachine/CMakeLists.txt` | Project CMake; sets `EXTRA_COMPONENT_DIRS` to `protocol_examples_common`, project name `wasmachine` |
| `examples/wasmachine/main/wm_main.c` | `app_main()` — init order: `bsp_init → fs_init → nvs/netif/event → wm_wamr_init → wm_wamr_app_mgr_init → wm_shell_init`; contains `fs_init()` (littleFS) and `bsp_init()`/`bsp_display_config()` (LVGL/BSP) |
| `examples/wasmachine/main/CMakeLists.txt` | Registers `wm_main.c`; runs `littlefs_create_partition_image(storage fs_image)` |
| `examples/wasmachine/main/idf_component.yml` | Managed deps: esp-box (S3), esp32_p4_function_ev_board (P4), lvgl 9.3.0 (S3/P4), joltwallet/littlefs, wasmachine_core, wasmachine_ext_wasm_native_rainmaker (non-P4), esp_wifi_remote (P4) |
| `examples/wasmachine/main/fs_image/wasm/hello_world.wasm` | Bundled test WASM app; `iwasm wasm/hello_world.wasm` prints `Hello World!` |
| `examples/wasmachine/sdkconfig.defaults` | Common config: BT/NimBLE, littleFS flags, App Manager, Shell, HTTP/MQTT/RainMaker/Wi-Fi-provisioning natives on |
| `examples/wasmachine/sdkconfig.defaults.esp32` | ESP32-DevKitC specifics (4 MB flash → single-app partition) |
| `examples/wasmachine/sdkconfig.defaults.esp32c6` | ESP32-C6-DevKitC specifics |
| `examples/wasmachine/sdkconfig.defaults.esp32s3` | ESP32-S3 specifics: QIO 8 MB flash, PSRAM octal, RMAKER factory partition, mbedtls external mem |
| `examples/wasmachine/sdkconfig.defaults.esp32p4` | ESP32-P4 specifics |
| `examples/wasmachine/sdkconfig.esp-box` | ESP32-S3-BOX / S3-BOX-Lite board overlay (pass via `-DSDKCONFIG_DEFAULTS="sdkconfig.defaults;sdkconfig.esp-box"`) |
| `examples/wasmachine/sdkconfig.esp32_p4_function_ev_board` | ESP32-P4-Function-EV-Board overlay |
| `examples/wasmachine/partitions.4mb.single_app.csv` | 4 MB single-OTA partition table (nvs, otadata, phy_init, ota_0 3072K, storage 0xf0000) |
| `examples/wasmachine/partitions.8mb.csv` | 8 MB dual-OTA partition table (adds ota_1 3072K, storage 0x1F0000) |
| `examples/wasmachine/partitions.16mb.csv` | 16 MB partition table (larger storage) |
| `examples/wasmachine/.build-test-rules.yml` | CI build-test rules |

## How to obtain it via the component manager

From `components/wasmachine_core/README.md`:

```shell
idf.py add-dependency "espressif/wasmachine_core=*"
idf.py create-project-from-example "espressif/wasmachine_core=*:wasmachine"
```

## Component-level documentation (in-repo READMEs)

| Component | README path |
|---|---|
| Core | `components/wasmachine_core/README.md` |
| Extended WASM Native | `components/wasmachine_ext_wasm_native/README.md` |
| Extended WASM VFS | `components/wasmachine_ext_wasm_vfs/README.md` |
| Data Sequence | `components/wasmachine_data_sequence/README.md` |
| RainMaker | `components/wasmachine_ext_wasm_native_rainmaker/README.md` |
| Shell | `components/wasmachine_shell/README.md` |

## Top-level documentation

| Path | Description |
|---|---|
| `README.md` | English: install env, tool/commands, compile/run, remote management |
| `README_CN.md` | Chinese counterpart |
| `docs/_static/esp-wasmachine_block_diagram.png` | Architecture block diagram (the only doc asset) |

> There is no Sphinx/`docs/index.rst` in this repo — the documentation lives in the READMEs and the source headers/Kconfig. Do not invent doc paths.
