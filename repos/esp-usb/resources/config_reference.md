# ESP-USB Configuration (Kconfig / menuconfig) Reference

Real menuconfig options from `device/esp_tinyusb/Kconfig` and `host/usb/Kconfig`. Set via `idf.py menuconfig` or `sdkconfig.defaults`. Names not listed here do not exist in this repo — do not invent them.

---

## Device Stack — `CONFIG_TINYUSB_*`

Path in menuconfig: **Component config → TinyUSB Stack**

### General / DCD

| Option | Type | Default | Notes |
|---|---|---|---|
| `CONFIG_TINYUSB_DEBUG_LEVEL` | int 0–3 | 1 | TinyUSB log verbosity |
| `CONFIG_TINYUSB_MODE_DMA` | choice | selected | Buffer DMA mode (vs `TINYUSB_MODE_SLAVE` Slave/IRQ) |
| `CONFIG_TINYUSB_MODE_SLAVE` | choice | — | Slave/IRQ DCD mode |

### Callbacks

| Option | Type | Default | Notes |
|---|---|---|---|
| `CONFIG_TINYUSB_SUSPEND_CALLBACK` | bool | n | esp_tinyusb 提供 `tud_suspend_cb` 强定义，派发 `TINYUSB_EVENT_SUSPENDED`。开启后应用**不得**自定义 `tud_suspend_cb` |
| `CONFIG_TINYUSB_RESUME_CALLBACK` | bool | n | 同上，对应 `tud_resume_cb` 与 `TINYUSB_EVENT_RESUMED` |

### Descriptor defaults

| Option | Type | Default | Notes |
|---|---|---|---|
| `CONFIG_TINYUSB_DESC_USE_ESPRESSIF_VID` | bool | y | 用 Espressif VID `0x303A`（`TINYUSB_ESPRESSIF_VID`） |
| `CONFIG_TINYUSB_DESC_CUSTOM_VID` | hex | 0x1234 | depends on `!USE_ESPRESSIF_VID` |
| `CONFIG_TINYUSB_DESC_USE_DEFAULT_PID` | bool | y | 默认 PID 0x4000…0x4007 |
| `CONFIG_TINYUSB_DESC_CUSTOM_PID` | hex | 0x5678 | depends on `!USE_DEFAULT_PID` |
| `CONFIG_TINYUSB_DESC_BCD_DEVICE` | hex | 0x0100 | 固件版本 |
| `CONFIG_TINYUSB_DESC_MANUFACTURER_STRING` | string | "Espressif Systems" | |
| `CONFIG_TINYUSB_DESC_PRODUCT_STRING` | string | "Espressif Device" | |
| `CONFIG_TINYUSB_DESC_SERIAL_STRING` | string | "123456" | |
| `CONFIG_TINYUSB_DESC_CDC_STRING` | string | "Espressif CDC Device" | depends on `TINYUSB_CDC_ENABLED` |
| `CONFIG_TINYUSB_DESC_MSC_STRING` | string | "Espressif MSC Device" | depends on `TINYUSB_MSC_ENABLED` |

> 自定义描述符应在 `tinyusb_driver_install()` 时注入 `tinyusb_config_t.descriptor`，置 `NULL` 才使用上述 menuconfig 默认值。

### Mass Storage (MSC)

| Option | Type | Default | Notes |
|---|---|---|---|
| `CONFIG_TINYUSB_MSC_ENABLED` | bool | n | **必须开**才能用 MSC 设备 API |
| `CONFIG_TINYUSB_MSC_BUFSIZE` | int | 512 (S2/S3/H4) / 8192 (P4/S31) | FS 范围 64–8192；HS 范围 64–32768。对吞吐影响最大 |
| `CONFIG_TINYUSB_MSC_MOUNT_PATH` | string | "/data" | MSC 默认挂载路径 |

### CDC

| Option | Type | Default | Notes |
|---|---|---|---|
| `CONFIG_TINYUSB_CDC_ENABLED` | bool | n | **必须开**才能 include `tinyusb_cdc_acm.h`（否则 `#error`） |
| `CONFIG_TINYUSB_CDC_COUNT` | int 1–2 | 1 | 独立串口数 |
| `CONFIG_TINYUSB_CDC_RX_BUFSIZE` | int 64–10000 | 512 | 须 ≥ `CDC_EP_BUFSIZE` |
| `CONFIG_TINYUSB_CDC_TX_BUFSIZE` | int | 512 | |
| `CONFIG_TINYUSB_CDC_EP_BUFSIZE` | int | 512 | 底层端点缓冲，设 8192 性能最佳，再大收益很小 |

### MIDI / HID / DFU / BTH / NET / Vendor

| Option | Type | Default | Notes |
|---|---|---|---|
| `CONFIG_TINYUSB_MIDI_COUNT` | int 0–2 | 0 | >0 启用 MIDI |
| `CONFIG_TINYUSB_HID_COUNT` | int 0–4 | 0 | >0 启用 HID |
| `CONFIG_TINYUSB_DFU_MODE_DFU` / `_DFU_RUNTIME` / `_NONE` | choice | NONE | DFU 模式 |
| `CONFIG_TINYUSB_DFU_BUFSIZE` | int | 512 | depends on `DFU_MODE_DFU` |
| `CONFIG_TINYUSB_BTH_ENABLED` | bool | n | 蓝牙 host class |
| `CONFIG_TINYUSB_BTH_ISO_ALT_COUNT` | int | 0 | depends on BTH |
| `CONFIG_TINYUSB_NET_MODE_ECM_RNDIS` / `_NCM` / `_NONE` | choice | NONE | 网络模式 |
| `CONFIG_TINYUSB_VENDOR_COUNT` | int 0–2 | 0 | >0 启用 vendor 类 |
| `CONFIG_TINYUSB_VENDOR_RX_BUFSIZE` | int 0–32768 | 512 (P4/S31) / 64 | 设 0 禁用内部缓冲 |
| `CONFIG_TINYUSB_VENDOR_TX_BUFSIZE` | int 0–32768 | 512 (P4/S31) / 64 | |
| `CONFIG_TINYUSB_VENDOR_EPSIZE` | int 64–32768 | 512 (P4/S31) / 64 | FS 最小 64，HS 最小 512 |
| `CONFIG_TINYUSB_NCM_OUT_NTB_BUFFS_COUNT` / `IN` | int 1–6 | 3 | NCM NTB 缓冲数 |
| `CONFIG_TINYUSB_NCM_OUT_NTB_BUFF_MAX_SIZE` / `IN` | int 1600–10240 | 3200 | NTB 缓冲大小（4 的倍数） |

---

## Host Library — `CONFIG_USB_HOST_*`

Path in menuconfig: **Component config → USB-OTG**（depends on `SOC_USB_OTG_SUPPORTED`）

| Option | Type | Default | Notes |
|---|---|---|---|
| `CONFIG_USB_HOST_CONTROL_TRANSFER_MAX_SIZE` | int | 256 | 控制端点最大传输（配置描述符上限受其约束） |
| `CONFIG_USB_HOST_HW_BUFFER_BIAS_BALANCED` / `_IN` / `_PERIODIC_OUT` | choice | BALANCED | 硬件 FIFO 偏置 |
| `CONFIG_USB_HOST_DEBOUNCE_DELAY_MS` | int | 250 | 连接去抖（规范 ≥100ms） |
| `CONFIG_USB_HOST_RESET_HOLD_MS` | int | 30 | 复位保持（规范 ≥10ms） |
| `CONFIG_USB_HOST_RESET_RECOVERY_MS` | int | 30 | 复位恢复 |
| `CONFIG_USB_HOST_SET_ADDR_RECOVERY_MS` | int | 10 | SetAddress 恢复 |
| `CONFIG_USB_HOST_RESUME_HOLD_MS` | int | 30 | 恢复保持 |
| `CONFIG_USB_HOST_RESUME_RECOVERY_MS` | int | 20 | 恢复恢复 |
| `CONFIG_USB_HOST_SUSPEND_ENTRY_MS` | int | 20 | 挂起进入延迟 |
| `CONFIG_USB_HOST_HUBS_SUPPORTED` | bool | — | 启用外部 Hub 支持 |
| `CONFIG_USB_HOST_HUB_MULTI_LEVEL` | bool | — | 多级 Hub |
| `CONFIG_USB_HOST_EXT_PORT_CUSTOM_POWER_ON_DELAY_ENABLE` | bool | n | 自定义 PwrOn2PwrGood |
| `CONFIG_USB_HOST_EXT_PORT_CUSTOM_POWER_ON_DELAY_MS` | int | — | 下游端口上电稳定延时 |
| `CONFIG_USB_HOST_EXT_PORT_RESET_RECOVERY_DELAY_MS` | int | 30 | 下游端口复位恢复 |
| `CONFIG_USB_HOST_AUTO_PM_LIGHT_SLEEP` | bool | n | 轻睡眠前自动挂起 root port |
| `CONFIG_USB_HOST_ENABLE_ENUM_FILTER_CALLBACK` | bool | — | 启用枚举过滤回调（多配置选择） |

### Light sleep 相关前置（ESP-IDF，非本仓库但常需配合）

`CONFIG_ESP_SLEEP_EVENT_CALLBACKS`、`CONFIG_PM_ENABLE`、`CONFIG_FREERTOS_USE_TICKLESS_IDLE`。

---

## Pin / Peripheral Notes (from docs)

| 目标 | FS PHY D+/D- GPIO | HS PHY | 说明 |
|---|---|---|---|
| ESP32-S2/S3 | DP=20, DM=19 | — | USB-OTG 与 USB-Serial-JTAG 共用一个 PHY（S3） |
| ESP32-P4 | DP=27, DM=26 | 专用 HS 引脚 | 两个 OTG：1 HS + 1 FS，可同时做主机 |
| ESP32-S31 | （单口，本身 HS） | — | 单端口 HS |
| ESP32-H4 | DP=22, DM=21 | — | FS |

> 多数带两个 USB 口的开发板，标 "USB" 的口已连到 D+/D-。
