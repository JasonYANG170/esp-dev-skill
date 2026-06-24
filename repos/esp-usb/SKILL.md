---
name: esp-usb-skill
description: >-
  AI Skill for ESP-USB (esp_tinyusb device stack + USB Host Library and host class drivers
  CDC/MSC/HID/UAC/UVC). Used when users need to create, modify, or debug USB device or USB host
  firmware on Espressif chips, including CDC-ACM serial, MSC storage, HID, composite devices,
  VBUS monitoring, external PHY, and host class driver integration.
  Trigger words: "esp-usb", "esp_tinyusb", "TinyUSB", "USB device", "USB host", "CDC-ACM",
  "MSC", "HID", "UVC", "UAC", "usb_host", "USB-OTG", "USB serial", "USB", "USB主机", "USB设备",
  "大容量存储", "USB摄像头", "USB串口"
tags:
  - embedded
  - esp-idf
  - usb
  - usb-host
  - usb-device
  - tinyusb
  - cdc
  - msc
  - hid
  - espressif
  - esp32
license: Apache-2.0
compatibility: >-
  Targets ESP-IDF >= 5.5.3 (host) / >= 5.0 (device). Chips with USB-OTG peripheral:
  ESP32-S2, ESP32-S3, ESP32-S31, ESP32-P4, ESP32-H4. Distributed as IDF managed components
  (esp_tinyusb, usb, usb_host_cdc_acm, usb_host_msc, usb_host_hid, usb_host_uac, usb_host_uvc).
  Built with idf.py.
metadata:
  author: Community
  version: "1.1.0"
---

# esp-usb-skill

AI Skill for ESP-USB development. Covers the **esp_tinyusb** device stack (CDC-ACM serial, MSC storage, console/VFS redirection, composite devices, external PHY, VBUS monitoring) and the **USB Host Library** plus host class drivers (CDC-ACM, MSC, HID, UAC, UVC) shipped in the `esp-usb` repository as IDF managed components. All APIs, structs, macros, config options, file paths, and code snippets are grounded in the repository's real headers, Kconfig, and test apps.

## Core Principles

1. **Never guess APIs** — Every function, struct, and macro must come from `resources/api_reference.md`. If it is not there, it does not exist; do not invent it.
2. **Device and Host are different stacks** — Device = `esp_tinyusb` component (TinyUSB-based). Host = `usb` component (USB Host Library) + class-driver components (`usb_host_cdc_acm`, `usb_host_msc`, `usb_host_hid`, ...). Pick the right one before writing any code.
3. **Initialize `tinyusb_config_t` with `TINYUSB_DEFAULT_CONFIG()`** — Then override only the fields you need. Descriptor pointers left `NULL` fall back to built-in defaults.
4. **Enable the class in menuconfig before calling its API** — `tinyusb_cdcacm_init()` hard-errors (`#error`) if `CONFIG_TINYUSB_CDC_ENABLED` is off; `tinyusb_msc_*` needs `CONFIG_TINYUSB_MSC_ENABLED`. Set these via menuconfig, not in code.
5. **Install order matters for Host** — `usb_host_install()` must run before any other Host Library call. Clients register with `usb_host_client_register()` after install. The Daemon Task drives `usb_host_lib_handle_events()`; client tasks drive `usb_host_client_handle_events()`.
6. **Host callbacks must not block** — Client event callbacks and transfer-completion callbacks run *inside* `usb_host_client_handle_events()`. Blocking there stalls all pending client events.
7. **Host clients open → claim interface → transfer → release → close** — `usb_host_device_open` → `usb_host_interface_claim` → `usb_host_transfer_submit` → `usb_host_interface_release` → `usb_host_device_close`. Two clients may share a device only on different interfaces.
8. **Uninstall has a lifecycle** — Device: `tinyusb_driver_uninstall()`. Host: deregister all clients, call `usb_host_device_free_all()` until `USB_HOST_LIB_EVENT_FLAGS_ALL_FREE`, then `usb_host_uninstall()`. Skipping steps leaks resources or fails with `ESP_ERR_INVALID_STATE`.
9. **Self-powered devices must monitor VBUS** — USB spec requires it. Set `tinyusb_phy_config_t::self_powered = true` and `vbus_monitor_io` via a voltage divider / comparator; the GPIO must read logic-low within 3 ms of unplug.
10. **External PHY needs `skip_setup = true`** — When you configure an external PHY with `usb_new_phy()` (ESP32-S3 SP5301/TUSB1106/STUSB03E), set `tinyusb_config_t.phy.skip_setup = true` (device) or `usb_host_config_t.skip_phy_setup = true` (host) to bypass internal PHY re-init. Unused PHY pins must be `-1`.
11. **MSC buffer size drives performance** — `CONFIG_TINYUSB_MSC_BUFSIZE` (and `CONFIG_TINYUSB_CDC_EP_BUFSIZE` for CDC) is the single biggest perf lever. 512 B → ~0.5 MB/s; 8192 B → ~5–6 MB/s on ESP32-P4. Use SD card over internal SPI flash for sustained throughput.
12. **Descriptors must match speed** — High-speed-capable chips (ESP32-P4, ESP32-S31) need *both* `full_speed_config` and `high_speed_config`, plus a `qualifier`. Leaving a descriptor `NULL` only works if esp_tinyusb has a built-in default for that class.

## When to Use

**Applicable:**
- Building a USB **device**: CDC-ACM serial, MSC flash/SD storage, HID keyboard/mouse, MIDI, composite (CDC+MSC), vendor-specific, NCM/RNDIS Ethernet-over-USB
- Building a USB **host**: talking to CDC-ACM modems/serial dongles (CP210x/FTDI/CH34x), MSC flash drives, HID keyboards/mice, UAC audio, UVC cameras
- Using the low-level USB Host Library API directly (custom class driver, raw transfers, descriptor parsing)
- Configuring external PHY, VBUS monitoring, self-powered device behavior, suspend/resume, remote wakeup
- Switching the console (stdin/stdout/stderr) from UART to USB, or registering a CDC interface in VFS

**Not applicable:**
- Non-USB serial (UART/SPI/I2C) — use the relevant ESP-IDF driver skills
- USB Type-C / USB-PD / TCPM (that is the separate `usb_tcpm` component; not covered by CDC/MSC/HID recipes here)
- PCB design, schematic review, or USB-C connector pinout questions
- Other vendors' USB stacks (STM32, NXP, etc.)

---

## Scenario Quick Reference (Recipes)

When the user's intent matches a scenario below, **read the corresponding recipe first** — it contains the full call chain, step-by-step instructions, common errors, and copy-able code.

### USB Device (esp_tinyusb)

| recipe | scenario |
|---|---|
| `recipes/device_cdc_serial.md` | USB device as a CDC-ACM serial port — init, RX/TX, callbacks, double-CDC |
| `recipes/device_msc_storage.md` | USB device as Mass Storage — SPI-flash or SD-card backing, mount/unmount events |
| `recipes/device_console_vfs.md` | Redirect stdin/stdout to USB (`tinyusb_console_init`), register CDC in VFS (`esp_vfs_tusb_cdc_register`) |
| `recipes/device_composite.md` | Composite device (CDC + MSC) — custom descriptors, IAD, `bDeviceClass = TUSB_CLASS_MISC` |
| `recipes/device_external_phy.md` | External PHY on ESP32-S3 (SP5301/TUSB1106/STUSB03E) — `usb_new_phy` + `skip_setup` |
| `recipes/device_install_uninstall.md` | Driver install/uninstall lifecycle, attach/detach events, self-powered VBUS monitoring |
| `recipes/device_ncm_net.md` | NCM / RNDIS Ethernet-over-USB device — `tinyusb_net_init`, RX/TX callbacks, sync/async send, NTB buffer tuning |

### USB Host (usb + class drivers)

| recipe | scenario |
|---|---|
| `recipes/host_library_basic.md` | USB Host Library install, Daemon Task, client register, device open, raw transfer |
| `recipes/host_cdc_acm.md` | CDC-ACM host driver — open modem/serial dongle, `tx_blocking`, RX data callback, vendor-specific |
| `recipes/host_msc.md` | MSC host — read/write sectors to a USB flash drive, VFS registration, device info |
| `recipes/host_hid.md` | HID host — keyboard/mouse, interface events, raw input reports |
| `recipes/host_uvc.md` | UVC host — USB camera video streaming, format negotiation, frame callbacks, PSRAM frame buffers |
| `recipes/host_uac.md` | UAC host — USB audio (speakers/mics), alt-setting format query, start/stop, suspend/resume, volume/mute |

---

## Chip & Component Support Reference

### Device Stack (esp_tinyusb)

| Feature | ESP32-S2 | ESP32-S3 | ESP32-P4 | ESP32-S31 | ESP32-H4 |
|---|---|---|---|---|---|
| USB peripheral | FS (OTG) | FS (OTG) | HS + FS (OTG) | HS (OTG) | FS (OTG) |
| External PHY | — | yes | — | — | — |
| Default `TINYUSB_DEFAULT_CONFIG()` port | `TINYUSB_PORT_FULL_SPEED_0` | `TINYUSB_PORT_FULL_SPEED_0` | `TINYUSB_PORT_HIGH_SPEED_0` | `TINYUSB_PORT_FULL_SPEED_0` | `TINYUSB_PORT_FULL_SPEED_0` |
| MSC SD-card backing | not available | yes | yes | yes | not available |

> USB 1.1 (FS) peripheral D+/D- are routed to GPIOs (default DP=20, DM=19 on most targets; P4 uses 27/26; H4 uses 22/21). USB 2.0 (HS, P4) routes to dedicated pins.

### Host Library (usb component)

| Capability | S2 / S3 / H4 | S31 / P4 |
|---|---|---|
| Speeds supported | FS, LS | HS, FS, LS |
| Host channels | 8 (S3 default) | 16 |
| High-bandwidth isochronous | no | yes |
| Dual independent host (P4) | — | yes (HS + FS controllers) |

### Host Class Driver Components (real versions from `idf_component.yml`)

| component | version | class |
|---|---|---|
| `usb_host_cdc_acm` | 2.4.0 | CDC-ACM host (incl. CP210x, FTDI, CH34x VCP helpers) |
| `usb_host_msc` | 1.2.0 | MSC host (Bulk-Only Transport) |
| `usb_host_hid` | 1.2.0 | HID host |
| `usb_host_uvc` | 2.5.1 | UVC host (USB cameras) |
| `usb_host_uac` | 1.5.0 | UAC host (USB audio) |
| `usb` | 1.4.1 | USB Host Library (low-level) |
| `esp_tinyusb` | 2.2.0 | Device stack |

---

## USB Host Library Lifecycle (state map)

```
[usb_host_install]
        │
        ▼
   Daemon Task loop: usb_host_lib_handle_events(portMAX_DELAY, &flags)
        │                └─ flags & NO_CLIENTS  → all clients gone
        │                └─ flags & ALL_FREE    → safe to uninstall
        │
   Client tasks: usb_host_client_register → handle_events loop
        │
        ▼  NEW_DEV event
   usb_host_device_open → get descriptors → usb_host_interface_claim
        │
        ▼
   usb_host_transfer_submit / submit_control  (transfer_cb on done)
        │
        ▼  DEV_GONE event (or done)
   usb_host_interface_release → usb_host_device_close
        │
        ▼
   usb_host_client_deregister
        │
        ▼  (back in Daemon)
   usb_host_device_free_all → wait ALL_FREE → usb_host_uninstall
```

---

## Critical Pitfalls (Must Read)

The most common errors. Violating any of these yields non-working firmware or linker/runtime failures.

### 1. Class API used without enabling the Kconfig option

```c
// ❌ WRONG — tinyusb_cdc_acm.h hard-errors at compile time
// if CONFIG_TINYUSB_CDC_ENABLED is not set in menuconfig
#include "tinyusb_cdc_acm.h"
tinyusb_cdcacm_init(&acm_cfg);   // #error "TinyUSB CDC driver must be enabled in menuconfig"

// ✅ CORRECT — enable the option in menuconfig first
// Component config → TinyUSB Stack → Communication Device Class (CDC) → Enable TinyUSB CDC feature = y
// (or CONFIG_TINYUSB_MSC_ENABLED, CONFIG_TINYUSB_HID_COUNT > 0, etc.)
#include "tinyusb_cdc_acm.h"
tinyusb_config_cdcacm_t acm_cfg = {
    .cdc_port = TINYUSB_CDC_ACM_0,
    .callback_rx = my_rx_cb,
};
ESP_ERROR_CHECK(tinyusb_cdcacm_init(&acm_cfg));
```

### 2. Not using `TINYUSB_DEFAULT_CONFIG()` as the starting point

```c
// ❌ WRONG — manual init, easy to miss PHY/task fields, broken descriptors
tinyusb_config_t tusb_cfg;
tusb_cfg.port = 0;
tinyusb_driver_install(&tusb_cfg);   // random stack fields, unpredictable behavior

// ✅ CORRECT — start from defaults, override only what you need
tinyusb_config_t tusb_cfg = TINYUSB_DEFAULT_CONFIG();
tusb_cfg.descriptor.device = &my_device_descriptor;
tusb_cfg.descriptor.full_speed_config = my_config_descriptor;
ESP_ERROR_CHECK(tinyusb_driver_install(&tusb_cfg));
```

### 3. High-speed chip missing the qualifier / high_speed_config

```c
// ❌ WRONG — on ESP32-P4/ESP32-S31 only full_speed_config set; USB 2.0 spec violation
tinyusb_config_t tusb_cfg = TINYUSB_DEFAULT_CONFIG();
tusb_cfg.descriptor.full_speed_config = my_config_desc;
// qualifier and high_speed_config left NULL when SOC_USB_OTG_PERIPH_NUM > 1

// ✅ CORRECT — provide both speed descriptors + qualifier on HS-capable targets
#if (SOC_USB_OTG_PERIPH_NUM > 1)
    tusb_cfg.descriptor.high_speed_config = my_config_desc;
    tusb_cfg.descriptor.qualifier = &my_qualifier;
#endif
```

### 4. Composite descriptor with wrong device class

```c
// ❌ WRONG — uses 0x00/0x00 for a composite (IAD) device; hosts may mis-enumerate
static const tusb_desc_device_t desc = {
    .bDeviceClass = 0x00,
    .bDeviceSubClass = 0x00,
    .bDeviceProtocol = 0x00,
    ...
};

// ✅ CORRECT — IAD composite must set MISC class/subclass/protocol
static const tusb_desc_device_t desc = {
    .bLength = sizeof(desc),
    .bDescriptorType = TUSB_DESC_DEVICE,
    .bcdUSB = 0x0200,
    .bDeviceClass = TUSB_CLASS_MISC,
    .bDeviceSubClass = MISC_SUBCLASS_COMMON,
    .bDeviceProtocol = MISC_PROTOCOL_IAD,
    .bMaxPacketSize0 = CFG_TUD_ENDPOINT0_SIZE,
    .idVendor = TINYUSB_ESPRESSIF_VID,
    ...
};
```

### 5. Blocking inside a Host Library client callback

```c
// ❌ WRONG — blocks usb_host_client_handle_events(); other events starve
static void client_event_cb(const usb_host_client_event_msg_t *msg, void *arg) {
    if (msg->event == USB_HOST_CLIENT_EVENT_NEW_DEV) {
        vTaskDelay(pdMS_TO_TICKS(500));          // never block here
        usb_host_device_open(hdl, msg->new_dev.address, &dev); // heavy work
    }
}

// ✅ CORRECT — set a flag in the callback; do the work in the client task loop
static void client_event_cb(const usb_host_client_event_msg_t *msg, void *arg) {
    struct ctrl *c = arg;
    if (msg->event == USB_HOST_CLIENT_EVENT_NEW_DEV) {
        c->actions |= ACT_OPEN;
        c->dev_addr = msg->new_dev.address;
    }
}
// ... then in the task loop, if (c->actions & ACT_OPEN) usb_host_device_open(...);
```

### 6. Calling `tinyusb_cdcacm_write_flush` with a timeout from inside a CDC callback

```c
// ❌ WRONG — TinyUSB defers endpoint flushing until callback processing completes;
//           the flush will block until the timeout expires
static void rx_cb(int itf, cdcacm_event_t *event) {
    tinyusb_cdcacm_write_queue(itf, reply, n);
    tinyusb_cdcacm_write_flush(itf, pdMS_TO_TICKS(100));  // deadlocks ~100 ms
}

// ✅ CORRECT — flush with timeout=0 (non-blocking), or flush from another task
static void rx_cb(int itf, cdcacm_event_t *event) {
    tinyusb_cdcacm_write_queue(itf, reply, n);
    tinyusb_cdcacm_write_flush(itf, 0);   // returns ESP_ERR_NOT_FINISHED if pending
}
```

### 7. Host uninstall before freeing all devices

```c
// ❌ WRONG — uninstall while devices still allocated → ESP_ERR_INVALID_STATE
usb_host_uninstall();   // clients/devices not cleaned up

// ✅ CORRECT — full teardown sequence in the Daemon Task
// 1. all clients deregistered
// 2. usb_host_device_free_all() returns ESP_OK only when all freed
// 3. wait for USB_HOST_LIB_EVENT_FLAGS_ALL_FREE
// 4. then usb_host_uninstall()
if (event_flags & USB_HOST_LIB_EVENT_FLAGS_NO_CLIENTS) {
    if (usb_host_device_free_all() == ESP_OK) {
        // all freed immediately
    } else {
        // wait for ALL_FREE flag in subsequent handle_events calls
    }
}
if (event_flags & USB_HOST_LIB_EVENT_FLAGS_ALL_FREE) {
    usb_host_uninstall();
}
```

### 8. External PHY configured but internal PHY still re-initialized

```c
// ❌ WRONG — esp_tinyusb re-inits the (nonexistent) internal PHY and overwrites the external one
usb_new_phy(&phy_config, &phy_hdl);
tinyusb_config_t tusb_cfg = TINYUSB_DEFAULT_CONFIG();
tinyusb_driver_install(&tusb_cfg);   // skip_setup still false

// ✅ CORRECT — bypass PHY setup so the external PHY you configured survives
usb_new_phy(&phy_config, &phy_hdl);
tinyusb_config_t tusb_cfg = TINYUSB_DEFAULT_CONFIG();
tusb_cfg.phy.skip_setup = true;            // device mode
tinyusb_driver_install(&tusb_cfg);
// For host mode: usb_host_config_t.skip_phy_setup = true;
```

### 9. Self-powered device without VBUS monitoring

```c
// ❌ WRONG — device keeps enumerating after unplug; USB spec violation for self-powered
tinyusb_config_t tusb_cfg = TINYUSB_DEFAULT_CONFIG();
// self_powered left false; no vbus_monitor_io

// ✅ CORRECT — self-powered devices MUST monitor VBUS
tinyusb_config_t tusb_cfg = TINYUSB_DEFAULT_CONFIG();
tusb_cfg.phy.self_powered = true;
tusb_cfg.phy.vbus_monitor_io = VBUS_GPIO;   // via divider/comparator
// Note (ESP32-P4 HS port): call gpio_install_isr_service() before tinyusb_driver_install()
```

### 10. `tud_suspend_cb` / `tud_resume_cb` defined by the app while callback option is enabled

```c
// ❌ WRONG — CONFIG_TINYUSB_SUSPEND_CALLBACK=y means esp_tinyusb already provides a
//            strong tud_suspend_cb(); defining your own → linker error (multiple definition)
void tud_suspend_cb(bool remote_wakeup) { my_logics(); }

// ✅ CORRECT — either disable the menuconfig option and provide your own,
//              or keep the option on and handle TINYUSB_EVENT_SUSPENDED in tinyusb_event_cb_t
// Option on (recommended): handle via the event callback
void my_event_cb(tinyusb_event_t *event, void *arg) {
    if (event->id == TINYUSB_EVENT_SUSPENDED) {
        // event->suspended.remote_wakeup tells if host enabled remote wakeup
    }
}
```

### 11. Host transfer submitted to an endpoint whose interface was not claimed

```c
// ❌ WRONG — interface 1 never claimed; submit fails or behavior is undefined
usb_host_device_open(client, addr, &dev);
usb_host_transfer_submit(transfer);   // bEndpointAddress = 0x01

// ✅ CORRECT — claim the interface before submitting to its endpoints
usb_host_device_open(client, addr, &dev);
usb_host_interface_claim(client, dev, 1, 0);   // interface 1, alt 0
transfer->device_handle = dev;
transfer->bEndpointAddress = 0x01;
usb_host_transfer_submit(transfer);
```

### 12. MSC SD-card backing used on a chip without SDMMC host support

```c
// ❌ WRONG — ESP32-S2 / ESP32-H4 have no SDMMC host; tinyusb_msc_new_storage_sdmmc
//            is #ifdef'd out (SOC_SDMMC_HOST_SUPPORTED)
tinyusb_msc_new_storage_sdmmc(&cfg, &hdl);   // link error / undefined reference

// ✅ CORRECT — on S2/H4 use SPI-flash backing only
tinyusb_msc_new_storage_spiflash(&cfg, &hdl);
// SD-card backing is available on S3 / P4 / S31
```

---

## Execution Workflow

| Step | Name | Description |
|------|------|-------------|
| 1 | Classify | Is this a USB **device** or USB **host** task? Device → `esp_tinyusb`; host → `usb` + class-driver component. |
| 2 | Recipe | Match the intent to a recipe in `recipes/` and follow its call chain and code. |
| 3 | Query | For APIs/structs/config not in the recipe, look them up in `resources/api_reference.md` and `resources/config_reference.md`. |
| 4 | Validate | Confirm: required Kconfig option enabled, correct descriptor set for the target speed, PHY config correct, callbacks non-blocking. |
| 5 | Confirm | Present the plan: component dependencies (`idf.py add-dependency`), includes, init order, menuconfig changes, main loop. |
| 6 | Execute | Add the dependency to `idf_component.yml`, set menuconfig, write/modify code in place. Adapt real example code from `resources/example_list.md`. |
| 7 | Build | `idf.py build` (host needs ESP-IDF >= 5.5.3; device >= 5.0). Fix descriptor/Kconfig errors first. |
| 8 | Verify | Device: host sees a new serial/MSC/HID device. Host: client gets `USB_HOST_CLIENT_EVENT_NEW_DEV` and the transfer completes. |
| 9 | Debug | Use `CONFIG_TINYUSB_DEBUG_LEVEL` (0–3) for device logs; CDC console (`tinyusb_console_init`) or UART for printf. |

---

## Failure Strategies

| Situation | Action |
|---|---|
| API/struct not found in `resources/` | Stop; do not fabricate. Tell the user it is not part of this component. |
| `#error "TinyUSB CDC driver must be enabled"` | Enable `CONFIG_TINYUSB_CDC_ENABLED` (or MSC/HID/etc.) in menuconfig. |
| `ESP_ERR_INVALID_STATE` on install | The driver/library is already installed. Call uninstall/teardown first. |
| Host device never reaches NEW_DEV | Verify Daemon Task runs `usb_host_lib_handle_events`, PHY is up, and the device is USB-2.0 compliant (try the enum timing Kconfig options). |
| Composite device not enumerated by Windows | Check `bDeviceClass = TUSB_CLASS_MISC` + IAD, and that both speed descriptors are provided on HS targets. |
| MSC throughput very low | Raise `CONFIG_TINYUSB_MSC_BUFSIZE` (up to 8192 FS / 32768 HS) and use SD-card instead of internal SPI flash. |
| External PHY not working | Confirm `phy.skip_setup` (device) / `skip_phy_setup` (host) is `true`; set unused PHY pins to `-1`. |
| Linker error: multiple `tud_suspend_cb` | Disable `CONFIG_TINYUSB_SUSPEND_CALLBACK` if you define your own, or remove yours and use the event callback. |
| Light sleep with USB host misbehaves | Enable `CONFIG_USB_HOST_AUTO_PM_LIGHT_SLEEP` and the sleep/PM prerequisites; root port auto-suspends before sleep. |

## References

- Scenario recipes → `recipes/` directory
- Device + Host API quick reference → `resources/api_reference.md`
- Kconfig / menuconfig options → `resources/config_reference.md`
- Consolidated gotchas → `resources/pitfalls.md`
- Real example/test-app index → `resources/example_list.md`
- Conventions & tooling → `AGENTS.md`
