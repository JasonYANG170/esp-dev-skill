# Changelog

All notable changes to this skill are documented here.
The format is based on [Keep a Changelog](https://keepachangelog.com/), and this
project adheres to its principles.

## [1.1.0] - 2026-06-22

Added three confirmed gap recipes (two Host class drivers + one device networking
class) that were documented-but-uncovered in v1.0.0. All APIs, structs, Kconfig
options, and code paths are grounded in the real `esp-usb` repository headers,
Kconfig, docs, and example/test-app sources.

### Added
- `recipes/host_uvc.md` — UVC host (USB cameras): install order, driver/device
  callbacks, `uvc_host_get_frame_list`, stream open/start/format-select, frame
  buffer ownership (`uvc_host_frame_return`), PSRAM frame buffers, user-provided
  frame buffers (v2.4.0+), overflow/underflow handling.
- `recipes/host_uac.md` — UAC host (USB audio): 12-step lifecycle, TX/RX logical
  devices, alt-setting format query, start/stop, suspend/resume, volume/mute
  (normalized + 1/256 dB), RX_DONE/TX_DONE water level, Kconfig tuning
  (`CONFIG_UAC_NUM_ISOC_URBS`, `CONFIG_UAC_NUM_PACKETS_PER_URB`).
- `recipes/device_ncm_net.md` — NCM / RNDIS Ethernet-over-USB device:
  `tinyusb_net_init` ordering (before `tinyusb_driver_install`), RX/free-TX/init
  callbacks, sync/async send, NTB buffer tuning, v2 API migration
  (removed `TINYUSB_USBDEV_0`).
- `SKILL.md`: new recipes added to the Device and Host scenario quick-reference
  tables; metadata.version bumped 1.0.0 → 1.1.0.
- `resources/api_reference.md`: replaced the "UVC/UAC advanced, not detailed"
  placeholder with real verified signatures for `uvc_host_*`, `uac_host_*`, and
  `tinyusb_net_*` (structs, enums, callback typedefs, function prototypes).

### Grounding
- UVC: `host/class/uvc/usb_host_uvc/include/usb/uvc_host.h`,
  `README.md`, `docs/arch_notes.md`, `docs/FAQ.md`;
  example `examples/basic_uvc_stream/main/basic_uvc_stream.c`.
- UAC: `host/class/uac/usb_host_uac/include/usb/uac_host.h`, `Kconfig`,
  `README.md`; example `examples/audio_player/main/main.c`.
- NCM: `device/esp_tinyusb/include/tinyusb_net.h`, `Kconfig` (NET menu),
  `docs/device/migration-guides/v2/tinyusb_ncm.md`;
  test app `test_apps/ncm/main/test_ncm.c`.

## [1.0.0] - 2026-06-18

Initial release of `esp-usb-skill`, grounded in the Espressif `esp-usb` repository.

### Added
- `SKILL.md` with 12 core principles, When-to-Use, scenario quick-reference tables,
  chip/component support reference, USB Host Library lifecycle map, 12 critical
  pitfalls (each with WRONG/CORRECT code), execution workflow, and failure strategies.
- `AGENTS.md` covering project context, IDF Component Manager dependencies, include
  patterns, standard device/host project structure, canonical entry patterns, build
  workflow, code-generation checklist, and Do-Not-Modify notes.
- `recipes/` (10 scenario recipes, Chinese prose): device CDC-ACM serial, device MSC
  storage, device console/VFS, device composite (CDC+MSC), device external PHY
  (ESP32-S3), device install/uninstall + VBUS, host library basic, host CDC-ACM,
  host MSC, host HID. Each with trigger intents, prerequisites table, step-by-step
  real code, and a common-errors table.
- `resources/api_reference.md` — real function signatures grouped by module
  (esp_tinyusb device, USB Host Library, CDC-ACM/MSC/HID host drivers).
- `resources/config_reference.md` — real `CONFIG_TINYUSB_*` (device) and
  `CONFIG_USB_HOST_*` (host) Kconfig options with defaults and notes.
- `resources/pitfalls.md` — consolidated device/host/cross-cutting gotchas.
- `resources/example_list.md` — index of real in-repo test apps and ESP-IDF
  examples referenced by the docs.
- `README.md` (Chinese) and `CHANGELOG.md`.

### Grounding
- Device APIs verified against `device/esp_tinyusb/include*` headers
  (`tinyusb.h`, `tinyusb_default_config.h`, `tinyusb_cdc_acm.h`, `tinyusb_msc.h`,
  `tinyusb_console.h`, `vfs_tinyusb.h`, `tusb_config.h`).
- Host Library API verified against `host/usb/include/usb/usb_host.h`.
- Host class-driver APIs verified against
  `host/class/{cdc,msc,hid,uvc,uac}/usb_host_*/include/usb/*.h`.
- Kconfig verified against `device/esp_tinyusb/Kconfig` and `host/usb/Kconfig`.
- Code snippets adapted from `device/esp_tinyusb/test_apps/{cdc,msc_storage,
  dconn_detection}/main/*.c`.
- Component versions from each `idf_component.yml`.
