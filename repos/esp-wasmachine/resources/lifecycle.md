# ESP-WASMachine Runtime Lifecycle

Describes the real boot/bring-up and the two execution models, grounded in `examples/wasmachine/main/wm_main.c`, `components/wasmachine_core/src/wm_wamr.c`, and `wm_wamr_app_mgr.c`.

## Boot sequence (firmware)

```
app_main()                              [wm_main.c]
  ├─ bsp_init()                         only if CONFIG_WASMACHINE_WASM_EXT_NATIVE_LVGL
  │     ├─ bsp_display_config()         bsp_display_start_with_config(...)
  │     └─ wm_ext_wasm_native_lvgl_register_ops(&lvgl_ops)
  ├─ fs_init()                          esp_vfs_littlefs_register(label="storage", base=WM_FILE_SYSTEM_BASE_PATH)
  ├─ nvs_flash_init()
  ├─ esp_netif_init()
  ├─ esp_event_loop_create_default()
  ├─ example_connect()  (only if !WASMACHINE_SHELL_CMD_WIFI && EXAMPLE_CONNECT_*)
  ├─ wm_wamr_init()                     [wm_wamr.c]
  │     ├─ wasm_runtime_full_init(...)  Alloc_With_Allocator + wamr_malloc/realloc/free
  │     ├─ wm_ext_wasm_native_export()  walks .wm_ext_wasm_native_export_fn section (if _EXT_NATIVE)
  │     └─ wm_ext_wasm_vfs_init()       registers /dev/uart/* + ext_vfs_init (if _EXT_VFS)
  ├─ wm_wamr_app_mgr_init()             only if CONFIG_WASMACHINE_APP_MGR
  │     └─ pthread_create(_app_mgr_thread)
  │           ├─ init_wasm_timer()
  │           ├─ wasm_set_wasi_root_dir(WM_FILE_SYSTEM_BASE_PATH)   if WAMR_ENABLE_LIBC_WASI
  │           ├─ pthread_create(_tcp_server_thread)                  if WASMACHINE_TCP_SERVER
  │           │     └─ bind/listen/accept on CONFIG_WASMACHINE_TCP_PORT, loop read → aee_host_msg_callback
  │           └─ app_manager_startup(&interface)
  └─ wm_shell_init()                    only if CONFIG_WASMACHINE_SHELL → registers enabled commands
```

After bring-up the console prints (README §4.3):

```
Type 'help' to get the list of commands.
Use UP/DOWN arrows to navigate through command history.
Press TAB when typing command name to auto-complete.
WASMachine>
```

## Execution model A — `iwasm` (one-shot)

`shell_iwasm.c` `iwasm_main`:

```
iwasm <file> [args] -s <stack> -h <heap> [-e/-d/-a ...]
  ├─ shell_open_file(file)                 read .wasm bytes from WM_FILE_SYSTEM_BASE_PATH
  ├─ start_iwasm_thread(...)
  │     ├─ pthread_create(iwasm_main_thread, stack = CONFIG_WASMACHINE_SHELL_WASM_TASK_STACK_SIZE)
  │     └─ pthread_join(...)               shell BLOCKS until app exits
  └─ iwasm_main_thread:
        ├─ (WASI) wasm_runtime_set_wasi_args / set_wasi_addr_pool
        ├─ get_package_type()              must be Wasm_Module_Bytecode (or AOT/XIP)
        ├─ wasm_runtime_load(buffer, size)
        ├─ wasm_runtime_instantiate(module, stack_size, heap_size)
        ├─ wasm_application_execute_main(inst, argc, argv)
        ├─ wasm_runtime_get_exception()    log if set
        ├─ wasm_runtime_deinstantiate()
        └─ wasm_runtime_unload()
```

Suitable for short-lived WASM programs. Default stack/heap = 16384 bytes each.

## Execution model B — App Manager (resident applets)

Requires `CONFIG_WASMACHINE_APP_MGR=y`. `install`/`uninstall`/`query` are available from the shell (and over TCP via `host_tool`).

```
install <file> -i <name> [--heap N] [--type T] [--timer N] [--watchdog ms]
  ├─ wm_wamr_app_mgr_lock()
  ├─ app_manager_lookup_module_data(name)   reject if already installed
  ├─ shell_open_file(file)
  ├─ init_request(req, "/applet?name=<name>&heap=...", COAP_PUT, FMT_APP_RAW_BINARY, payload, size)
  ├─ wm_wamr_app_send_request(req, INSTALL_WASM_APP)   → aee_host_msg_callback(...)
  └─ poll app_manager_lookup_module_data for ≤ INSTALL_TIMEOUT (2000 ms)
```

```
uninstall -u <name>
  ├─ COAP_DELETE / FMT_ATTR_CONTAINER via wm_wamr_app_send_request(req, REQUEST_PACKET)
  └─ poll until lookup returns NULL (≤ UNISTALL_TIMEOUT = 1000 ms)
```

```
query [-q name]
  └─ iterates module_data_list; prints JSON:
       { "num": N, "applet1": "...", "heap1": H, ... }
```

`host_tool` (Linux) talks to `_tcp_server_thread` using the same leading-bytes `0x12 0x34` + msg_type + size framing that `wm_wamr_app_send_request` produces. Default cap: 3 applets.

## Shutdown / errors

- `wm_wamr_init()` asserts on `wasm_runtime_full_init` failure (typically reboot).
- `_app_mgr_thread` on failure path calls `wasm_runtime_destroy()`.
- A WASM app exception is captured by `wasm_runtime_get_exception()` and logged; it does not crash the VM.
