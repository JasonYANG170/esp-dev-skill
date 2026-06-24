# AGENTS.md — Supplementary Agent Guide

> Core principles, transport tables, pitfalls, recipe index, and execution workflow live in `SKILL.md`.
> This file covers **only** conventions and tooling guidance not present in `SKILL.md`. Do not duplicate.

## Project Context

**Language**: C · **Framework**: ESP-IDF (>= 5.3) · **Build tool**: `idf.py` (CMake + Ninja)
**Targets**: Host = any ESP chipset (or non-ESP MCU via port layer); Slave/co-processor = ESP32, ESP32-C2/C3/C5/C6/C61, ESP32-S2/S3, ESP32-H2/H4.
**Component**: `espressif/esp_hosted` (registry), depends on `espressif/esp_wifi_remote` + `protobuf-c` submodule at `common/protobuf-c`.

## Repository Layout (esp-hosted-mcu)

```
esp-hosted-mcu/
├── host/                  # Host-side driver + API (linked into host app via esp_hosted component)
│   ├── esp_hosted.h       # Minimal public API: esp_hosted_init/connect_to_slave/...
│   ├── esp_hosted_misc.h  # BT controller + MAC + heartbeat + mem monitor + custom data
│   ├── esp_hosted_event.h # ESP_HOSTED_EVENT base + event IDs + payload structs
│   ├── esp_hosted_bt.h / esp_hosted_bluedroid.h   # BT HCI hooks
│   ├── api/
│   │   ├── include/       # transport_config, ota, cp_gpio, cp_ext_coex, power_save, openthread
│   │   ├── src/           # esp_wifi_weak.c (the esp_wifi_* weak impls -> RPC)
│   │   └── priv/
│   ├── drivers/transport/ # spi/sdio/uart slave-side transport drivers
│   ├── port/              # OS/platform abstraction (port here for non-ESP hosts)
│   └── utils/
├── slave/                 # Co-processor firmware project (the `slave` example target)
│   ├── main/              # slave app: spi_slave_api.c, sdio_slave_api.c, uart_slave_api.c, h_bt.c
│   ├── CMakeLists.txt
│   ├── partitions.esp32*.csv   # per-target partition tables
│   ├── sdkconfig.defaults.esp32*   # per-target defaults (transport, OpenThread RCP, etc.)
│   └── sdkconfig.ci.*            # CI build variants
├── common/                # shared (protobuf-c submodule, headers)
├── examples/              # host_* examples (see resources/example_list.md)
├── docs/                  # PRIMARY documentation tree
├── Kconfig                # Host-side menuconfig symbols (ESP_HOSTED_*)
├── idf_component.yml      # version 2.12.9, examples: [./slave]
└── README.md
```

## File & Header Conventions

- Host app source: `main/*.c`, component manifest: `main/idf_component.yml` (or root `idf_component.yml`).
- Slave source: `slave/main/*.c`. Slave project = the `slave` example from the `esp_hosted` component.
- Public host API headers live in `host/` and `host/api/include/`. Always `#include "esp_hosted.h"` first — it pulls in the relevant sub-headers.
- Do not include private headers under `host/api/priv/` or `host/drivers/` from application code.

## Include Pattern (host application)

```c
#include "esp_log.h"
#include "esp_event.h"
#include "esp_netif.h"
// ESP-IDF Wi-Fi API — weak defs fulfilled by esp_wifi_remote/esp_hosted
#include "esp_wifi.h"
// ESP-Hosted minimal + transport + misc API
#include "esp_hosted.h"
// Optionally:
// #include "esp_hosted_misc.h"        // BT controller, MAC, heartbeat (also via esp_hosted.h)
// #include "esp_hosted_cp_gpio.h"     // GPIO expander
// #include "esp_hosted_power_save.h"  // host deep sleep
```

On the slave side you normally do **not** write app code — you configure the `slave` example via menuconfig and flash it.

## Canonical Host Init Pattern

```c
void app_main(void)
{
    // 1. Netif + event loop
    ESP_ERROR_CHECK(esp_netif_init());
    ESP_ERROR_CHECK(esp_event_loop_create_default());

    // 2. Register ESP_HOSTED_EVENT handler (handles CP_INIT, TRANSPORT_UP/DOWN/FAILURE, HEARTBEAT)

    // 3. Bring up ESP-Hosted
    esp_hosted_init();
    esp_hosted_connect_to_slave();

    // 4. Wait for transport up (semaphore given in the TRANSPORT_UP handler)
    xSemaphoreTake(sem_hosted_is_up, portMAX_DELAY);

    // 5. Optional: enable co-processor BT controller (v2.5.2+)
    // esp_hosted_bt_controller_init();
    // esp_hosted_bt_controller_enable();

    // 6. Standard ESP-IDF Wi-Fi init — transparently RPC'd to the slave
    esp_netif_create_default_wifi_sta();
    wifi_init_config_t cfg = WIFI_INIT_CONFIG_DEFAULT();
    esp_wifi_init(&cfg);
    esp_wifi_set_mode(WIFI_MODE_STA);
    esp_wifi_start();
}
```

If you need to override transport pins/clock at runtime (instead of menuconfig), call `esp_hosted_<transport>_set_config()` **before** `esp_hosted_init()` — see `recipes/transport_config_code.md`.

## Standard Project Structures

### Host project (an ESP-IDF example adapted to ESP-Hosted)

```
my_host_app/
├── CMakeLists.txt
├── idf_component.yml          # add esp_wifi_remote + esp_hosted here
├── sdkconfig.defaults.<host_target>
└── main/
    ├── CMakeLists.txt
    ├── idf_component.yml      # remove esp-extconn here (ESP32-P4)
    └── main.c
```

### Co-processor (slave) project

```
my_slave/
├── CMakeLists.txt
├── sdkconfig.defaults.<slave_target>
└── main/
    └── (generated from "espressif/esp_hosted:slave" example)
```

## Build Workflow

1. **Setup ESP-IDF** (>= 5.3): Windows installer or `bash docs/setup_esp_idf__latest_stable__linux_macos.sh`.
2. **Slave** (do this first):
   ```
   idf.py create-project-from-example "espressif/esp_hosted:slave"
   cd <slave_project>
   idf.py set-target <slave_target>
   idf.py menuconfig      # Example configuration -> Bus Config -> Transport layer
   idf.py build
   idf.py -p <slave_port> flash
   ```
3. **Host**:
   ```
   cd <idf_example>                     # e.g. $IDF_PATH/examples/wifi/iperf
   idf.py add-dependency "espressif/esp_wifi_remote"
   idf.py add-dependency "espressif/esp_hosted"
   # edit main/idf_component.yml: remove esp-extconn block
   # if host has native Wi-Fi: disable WIFI in soc Kconfig.soc_caps.in
   idf.py set-target <host_target>
   idf.py menuconfig      # Component config -> ESP-Hosted config -> transport + slave chip
   idf.py build
   idf.py -p <host_port> flash monitor
   ```
4. **Cannot flash the slave?** The host may be holding lines. Put host in bootloader first:
   `esptool.py -p <host_port> --before default_reset --after no_reset run`, then retry slave flash.
5. **Subsequent slave updates**: prefer co-processor OTA over the live link (see `recipes/slave_ota.md`).

## Codegen Checklist

- [ ] Host `idf_component.yml` has `espressif/esp_wifi_remote` and `espressif/esp_hosted`
- [ ] No `espressif/esp-extconn` block remains anywhere in the host project
- [ ] Host native Wi-Fi disabled if host chip is Wi-Fi-capable
- [ ] `ESP_HOSTED_CP_TARGET_*` matches the real slave chip
- [ ] Transport selection matches on host and slave menuconfig
- [ ] Reset GPIO wired host→slave EN/RST; polarity correct
- [ ] SDIO: 51 kΩ pull-ups on CMD/DAT0–D3; classic ESP32 eFuse checked
- [ ] SPI/SDIO checksum enabled
- [ ] Init order: netif → event loop → `esp_hosted_init` → `connect_to_slave` → wait TRANSPORT_UP → Wi-Fi/BT
- [ ] BT: `esp_hosted_bt_controller_init/enable` called before BT host stack (v2.5.2+)
- [ ] `ESP_HOSTED_EVENT` handler registered for TRANSPORT_UP/DOWN/FAILURE + CP_INIT
- [ ] `esp_hosted_*_set_config()` return values checked (warn_unused_result)

## Logging

Host and slave both log via standard ESP-IDF `ESP_LOGI/...` on UART0. Look for:
- Slave: `fg_mcu_slave: ESP-Hosted-MCU Slave FW version :: X.Y.Z` and `Transport used :: SPI/SDIO/UART`.
- Host: `transport: Received INIT event from ESP32 peripheral` and `Base transport is set-up`.

## Do Not Modify

- `common/protobuf-c/` — vendored protobuf-c submodule; regenerate via protobuf, do not hand-edit.
- `host/drivers/` and `host/api/priv/` — transport internals; configure via menuconfig or the public `esp_hosted_*` API only.
- `slave/main/` transport driver sources — configure via menuconfig, do not patch for app behavior.
- `Kconfig` symbol names — reference them verbatim; do not invent `CONFIG_ESP_HOSTED_*` symbols.
- This skill's `SKILL.md` frontmatter — skill metadata.
