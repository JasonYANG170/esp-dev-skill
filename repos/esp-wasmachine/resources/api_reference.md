# ESP-WASMachine API Reference (host + WASM-native)

All entries below are taken directly from the repo source (`components/*/include/*.h`, `private_include/*.h`, and the `wasm_native_register_natives("env", ...)` registrations in `components/*/src/*.c`). If a symbol is not in this file, ESP-WASMachine does not expose it — do not invent it.

Signature convention for WAMR native symbols (registered as the `"env"` module): `i`=int32, `I`=int64, `f`=float32, `F`=float64, `$`=C-string, `*`=pointer/buffer, `~`=buffer with length implied. These are the WASM-side import signatures; the C-side wrapper receives `wasm_exec_env_t exec_env` plus mapped native pointers.

---

## 1. Host-side API (firmware calls these in `app_main`)

### Core runtime — `wm_wamr.h` / `components/wasmachine_core`

```c
void wm_wamr_init(void);                                   /* full_init + native/vfs export. Always call. */

#ifdef CONFIG_WASMACHINE_APP_MGR
void wm_wamr_app_mgr_init(void);                           /* spawns App Manager (+ TCP server) thread */
void wm_wamr_app_mgr_lock(void);
void wm_wamr_app_mgr_unlock(void);
int  wm_wamr_app_send_request(request_t *request, uint16_t msg_type);
#endif
```

- `wm_wamr_init()` sets `RuntimeInitArgs` with `Alloc_With_Allocator` and the `wamr_malloc`/`wamr_realloc`/`wamr_free` heap hooks (PSRAM-aware when `CONFIG_SPIRAM=y`), then calls `wasm_runtime_full_init(&init_args)`. It then runs `wm_ext_wasm_native_export()` and `wm_ext_wasm_vfs_init()` if the matching `CONFIG_*` is set.
- `wm_wamr_app_mgr_init()` creates the App Manager thread; that thread calls `init_wasm_timer()`, optionally sets the WASI root dir to `WM_FILE_SYSTEM_BASE_PATH`, starts a TCP server thread (port `CONFIG_WASMACHINE_TCP_PORT`, default 8080), and runs `app_manager_startup(&interface)`.
- Base path macro: `WM_FILE_SYSTEM_BASE_PATH` == `CONFIG_WASMACHINE_FILE_SYSTEM_BASE_PATH` (default `/storage`).

### WASM Extended Native (host glue) — `wm_ext_wasm_native.h`

```c
void wm_ext_wasm_native_export(void);                      /* walks the export-fn linker section, calls each */

typedef struct wm_ext_wasm_native_lvgl_ops {
    esp_err_t (*backlight_on)(void);
    esp_err_t (*backlight_off)(void);
    bool      (*lock)(uint32_t timeout_ms);
    void      (*unlock)(void);
} wm_ext_wasm_native_lvgl_ops_t;

esp_err_t wm_ext_wasm_native_lvgl_register_ops(wm_ext_wasm_native_lvgl_ops_t *ops);
```

### VFS init — `wm_ext_wasm_vfs.h`

```c
void wm_ext_wasm_vfs_init(void);   /* registers /dev/uart/x (if EXT_VFS_UART) and ext_vfs_init() */
```

### Shell — `wm_shell.h`

```c
void wm_shell_init(void);          /* registers all enabled shell commands */
```

Registered commands (each gated by `CONFIG_WASMACHINE_SHELL_CMD_*`): `iwasm`, `ls`, `free`, `sta`, and (with `CONFIG_WASMACHINE_APP_MGR`) `install`, `uninstall`, `query`.

---

## 2. WASM Native — libc (`env` module)

Source: `wm_ext_wasm_native_libc.c` → `wm_libc_wrapper_native_symbol[]`. Enabled by `CONFIG_WASMACHINE_WASM_EXT_NATIVE_LIBC` (default y).

| Import name | WAMR sig | Notes |
|---|---|---|
| `open`   | `($ii)i` | WASM-side O_* flags are remapped (e.g. `WASM_O_RDWR`, `WASM_O_CREAT`). `ioctl` added only if `CONFIG_WASMACHINE_EXT_VFS`. |
| `read`   | `(i*~)i` | |
| `write`  | `(i*~)i` | |
| `pread`  | `(i*~i)i` | |
| `pwrite` | `(i*~i)i` | |
| `lseek`  | `(iIi)I` | offset int64, returns int64 |
| `fcntl`  | `(iii)i` | |
| `fsync`  | `(i)i`   | |
| `close`  | `(i)i`   | |
| `ioctl`  | `(ii*)i` | only with `CONFIG_WASMACHINE_EXT_VFS`; dispatches to GPIO/I2C/SPI/LEDC ioctl handlers |
| `fstat`  | `(i*)i`  | **unsupported** — returns -1, logs a warning |
| `sleep`  | `(i)i`   | |
| `usleep` | `(i)i`   | |
| `time`   | `(*)I`   | |
| `srand`  | `(i)`    | |
| `rand`   | `()i`    | |
| `localtime_r` | `NULL` | manually registered (returns WASM offset of `tp`) |

WASM errno is translated C→WASI (see `WASI_E*` constants in the source).

### ioctl command IDs (Extended VFS, only when configured)

Dispatched by `ioctl_wrapper` based on `cmd`. Each is from esp-iot-solution's ioctl headers (pulled in via `wm_ext_vfs_ioctl.h`):

| Config guard | cmd values | Handler |
|---|---|---|
| `CONFIG_EXTENDED_VFS_GPIO` | `GPIOCSCFG` | `wm_ext_wasm_gpio_ioctl` |
| `CONFIG_EXTENDED_VFS_I2C`  | `I2CIOCSCFG`, `I2CIOCRDWR`, `I2CIOCEXCHANGE` | `wm_ext_wasm_i2c_ioctl` |
| `CONFIG_EXTENDED_VFS_SPI`  | `SPIIOCSCFG`, `SPIIOCEXCHANGE` | `wm_ext_wasm_native_spi_ioctl` |
| `CONFIG_EXTENDED_VFS_LEDC` | `LEDCIOCSCFG`, `LEDCIOCSSETFREQ`, `LEDCIOCSSETDUTY`, `LEDCIOCSSETPHASE`, `LEDCIOCSPAUSE`, `LEDCIOCSRESUME` | `wm_ext_wasm_native_ledc_ioctl` |

---

## 3. WASM Native — libm (`env` module)

Source: `wm_ext_wasm_native_libm.c`. Enabled by `CONFIG_WASMACHINE_WASM_EXT_NATIVE_LIBMATH` (default y).

| Import name | WAMR sig |
|---|---|
| `sinf` | `(f)f` |
| `cosf` | `(f)f` |
| `pow`  | `(FF)F` |

---

## 4. WASM Native — HTTP client (`env` module)

Source: `wm_ext_wasm_native_http_client.c`. Enabled by `CONFIG_WASMACHINE_WASM_EXT_NATIVE_HTTP_CLIENT` (default n).

Single registered import that dispatches by function ID:

| Import name | WAMR sig | Notes |
|---|---|---|
| `wasm_http_client_call_native_func` | `(ii*)i` | `(func_id, argc, args_buf)` |

Function IDs (`#define`s in the source):
- Top-level: `HTTP_CLIENT_INIT`(0), `HTTP_CLIENT_SET_STR`(1), `HTTP_CLIENT_GET_STR`(2), `HTTP_CLIENT_SET_INT`(3), `HTTP_CLIENT_GET_INT`(4), `HTTP_CLIENT_COMMON`(5).
- SET_STR ids: `HTTP_CLIENT_SET_URL`(0), `_SET_POST_FILED`(1), `_SET_HEADER`(2), `_SET_USERNAME`(3), `_SET_PASSWORD`(4), `_DELETE_HEADER`(5), `_WRITE_DATA`(6).
- GET_STR ids: `_GET_POST_FILED`(0), `_GET_HEADER`(1), `_GET_USERNAME`(2), `_GET_PASSWORD`(3), `_READ_DATA`(4), `_READ_RESP`(5), `_GET_URL`(6).
- SET_INT ids: `_SET_AUTHTYPE`(0), `_SET_METHOD`(1), `_SET_TIMEOUT`(2), `_OPEN`(3).
- GET_INT ids: `_GET_ERRNO`(0), `_FETCH_HEADER`(1), `_IS_CHUNKED`(2), `_GET_STATUS_CODE`(3), `_GET_CONTENT_LENGTH`(4), `_GET_TRANSPORT_TYPE`(5), `_IS_COMPLETE`(6), `_FLUSH_RESPONSE`(7), `_GET_CHUNK_LENGTH`(8).
- COMMON ids: `_PERFORM`(0), `_CLOSE`(1), `_CLEANUP`(2), `_SET_REDIRECTION`(3), `_ADD_AUTH`(4).

> The HTTP wrapper maps onto `esp_http_client_*` underneath; the WASM app drives it through the function-ID dispatch, not direct esp_http_client calls.

---

## 5. WASM Native — MQTT (`env` module)

Source: `wm_ext_wasm_native_mqtt.c`. Enabled by `CONFIG_WASMACHINE_WASM_EXT_NATIVE_MQTT` (default n, **requires** `CONFIG_WASMACHINE_APP_MGR`).

| Import name | WAMR sig | Notes |
|---|---|---|
| `wasm_mqtt_init`        | `(**)i`     | attr-container config in/out |
| `wasm_mqtt_destory`     | `(i)i`      | note the source spelling "destory" |
| `wasm_mqtt_start`       | `(i)i`      | |
| `wasm_mqtt_stop`        | `(i)i`      | |
| `wasm_mqtt_reconnect`   | `(i)i`      | |
| `wasm_mqtt_disconnect`  | `(i)i`      | |
| `wasm_mqtt_publish`     | `(i$*ii)i`  | (handle, topic, data, len, qos) |
| `wasm_mqtt_subscribe`   | `(i$i)i`    | (handle, topic, qos) |
| `wasm_mqtt_unsubscribe` | `(i$)i`     | (handle, topic) |
| `wasm_mqtt_enqueue`     | `(i$*iiii)i`| |
| `wasm_mqtt_set_uri`     | `(i$)i`     | |
| `wasm_mqtt_config`      | `(i*)i`     | attr-container keys below |
| `wasm_mqtt_get_outbox_size` | `(i)i`   | |

Config attr-container keys (strings) for `wasm_mqtt_init`/`wasm_mqtt_config`:
`"host"`, `"uri"`, `"port"`, `"client_id"`, `"username"`, `"password"`, `"lwt_topic"`, `"lwt_msg"`, `"lwt_qos"`, `"lwt_retain"`, `"lwt_msg_len"`, `"disable_clean_session"`, `"keepalive"`, `"disable_auto_reconnect"`, `"cert_pem"`, `"cert_len"`, `"client_cert_pem"`, `"client_cert_len"`, `"client_key_pem"`, `"client_key_len"`, `"clientkey_password"`, `"clientkey_password_len"`, `"path"`.

Event attr-container keys delivered to the WASM app: `"event_id"`, `"data"`, `"data_len"`, `"total_len"`, `"offset"`, `"topic"`, `"topic_len"`, `"msg_id"`, `"session"`, `"error_code"`, `"retain"`, `"qos"`, `"dup"`. MQTT events reach the app as `MQTT_EVENT_WASM` (`WASM_Msg_Start + 5`).

---

## 6. WASM Native — Wi-Fi provisioning (`env` module)

Source: `wm_ext_wasm_native_wifi_provisioning.c`. Enabled by `CONFIG_WASMACHINE_WASM_EXT_NATIVE_WIFI_PROVISIONING` (default n).

| Import name | WAMR sig |
|---|---|
| `wasm_wifi_prov_mgr_init`                  | `(ii*)i` |
| `wasm_wifi_prov_mgr_deinit`                | `()i` |
| `wasm_wifi_prov_mgr_is_provisioned`        | `(*)i` |
| `wasm_wifi_prov_mgr_start_provisioning`    | `(ii*i)i` |
| `wasm_wifi_prov_mgr_stop_provisioning`     | `()i` |
| `wasm_wifi_prov_mgr_wait`                  | `()i` |
| `wasm_wifi_prov_mgr_disable_auto_stop`     | `(i)i` |
| `wasm_wifi_prov_mgr_set_app_info`          | `(**ii)i` |
| `wasm_wifi_prov_mgr_endpoint_create`       | `(*)i` |
| `wasm_wifi_prov_mgr_endpoint_register`     | `(*ii)i` |
| `wasm_wifi_prov_mgr_endpoint_unregister`   | `(*)i` |
| `wasm_wifi_prov_mgr_get_wifi_state`        | `($)i` |
| `wasm_wifi_prov_mgr_get_wifi_disconnect_reason` | `($)i` |
| `wasm_wifi_prov_mgr_configure_sta`         | `($)i` |
| `wasm_wifi_prov_mgr_reset_provisioning`    | `()i` |
| `wasm_wifi_prov_mgr_reset_sm_state_on_failure` | `()i` |
| `wasm_wifi_prov_scheme_ble_set_service_uuid` | `($)i` |
| `wasm_wifi_prov_scheme_ble_set_mfg_data`   | `($i)i` |

---

## 7. WASM Native — RainMaker (`env` module)

Source: `wm_ext_wasm_native_rainmaker.c`. Enabled by `CONFIG_WASMACHINE_WASM_EXT_NATIVE_RMAKER` (default n; off on P4).

| Import name | WAMR sig |
|---|---|
| `wasm_rmaker_run`                     | `(i)i` |
| `wasm_rmaker_param_add_valid_str_list` | `(i$i)i` |
| `wasm_rmaker_call_native_func`        | `(ii*)i` |

---

## 8. WASM Native — LVGL (`env` module)

Source: `wm_ext_wasm_native_lvgl.c` + `private_include/wm_ext_wasm_native_lvgl.h`. Enabled by `CONFIG_WASMACHINE_WASM_EXT_NATIVE_LVGL` (default n; needs BSP). Optional `CONFIG_WASMACHINE_WASM_EXT_NATIVE_LVGL_USE_WASM_HEAP` redirects LVGL allocations to the WASM app heap (requires `configNUM_THREAD_LOCAL_STORAGE_POINTERS >= 3`).

Single dispatch import like the HTTP client; the function-ID constants live in `wm_ext_wasm_native_lvgl.h`. There are 460+ IDs covering the LVGL API surface, e.g.:

| ID macro | Value | What it maps to |
|---|---|---|
| `LV_OBJ_CREATE` | 11 | `lv_obj_create` |
| `LV_LABEL_CREATE` | 31 | `lv_label_create` |
| `LV_LABEL_SET_TEXT` | 32 | `lv_label_set_text` |
| `LV_OBJ_ALIGN` | 15 | `lv_obj_align` |
| `LV_BTN_CREATE` | 89 | `lv_button_create`-family |
| `LV_SLIDER_CREATE` | 99 | `lv_slider_create` |
| `LV_CHART_CREATE` | 108 | `lv_chart_create` |
| `LV_SCREEN_ACTIVE` | 316 | active screen accessor |
| `LV_TIMER_CREATE` | 38 | `lv_timer_create` |
| `LV_ANIM_START` | 73 | `lv_anim_start` |

LVGL version stamp baked into the header: `WM_LV_VERSION_MAJOR/MINOR/PATCH` = 1.0.0.

For the full ID list see `components/wasmachine_ext_wasm_native/private_include/wm_ext_wasm_native_lvgl.h`.

---

## 9. data_seq helper (`wasmachine_data_sequence`)

Used internally by native modules to serialize parameters between VM and WASM app. Header: `data_seq.h`.

```c
typedef uint16_t data_seq_type_t;
typedef uint16_t data_seq_size_t;

typedef struct data_seq_frame {
    data_seq_type_t type;
    data_seq_size_t size;
    uintptr_t       ptr;
} data_seq_frame_t;

typedef struct data_seq {
    uint32_t          version;   /* DATA_SEQ_V_1 = 0x1 */
    uint32_t          num;
    uint32_t          index;
    data_seq_frame_t  frame[0];
} data_seq_t;

data_seq_t *data_seq_alloc(uint32_t num);
void        data_seq_free(data_seq_t *ds);
void        data_seq_reset(data_seq_t *ds);

int data_seq_push(data_seq_t *ds, data_seq_type_t type, data_seq_size_t size, const void *data);   /* 0 / -EINVAL / -ENOSPC */
int data_seq_pop(data_seq_t *ds, data_seq_type_t type, data_seq_size_t size, void *data);          /* 0 / -EINVAL / -ENOENT */
int data_seq_update_frame_data(data_seq_t *ds, data_seq_type_t type, data_seq_size_t size, void *data);

/* Macros (sizeof-based) */
DATA_SEQ_PUSH(ds, t, v)
DATA_SEQ_POP(ds, t, v)
DATA_SEQ_UPDATE(ds, t, v)
DATA_SEQ_FORCE_PUSH(ds, t, v)   /* asserts on failure */
DATA_SEQ_FORCE_POP(ds, t, v)
DATA_SEQ_FORCE_UPDATE(ds, t, v)
```

Helper used by native modules: `wm_ext_data_seq_addr_wasm2c(exec_env, ds)` and `wm_ext_wasm_native_get_data_seq(exec_env, va_args)` (header `wm_ext_wasm_native_common.h`).

---

## 10. Shell command surface (host console, for reference)

These are not imports — they are the firmware console commands the user types at the `WASMachine>` prompt. Details in each command's recipe.

| Command | Options | Requires |
|---|---|---|
| `iwasm <file> [args]` | `-s/--stack_size`, `-h/--heap_size`, `-e/--env`, `-d/--dir`, `-a/--addr-pool` (WASI) | `CONFIG_WASMACHINE_SHELL_CMD_IWASM` |
| `install <file>` | `-i <name>`, `--heap <bytes>`, `--type <type>`, `--timer <n>`, `--watchdog <ms>` | `CONFIG_WASMACHINE_SHELL_CMD_INSTALL` + APP_MGR |
| `uninstall` | `-u <name>`, `--type <type>` | `CONFIG_WASMACHINE_SHELL_CMD_UNINSTALL` + APP_MGR |
| `query` | `-q <name>` (optional) | `CONFIG_WASMACHINE_SHELL_CMD_QUERY` + APP_MGR |
| `ls <path>` | — | `CONFIG_WASMACHINE_SHELL_CMD_LS` |
| `free` | — | `CONFIG_WASMACHINE_SHELL_CMD_FREE` |
| `sta` | `-s <ssid>`, `-p <password>` | `CONFIG_WASMACHINE_SHELL_CMD_WIFI` |

Remote tool: `host_tool -i/-u/-q -f <file> -S <ip> -P <port>` (built from `wasm-micro-runtime/test-tools/host-tool`, Linux only).
