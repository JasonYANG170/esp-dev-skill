# AGENTS.md — Supplementary Agent Guide

> Core principles, recipe index, reference tables, pitfalls, and the execution workflow all live in `SKILL.md`.
> This file covers **only** conventions, project layout, include/build patterns, and checklists not duplicated elsewhere.

## Project Context

**Language**: C (C11) · **Framework**: ESP-IDF (FreeRTOS) · **Targets**: ESP32-S2, ESP32-S3, ESP32-S31, ESP32-P4, ESP32-H4 (all must have a USB-OTG peripheral; `SOC_USB_OTG_SUPPORTED`). **Toolchain/Build**: ESP-IDF `idf.py` (GCC Xtensa / RISC-V). **Distribution**: IDF Component Manager managed components — not edited in place; added as dependencies.

## Component Dependencies (how to bring code in)

Code in this repository is shipped as **managed components**, consumed via `idf_component.yml`, not vendored source. Choose the dependency for the task:

| Task | Command | Adds |
|---|---|---|
| USB device (any class) | `idf.py add-dependency "espressif/esp_tinyusb^2.2.0"` | `esp_tinyusb` + public `tinyusb` (TinyUSB core) |
| Low-level USB host | `idf.py add-dependency "espressif/usb^1.4.1"` | `usb` Host Library component |
| CDC-ACM host driver | `idf.py add-dependency "espressif/usb_host_cdc_acm^2.4.0"` | `usb_host_cdc_acm` (+ pulls `usb`) |
| MSC host driver | `idf.py add-dependency "espressif/usb_host_msc^1.2.0"` | `usb_host_msc` |
| HID host driver | `idf.py add-dependency "espressif/usb_host_hid^1.2.0"` | `usb_host_hid` |
| UVC host driver | `idf.py add-dependency "espressif/usb_host_uvc^2.5.1"` | `usb_host_uvc` |
| UAC host driver | `idf.py add-dependency "espressif/usb_host_uac^1.5.0"` | `usb_host_uac` |

Host Library components require ESP-IDF **>= 5.5.3**; the device stack requires **>= 5.0**.

## Include Patterns

```c
// ---- USB Device (esp_tinyusb) ----
#include "tinyusb_default_config.h"   // TINYUSB_DEFAULT_CONFIG(), must be included to use the macro
#include "tinyusb.h"                   // tinyusb_config_t, tinyusb_driver_install/uninstall, tinyusb_event_t
#include "tinyusb_cdc_acm.h"           // CDC-ACM device API (needs CONFIG_TINYUSB_CDC_ENABLED)
#include "tinyusb_msc.h"               // MSC device API (needs CONFIG_TINYUSB_MSC_ENABLED)
#include "tinyusb_console.h"           // tinyusb_console_init/deinit (stdin/stdout redirect)
#include "vfs_tinyusb.h"               // esp_vfs_tusb_cdc_register (file I/O over CDC)
#include "tusb_config.h"               // exposes CFG_TUD_* macros derived from CONFIG_TINYUSB_*

// ---- USB Host Library (low-level) ----
#include "usb/usb_host.h"              // umbrella header; pulls usb_helpers.h, usb_types_ch9.h, etc.
// Optional, for descriptor parsing helpers:
#include "usb/usb_helpers.h"

// ---- Host class drivers ----
#include "usb/cdc_acm_host.h"          // CDC-ACM host driver
#include "usb/msc_host.h"              // MSC host driver
#include "usb/msc_host_vfs.h"          // MSC host + VFS registration
#include "usb/hid_host.h"              // HID host driver
```

## Standard Device Project Structure

```
MyUsbDeviceProject/
├── CMakeLists.txt
├── idf_component.yml                 # lists espressif/esp_tinyusb (+ tinyusb)
├── main/
│   ├── CMakeLists.txt
│   ├── main.c                        # app_main(): TINYUSB_DEFAULT_CONFIG + tinyusb_driver_install + class init
│   ├── tusb_config.h                 # only if overriding (otherwise use component's)
│   └── usb_descriptors.c             # custom device/config/string descriptors (optional)
├── partitions.csv                    # needed for MSC: a 'storage' DATA/FAT partition for SPI-flash backing
└── sdkconfig.defaults                # e.g. CONFIG_TINYUSB_CDC_ENABLED=y
```

## Standard Host Project Structure

```
MyUsbHostProject/
├── CMakeLists.txt
├── idf_component.yml                 # e.g. espressif/usb_host_cdc_acm (+ espressif/usb)
├── main/
│   ├── CMakeLists.txt
│   ├── main.c                        # app_main(): start Daemon Task, start client/class-driver task
│   ├── usb_daemon_task.c             # usb_host_install + usb_host_lib_handle_events loop
│   └── class_driver.c                # e.g. cdc_acm_host_open + data callback
└── sdkconfig.defaults                # e.g. CONFIG_USB_HOST_HUBS_SUPPORTED=y
```

## Canonical Entry / Init Patterns

### Device — minimal CDC-ACM in `app_main`

```c
#include "tinyusb_default_config.h"
#include "tinyusb.h"
#include "tinyusb_cdc_acm.h"

void app_main(void) {
    tinyusb_config_t tusb_cfg = TINYUSB_DEFAULT_CONFIG();
    ESP_ERROR_CHECK(tinyusb_driver_install(&tusb_cfg));

    const tinyusb_config_cdcacm_t acm_cfg = {
        .cdc_port = TINYUSB_CDC_ACM_0,
        .callback_rx = NULL,
        .callback_rx_wanted_char = NULL,
        .callback_line_state_changed = NULL,
        .callback_line_coding_changed = NULL,
    };
    ESP_ERROR_CHECK(tinyusb_cdcacm_init(&acm_cfg));
}
```

### Host — Daemon Task skeleton

```c
#include "usb/usb_host.h"

static void daemon_task(void *arg) {
    // (caller did usb_host_install before starting this task)
    bool exit = false;
    while (!exit) {
        uint32_t flags;
        usb_host_lib_handle_events(portMAX_DELAY, &flags);
        if (flags & USB_HOST_LIB_EVENT_FLAGS_NO_CLIENTS) {
            // all clients deregistered; try to free devices
            if (usb_host_device_free_all() == ESP_OK) { /* freed */ }
        }
        if (flags & USB_HOST_LIB_EVENT_FLAGS_ALL_FREE) {
            exit = true;
        }
    }
    usb_host_uninstall();
    vTaskDelete(NULL);
}
```

## Build Workflow

1. Set up ESP-IDF environment (`idf.py` on `PATH`, target set via `idf.py set-target esp32s3|esp32p4|...`).
2. Add the component dependency: `idf.py add-dependency "espressif/esp_tinyusb^2.2.0"` (writes `idf_component.yml`).
3. Configure class options in menuconfig: `idf.py menuconfig` → Component config → TinyUSB Stack → enable CDC/MSC/HID/etc.
4. Build: `idf.py build`.
5. Flash + monitor: `idf.py -p (PORT) flash monitor`.
6. For host tests on Linux host builds, the `usb` component also targets `linux`.

## Code Generation Checklist

- [ ] Correct component in `idf_component.yml` (device vs host; right class driver).
- [ ] ESP-IDF version matches (host needs >= 5.5.3).
- [ ] Required Kconfig option enabled (`CONFIG_TINYUSB_CDC_ENABLED`, `CONFIG_TINYUSB_MSC_ENABLED`, `CONFIG_USB_HOST_HUBS_SUPPORTED`, etc.).
- [ ] `tinyusb_config_t` initialized from `TINYUSB_DEFAULT_CONFIG()`, not bare.
- [ ] On HS-capable targets, both `full_speed_config` and `high_speed_config` (+ `qualifier`) provided.
- [ ] Composite device uses `TUSB_CLASS_MISC` / `MISC_SUBCLASS_COMMON` / `MISC_PROTOCOL_IAD`.
- [ ] External PHY: `phy.skip_setup` / `skip_phy_setup` = true; unused PHY pins = `-1`.
- [ ] Self-powered device: `self_powered = true` + valid `vbus_monitor_io`.
- [ ] Host: install before any other call; callbacks non-blocking; open→claim→transfer→release→close; full teardown before uninstall.
- [ ] CDC flush from a callback uses timeout `0`; `tud_suspend_cb`/`tud_resume_cb` not double-defined with `CONFIG_TINYUSB_SUSPEND_CALLBACK`.
- [ ] MSC backing matches chip capabilities (no SD-card on S2/H4).

## Do Not Modify

- `resources/` — generated reference docs (the API/Kconfig source of truth).
- The installed managed components under `managed_components/espressif__*` — these are downloaded; edit your own `main/` code instead.
- `SKILL.md` frontmatter — skill metadata.
- The upstream `esp-usb` repository itself — it only stores Espressif components; report issues upstream, do not patch in place.
