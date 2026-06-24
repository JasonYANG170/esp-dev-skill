# ESP-USB Real Examples / Test Apps Index

Two kinds of real examples exist in the `esp-usb` tree:
1. **In-repo test apps** (paths below) — buildable, under `device/esp_tinyusb/test_apps/` and each host component's `test_app/` or `host_test/`.
2. **ESP-IDF examples** referenced from the repo docs (in `esp-idf`, not vendored here) — listed in the second table.

All paths are relative to the repository root.

---

## In-Repository: Device test apps (`device/esp_tinyusb/test_apps/`)

| Path | Description |
|---|---|
| `device/esp_tinyusb/test_apps/cdc/` | CDC-ACM device: dual-CDC + VFS echo, throughput test (`main/test_cdc.c`). Demonstrates `TINYUSB_DEFAULT_CONFIG`, custom descriptors, `tinyusb_cdcacm_init`, `esp_vfs_tusb_cdc_register`. |
| `device/esp_tinyusb/test_apps/dconn_detection/` | Device connection/disconnect detection via VBUS monitor (`main/test_dconn_detection.c`); handles `TINYUSB_EVENT_ATTACHED/DETACHED/SUSPENDED/RESUMED`. |
| `device/esp_tinyusb/test_apps/msc_storage/` | MSC device with SPI-flash **and** SD-card backing (`main/storage_common.c`, `device_common.c`, `test_msc_*.c`); mount/unmount/format, multi-task access. |
| `device/esp_tinyusb/test_apps/ncm/` | NCM (CDC subclass) Ethernet-over-USB device test (`main/test_ncm.c`: `tinyusb_net_init` → `TINYUSB_DEFAULT_CONFIG` → `tinyusb_driver_install` → wait mount → `tinyusb_net_deinit`; `sdkconfig.defaults` sets `CONFIG_TINYUSB_NET_MODE_NCM=y`). |
| `device/esp_tinyusb/test_apps/power_management/` | USB device with light-sleep / power-management integration. |
| `device/esp_tinyusb/test_apps/runtime_config/` | Runtime reconfiguration of the device stack. |
| `device/esp_tinyusb/test_apps/teardown_device/` | Full install/uninstall lifecycle (teardown) of the device stack. |
| `device/esp_tinyusb/test_apps/usb_cv/` | USB Compliance (Chapter 9 / HID / MSC) verification app. (USBCV results under `docs/device/usbcv_results/`.) |
| `device/esp_tinyusb/test_apps/vendor/` | Vendor-specific class device test. |

## In-Repository: Host component test apps

| Path | Description |
|---|---|
| `host/class/cdc/usb_host_cdc_acm/test_app/main/` | CDC-ACM host driver tests (open, TX/RX, vendor-specific, descriptor parsing). |
| `host/class/cdc/usb_host_cdc_acm/host_test/` | CDC-ACM host interaction tests (`descriptors/`, `device_interaction/`, `parsing_tests/`). |
| `host/class/msc/usb_host_msc/test_app/main/` | MSC host driver tests (read/write sectors, VFS). |
| `host/class/msc/usb_host_msc/host_test/main/` | MSC host hardware test app. |
| `host/class/hid/usb_host_hid/test_app/main/` | HID host driver tests. |
| `host/class/hid/usb_host_hid/host_test/main/` | HID host hardware test app. |
| `host/class/uvc/usb_host_uvc/test_app/main/` | UVC host driver tests. |
| `host/class/uac/usb_host_uac/test_app/main/` | UAC host driver tests. |

## In-Repository: Host component runnable examples

| Path | Description |
|---|---|
| `host/class/uvc/usb_host_uvc/examples/basic_uvc_stream/` | UVC camera stream capture (`main/basic_uvc_stream.c`): USB Host Daemon Task, `uvc_host_install`, `uvc_host_get_frame_list`, open/start/stop, frame queue + `uvc_host_frame_return`, `UVC_HOST_DEVICE_DISCONNECTED` handling. Multi-stream + H.265 selectable. |
| `host/class/uvc/usb_host_uvc/examples/camera_display/` | UVC camera frame display demo. |
| `host/class/uac/usb_host_uac/examples/audio_player/` | USB audio host (`main/main.c`, ~534 lines): speaker siren playback via `uac_host_device_write`, microphone capture via `uac_host_device_read`, alt-setting format search, suspend/resume switching, volume/mute control, `UAC_HOST_DRIVER_EVENT_DISCONNECTED` close. |

## In-Repository: Host component sub-components (CDC vendor VCPs + modem)

| Path | Description |
|---|---|
| `host/class/cdc/usb_host_ch34x_vcp/` | CH340/CH34x virtual COM port host support (`include/usb/vcp_ch34x.h`). |
| `host/class/cdc/usb_host_cp210x_vcp/` | CP210x VCP host support (`include/usb/vcp_cp210x.h`). |
| `host/class/cdc/usb_host_ftdi_vcp/` | FTDI FT23x VCP host support (`include/usb/vcp_ftdi.h`). |
| `host/class/cdc/usb_host_vcp/` | Common VCP abstraction. |
| `host/class/cdc/esp_modem_usb_dte/` | CDC-ACM-backed DTE for `esp_modem` (cellular modems); `include/esp_modem_usb_config.h`, `esp_modem_usb_c_api.h`. |

## In-Repository: USB Host Library low-level

| Path | Description |
|---|---|
| `host/usb/include/usb/usb_host.h` | USB Host Library public API (install/uninstall, clients, devices, transfers). |
| `host/usb/include/usb/usb_helpers.h` | Descriptor-parsing helpers. |
| `host/usb/include/usb/usb_types_ch9.h` / `usb_types_ch11.h` / `usb_types_stack.h` | Chapter 9 / hub / stack types. |
| `host/usb/test/target_test/` | Target tests for enum, ext_port, hcd, usb_host layers. |

## Out-of-Repo: ESP-IDF examples (referenced by `docs/en/usb_device.rst` and `usb_host.rst`)

These live in `esp-idf` (github.com/espressif/esp-idf), not in `esp-usb`. Cited here because the repo docs point to them as the canonical application examples.

| ESP-IDF path | Description |
|---|---|
| `examples/peripherals/usb/device/tusb_console` | Log output via USB Serial Device (TinyUSB). |
| `examples/peripherals/usb/device/tusb_serial_device` | USB Serial Device (configurable as double serial). |
| `examples/peripherals/usb/device/tusb_midi` | USB MIDI Device. |
| `examples/peripherals/usb/device/tusb_hid` | USB keyboard + mouse (square trajectory). |
| `examples/peripherals/usb/device/tusb_msc` | Mass Storage Device (SPI Flash + SD MMC). |
| `examples/peripherals/usb/device/tusb_composite_msc_serialdevice` | Composite: MSC + Serial Device. |
| `examples/peripherals/usb/device/tusb_ncm` | NCM Ethernet-over-USB (Wi-Fi passthrough). |
| `examples/peripherals/usb/host/usb_host_lib` | Bare USB Host Library (install, client, device print, reconnect). |
| `examples/peripherals/usb/host/cdc` | CDC-ACM host (incl. CP210x/FTDI/CH34x). |
| `examples/peripherals/usb/host/msc` | MSC host (read/write/format a USB flash drive). |
| `examples/peripherals/usb/host/hid` | HID host (keyboard/mouse report scanning). |
| `examples/peripherals/usb/host/uvc` | UVC host (USB camera frames). |

> Note: the separate `type_c/usb_tcpm` component (USB Type-C / USB-PD / TCPM, e.g. FUSB302) is **not** part of the CDC/MSC/HID device/host class scope of this skill.
