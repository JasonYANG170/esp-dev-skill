# ESP-USB Consolidated Pitfalls

Single place for the most common errors across device and host stacks. Each entry lists the symptom, root cause, and fix. The `SKILL.md` "Critical Pitfalls" section has the WRONG/CORRECT code pairs for these.

---

## Device Stack (esp_tinyusb)

### D1. Class header hard-errors at compile time
- **Symptom**: `#error "TinyUSB CDC driver must be enabled in menuconfig"`
- **Cause**: Including `tinyusb_cdc_acm.h` while `CONFIG_TINYUSB_CDC_ENABLED` is off (same pattern for MSC/HID via their Kconfig knobs).
- **Fix**: Enable the matching `CONFIG_TINYUSB_*` option in menuconfig before using the API.

### D2. Random/broken init from bare `tinyusb_config_t`
- **Symptom**: Unpredictable behavior, PHY/task/descriptor fields uninitialized.
- **Cause**: Declaring `tinyusb_config_t tusb_cfg;` without an initializer.
- **Fix**: Always start from `tinyusb_config_t tusb_cfg = TINYUSB_DEFAULT_CONFIG();` and override only needed fields. `tinyusb_default_config.h` must be included to use the macro.

### D3. High-speed chip missing qualifier / high_speed_config
- **Symptom**: Enumeration failures or USB 2.0 spec violation on ESP32-P4 / ESP32-S31.
- **Cause**: Only `full_speed_config` provided on a HS-capable target.
- **Fix**: Provide `full_speed_config`, `high_speed_config`, **and** `qualifier` (or rely on built-in defaults by leaving `NULL`).

### D4. Composite device mis-enumerated (especially Windows)
- **Symptom**: Host sees only one function, or refuses the device.
- **Cause**: Device descriptor uses `bDeviceClass = 0x00` for an IAD composite.
- **Fix**: Set `bDeviceClass = TUSB_CLASS_MISC`, `bDeviceSubClass = MISC_SUBCLASS_COMMON`, `bDeviceProtocol = MISC_PROTOCOL_IAD`.

### D5. Blocking `tinyusb_cdcacm_write_flush` inside a CDC callback
- **Symptom**: Flush blocks for the full timeout inside `callback_rx` etc.
- **Cause**: TinyUSB defers endpoint flushing until callback processing completes.
- **Fix**: Inside callbacks use `tinyusb_cdcacm_write_flush(itf, 0)` (non-blocking); flush with a timeout from another task.

### D6. External PHY overwritten by internal PHY init
- **Symptom**: External PHY configured via `usb_new_phy()` but enumeration fails.
- **Cause**: `tinyusb_driver_install` re-initializes the internal PHY.
- **Fix**: Set `tinyusb_config_t.phy.skip_setup = true` (device) / `usb_host_config_t.skip_phy_setup = true` (host).

### D7. `tud_suspend_cb` / `tud_resume_cb` linker multiple-definition
- **Symptom**: Link error on those symbols.
- **Cause**: `CONFIG_TINYUSB_SUSPEND_CALLBACK` (or RESUME) is enabled (esp_tinyusb provides a strong def) **and** the app defines its own.
- **Fix**: Disable the Kconfig option and define your own, OR keep the option on and handle `TINYUSB_EVENT_SUSPENDED` in the event callback.

### D8. Self-powered device without VBUS monitoring
- **Symptom**: Device stays enumerated after unplug; USB spec violation.
- **Cause**: `self_powered` left false, no `vbus_monitor_io`.
- **Fix**: `phy.self_powered = true` + `vbus_monitor_io` via divider/comparator; sensing pin must go low within 3 ms of unplug. On P4 HS port, call `gpio_install_isr_service()` before install.

### D9. MSC SD-card backing on unsupported chip
- **Symptom**: Link error / undefined `tinyusb_msc_new_storage_sdmmc`.
- **Cause**: ESP32-S2 / ESP32-H4 lack SDMMC host (`SOC_SDMMC_HOST_SUPPORTED` is false), so the function is `#ifdef`-ed out.
- **Fix**: Use `tinyusb_msc_new_storage_spiflash` on those chips.

### D10. Low MSC throughput
- **Symptom**: Sub-MB/s writes, especially internal SPI flash.
- **Cause**: Small `CONFIG_TINYUSB_MSC_BUFSIZE`; internal flash suspends code during writes.
- **Fix**: Raise `CONFIG_TINYUSB_MSC_BUFSIZE` (up to 8192 FS / 32768 HS); use SD card on S3/P4/S31 for sustained throughput.

### D11. Endpoint/address collisions in composite descriptors
- **Symptom**: One function misbehaves or fails enumeration.
- **Cause**: Two class descriptors reuse the same endpoint address.
- **Fix**: Each endpoint address must be unique; notify endpoints are IN (`0x8x`).

---

## Host Library (usb component)

### H1. Calling any API before `usb_host_install`
- **Symptom**: `ESP_ERR_INVALID_STATE` or crash.
- **Cause**: Host Library not installed yet.
- **Fix**: Install once first; it must precede every other call.

### H2. No device ever reaches `USB_HOST_CLIENT_EVENT_NEW_DEV`
- **Symptom**: Client task idles; device ignored.
- **Cause**: Daemon Task not running `usb_host_lib_handle_events` (it drives enumeration).
- **Fix**: Ensure a task repeatedly calls `usb_host_lib_handle_events(portMAX_DELAY, &flags)`.

### H3. Blocking inside client / transfer callbacks
- **Symptom**: Other client events starve; transfer callbacks stall.
- **Cause**: Callbacks run inside `usb_host_client_handle_events`; blocking there blocks all events.
- **Fix**: Set flags in callbacks; do heavy work in the client task loop.

### H4. Transfer submitted to an un-claimed interface
- **Symptom**: Submit fails or undefined behavior.
- **Cause**: Interface never claimed via `usb_host_interface_claim`.
- **Fix**: Open → claim interface → submit transfers.

### H5. Uninstall fails with `ESP_ERR_INVALID_STATE`
- **Symptom**: `usb_host_uninstall` rejects.
- **Cause**: Devices still allocated / clients still registered.
- **Fix**: Deregister all clients, call `usb_host_device_free_all()` until `USB_HOST_LIB_EVENT_FLAGS_ALL_FREE`, then uninstall.

### H6. Two clients claiming the same interface
- **Symptom**: Second claim fails.
- **Cause**: An interface can be claimed by only one client.
- **Fix**: Different clients use different interfaces of the same device (EP0 control transfers are serialized automatically).

### H7. Light sleep with USB host misbehaves
- **Symptom**: Bus looks suspended; devices confused after wake.
- **Cause**: SoC gates USB clocks during light sleep without a spec-compliant suspend.
- **Fix**: Enable `CONFIG_USB_HOST_AUTO_PM_LIGHT_SLEEP` + `CONFIG_ESP_SLEEP_EVENT_CALLBACKS`/`CONFIG_PM_ENABLE`/`CONFIG_FREERTOS_USE_TICKLESS_IDLE`; root port auto-suspends before sleep and resumes via `usb_host_lib_root_port_resume` or transfer submission.

### H8. Deep sleep breaks USB
- **Symptom**: After deep-sleep wake, devices gone.
- **Cause**: USB OTG peripheral powered down; enumeration state not persistent.
- **Fix**: After wake, re-`usb_host_install`, re-register clients, wait for `NEW_DEV`. Switch VBUS off in hardware for lowest current.

---

## Cross-cutting

### X1. Wrong component / dependency
- **Symptom**: Missing headers, undefined APIs.
- **Cause**: Confusing device vs host, or the low-level `usb` vs a class driver.
- **Fix**: Device → `esp_tinyusb`. Low-level host → `usb`. CDC/MSC/HID/UVC/UAC host → the matching `usb_host_*` component (they pull `usb` themselves).

### X2. ESP-IDF too old
- **Symptom**: Build errors against the host components.
- **Cause**: Host Library needs ESP-IDF >= 5.5.3.
- **Fix**: Upgrade ESP-IDF (device stack needs >= 5.0).

### X3. Editing managed components in place
- **Symptom**: Changes vanish on next `idf.py reconfigure`.
- **Cause**: Components are downloaded into `managed_components/`.
- **Fix**: Edit only your own `main/` code; report upstream issues.
