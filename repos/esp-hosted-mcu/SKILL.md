---
name: esp-hosted-mcu-skill
description: >-
  AI Skill for ESP-Hosted-MCU firmware development, where an ESP chip is used as a
  Wi-Fi/Bluetooth/OpenThread/Zigbee communication co-processor driven by a host MCU over
  SDIO/SPI/UART. Used when creating, configuring, building, or debugging ESP-Hosted host
  and co-processor (slave) projects, transport setup, RPC-based Wi-Fi, BT Hosted HCI / VHCI,
  co-processor OTA, GPIO expander, host power save, and transport recovery.
  Trigger words: "ESP-Hosted", "esp-hosted-mcu", "co-processor", "协处理器", "Hosted", "RPC", "SDIO slave", "SPI Full Duplex", "SPI Half Duplex", "esp_hosted", "esp_wifi_remote", "esp_hosted_init", "transport", "slave firmware", "OpenThread RCP", "Hosted HCI", "ESP32-P4 + C6", "Bluedroid", "NimBLE", "TIWT", "iTWT", "GPIO expander"
tags:
  - embedded
  - esp-hosted
  - co-processor
  - wifi
  - bluetooth
  - ble
  - openthread
  - zigbee
  - SDIO
  - SPI
  - UART
  - ESP-IDF
  - RPC
  - firmware
license: Apache-2.0
compatibility: "Host: any ESP chipset (or non-ESP MCU via port layer); Co-processor (slave): ESP32, ESP32-C2/C3/C5/C6/C61, ESP32-S2/S3, ESP32-H2/H4. Build requires ESP-IDF >= 5.3 (idf.py). Transports: SDIO 1/4-bit, SPI Full-Duplex, SPI Half-Duplex (1/2/4-line), UART."
metadata:
  author: Community
  version: "1.1.0"
---

# esp-hosted-mcu-skill

AI Skill for ESP-Hosted-MCU development. ESP-Hosted-MCU lets any host MCU extend an ESP chip into a Wi-Fi/Bluetooth/OpenThread/Zigbee **communication co-processor (slave)**, communicating over SDIO, SPI, or UART via protobuf-encoded RPCs. This skill provides scenario-driven recipes, the real `esp_hosted_*` host API reference, Kconfig/configuration reference, transport pin tables, and the consolidated pitfalls that break a Hosted link. Everything here is grounded in the `esp-hosted-mcu` repository documentation and source headers.

## Core Principles

1. **Host vs Slave are two separate firmware images** — The host runs your app + Wi-Fi/BT host stack; the slave (co-processor) runs ESP-Hosted slave firmware. Both must be built and flashed separately, and both must agree on transport.
2. **`esp_wifi_remote` is the API surface on the host** — On the host you call standard ESP-IDF Wi-Fi APIs (`esp_wifi_init`, `esp_wifi_set_mode`, ...); `esp_wifi_remote` provides weak empty definitions that ESP-Hosted fulfills via RPC. Do not invent custom RPC calls.
3. **Init order is fixed** — `esp_netif_init()` → `esp_event_loop_create_default()` → `esp_hosted_init()` → `esp_hosted_connect_to_slave()` → (wait for `ESP_HOSTED_EVENT_TRANSPORT_UP`) → Wi-Fi/BT init. Reversing breaks the link.
4. **Transport must match on both ends** — If the host selects "SPI Full-duplex", the slave must be configured for the same transport. Mismatch = no INIT event.
5. **Co-processor chip is chosen in Kconfig** — On the host set `ESP_HOSTED_CP_TARGET_*` ("Slave chipset to be used"); on the slave `idf.py set-target` selects the real chip. These must be consistent.
6. **Only RPC (control) packets are protobuf-serialized** — Data packets (network/BT) are not serialized; do not hand-encode them. The interface-type header (`if_type`) multiplexes STA/AP/SERIAL/HCI/PRIV traffic on the same bus.
7. **Disable native Wi-Fi on a Wi-Fi-capable host** — If your host ESP chip has native Wi-Fi (e.g. ESP32-C3), you must disable it in `Kconfig.soc_caps.in`, otherwise `esp_wifi_remote`/`esp_hosted` conflict.
8. **`esp-extconn` and `esp_hosted` are mutually exclusive** — In the host `idf_component.yml`, remove/comment any `espressif/esp-extconn` block (common on ESP32-P4 examples) before adding `esp_hosted`.
9. **External pull-ups are mandatory for SDIO** — CMD, DAT0–DAT3 need ~51 kΩ pull-ups (PCB). Classic ESP32 may additionally need one-time eFuse burning. SDIO is PCB-only; do not jumpers for 4-bit.
10. **BT Controller is disabled by default on the slave (v2.5.2+)** — You must call `esp_hosted_bt_controller_init()`/`enable()` on the host after `esp_hosted_connect_to_slave()` before starting any BT host stack. Set the BT MAC *before* enabling the controller.
11. **Heartbeat + transport events drive recovery** — Register on `ESP_HOSTED_EVENT` to detect `CP_INIT` / `TRANSPORT_FAILURE` / `TRANSPORT_DOWN`; use `esp_hosted_configure_heartbeat()` to liveness-check the co-processor and re-init on failure.
12. **Co-processor OTA reuses the live transport** — After initial serial flashing, slave updates go through `esp_hosted_slave_ota_begin/write/end/activate` over the existing SDIO/SPI/UART link; no extra GPIO/cable needed.
13. **EXT_COEX + BT coexistence depends on slave chip & IDF version** — On advanced-coex chips (`SOC_EXTERNAL_COEX_ADVANCE`: C2/C5/C6/C61/H2/H4/H21) external coexistence works alongside BT/BLE; on others (e.g. S3) the BT controller must be disabled when EXT_COEX is enabled. EXT_COEX with BT enabled currently requires ESP-IDF `master`. Configure the slave PTA remotely from the host via `esp_hosted_cp_ext_coex_*` (no slave firmware change).

## When to Use

**Applicable:**
- Building/bringing up an ESP-Hosted link (host + co-processor) over SDIO, SPI (FD/HD), or UART
- Porting an ESP-IDF Wi-Fi or Bluetooth example to run "over ESP-Hosted" on a host that lacks radio (e.g. ESP32-P4)
- Configuring transport pins/clock/buffers, switching SDIO stream vs packet mode
- Implementing co-processor (slave) OTA from the host
- Using the GPIO expander to control slave GPIOs, or external coexistence (EXT_COEX)
- Host power save (deep sleep) with slave-driven wake-up
- Network Split (host & slave share one IP, port-based routing, MQTT wake-on-packet)
- Wi-Fi 6 iTWT power save on C5/C6 co-processors
- Wi-Fi Easy Connect (DPP) QR-code enrollee onboarding
- Private host↔slave binary channel (peer/custom data, `msg_id`-keyed)
- OpenThread RCP / Zigbee gateway scenarios over a dedicated UART
- Debugging "no INIT event", transport mismatch, heartbeat timeout, IRAM exhaustion

**Not applicable:**
- Native (single-chip) ESP-IDF Wi-Fi/BT development without a co-processor — use ESP-IDF docs directly
- The Linux-host variant — that is the separate `esp-hosted` repo, not `esp-hosted-mcu`
- PCB/schematic design or RF shielding physical design beyond the pin/pull-up facts documented here
- ESP-Hosted version < 2.5.2 BT flows (controller enabled by default) — confirm version first

---

## Scenario Quick Reference (Recipes)

When user intent matches a scenario below, **read the corresponding recipe first** — it contains the full call chain, step-by-step setup, common errors, and grounded code.

### Project Setup & Transport

| recipe | scenario |
|---|---|
| `recipes/bringup_spi_fd.md` | SPI Full-Duplex bring-up: hardware pins, slave + host config, build/flash, verify INIT event |
| `recipes/bringup_sdio.md` | SDIO 1-bit / 4-bit bring-up: pull-ups, eFuse, slot/bus-width config, stream vs packet mode |
| `recipes/bringup_uart.md` | UART transport bring-up: low-throughput Wi-Fi+BT, baud rate, reset GPIO |
| `recipes/transport_config_code.md` | Configure transport at runtime via `esp_hosted_*_set_config()` + `INIT_DEFAULT_HOST_*` macros |

### Host Application

| recipe | scenario |
|---|---|
| `recipes/host_wifi_sta.md` | Run a Wi-Fi STA app on the host over ESP-Hosted (event handlers, `esp_wifi_*`, iperf) |
| `recipes/host_events_recovery.md` | Subscribe to `ESP_HOSTED_EVENT`, heartbeat monitor, transport failure recovery |
| `recipes/host_bt_controller.md` | Initialize/enable the co-processor BT controller, set BT MAC (v2.5.2+ flow) |
| `recipes/host_gpio_expander.md` | Control co-processor GPIOs from the host via `esp_hosted_cp_gpio_*` |
| `recipes/wifi_dpp_enrollee.md` | Wi-Fi Easy Connect (DPP) enrollee: QR-code onboarding without a pre-shared password (`esp_supp_dpp_*`) |

### Advanced Features

| recipe | scenario |
|---|---|
| `recipes/slave_ota.md` | Co-processor OTA over the live transport (`esp_hosted_slave_ota_*`) |
| `recipes/host_power_save.md` | Host deep sleep with slave wake-up (`esp_hosted_power_save_*`, Kconfig) |
| `recipes/network_split.md` | Share one IP between host & slave, route traffic by port (C5/C6/S2/S3), MQTT wake-on-packet |
| `recipes/wifi_itwt.md` | Wi-Fi 6 iTWT power save on C5/C6 (`esp_wifi_sta_itwt_setup`, modem-sleep) |
| `recipes/host_ext_coex.md` | External coexistence: configure slave PTA from host (1/2/3/4-wire, leader/follower) |
| `recipes/peer_custom_data.md` | Private raw-binary host↔slave channel via `esp_hosted_send_custom_data` / `register_custom_callback` |
| `recipes/openthread_zigbee_rcp.md` | OpenThread RCP / Zigbee over dedicated UART (co-processor RCP config) |

---

## Transport Comparison (from README Hosted Transports table)

| Transport | Type | Host GPIOs | Co-processor supported | Typical Host Tx iperf | Remarks |
|:---:|:---:|:---:|:---:|:---:|:---|
| Standard SPI | FD | 6 (+2) | Any_Slave | udp 24 / tcp 22 | Simplest, jumper-friendly |
| 1-bit SPI | HD | 4 (+2) | Any_Slave [1] | udp 22 / tcp 19 | Half duplex, 1 data line |
| Dual SPI | HD | 5 (+2) | Any_Slave [1] | udp 32 / tcp 26 | Better throughput, HD |
| Quad SPI | HD | 7 (+2) | Any_Slave [1] | udp 41 / tcp 29 | PCB mandatory (signal integrity) |
| SDIO 1-Bit | HD | 4 (+2) | ESP32, C6, C5, C61 | — | Stepping stone to 4-bit |
| SDIO 4-Bit | HD | 6 (+2) | ESP32, C6, C5, C61 | udp 79.5 / tcp 53.4 (S) | Highest performance |
| Only BT over UART | FD | 2 or 4 (+2) | Any_Slave | NA | Dedicated HCI over UART |
| UART (Wi-Fi+BT) | FD | 2 (+2) | Any_Slave | udp 0.68 / tcp 0.67 | Low throughput; 921600 baud ref |

[1] 1-bit/Dual/Quad SPI is **not** supported on classic ESP32. `+2` = Reset + Ground (always required). (S)=shield box, (O)=over-the-air. BLE-only on all chips except classic ESP32 (Classic BT + BLE 4.2).

## Co-processor (Slave) Chip Support

| Slave chip | Wi-Fi | Bluetooth | SDIO slave | SPI | UART | Notes |
|---|---|---|---|---|---|---|
| ESP32 | Yes | Classic BT + BLE 4.2 | Yes (IO_MUX only) | Yes | Yes | May need eFuse burn for SDIO |
| ESP32-C2 | Yes | BLE | — | Yes | Yes | SDIO slave not supported |
| ESP32-C3 | Yes | BLE | — | Yes | Yes | SDIO slave not supported |
| ESP32-C5 | Yes | BLE | Yes | Yes | Yes | Wi-Fi 6; iTWT; RCP capable |
| ESP32-C6 | Yes | BLE | Yes | Yes | Yes | Wi-Fi 6; iTWT; RCP capable |
| ESP32-C61 | Yes | BLE | Yes | Yes | Yes | SDIO slave supported |
| ESP32-S2 | Yes | — (no BT) | — | Yes | Yes | No Bluetooth |
| ESP32-S3 | Yes | BLE | SDIO **master** only | Yes | Yes | SDIO slave not supported |
| ESP32-H2 | — (no Wi-Fi) | BLE | — | — | Yes | OpenThread/Zigbee RCP |
| ESP32-P4 | — (no radio) | — | SDIO master | — | — | Host only (no native radio) |

> SDIO **slave** providers: ESP32, ESP32-C6, ESP32-C5, ESP32-C61. SDIO **master** hosts: ESP32, ESP32-S3, ESP32-P4. ESP32-P4 is host-only (no radio) — the canonical pairing is ESP32-P4 (host) + ESP32-C6 (slave) on the ESP32-P4-Function-EV-Board.

## SPI Full-Duplex Pin Map (default Kconfig, host side)

| Signal | ESP32 | ESP32-S2/S3 | ESP32-C2/C3/C5/C6 | ESP32-P4 |
|---|---|---|---|---|
| CLK | 14 | 12 | 6 | 9 |
| MOSI | 13 | 11 | 7 | 8 |
| MISO | 12 | 13 | 2 | 10 |
| CS | 15 | 10 | 10 | 7 |
| Handshake | 26 | 17 | 3 | 6 |
| Data Ready | 4 | 4 | 4 | 11 |
| Reset Out | 5 | 5 | 5 | 12 |

## SDIO Pin Map (host side; +ext pull-up on CMD/D0–D3)

| Signal | ESP32 | ESP32-S3 | ESP32-P4 + C6 (EV board) |
|---|---|---|---|
| CLK | 14 | 19 | 18 |
| CMD | 15 | 47 | 19 |
| D0 | 2 | 13 | 14 |
| D1 | 4 | 35 | 15 |
| D2 | 12 | 20 | 16 |
| D3 | 13 | 9 | 17 |
| Reset Out | 5 | 42 | 54 |

## ESP-Hosted Frame Header (interface multiplexing)

Every frame on the bus carries this header. The `if_type` field selects the payload category.

| Interface Type | Value | Carries |
|---|---|---|
| `ESP_INVALID_IF` | 0 | Invalid |
| `ESP_STA_IF` | 1 | Station (Wi-Fi) frame |
| `ESP_AP_IF` | 2 | SoftAP (Wi-Fi) frame |
| `ESP_SERIAL_IF` | 3 | Control / RPC frame |
| `ESP_HCI_IF` | 4 | Bluetooth Hosted HCI frame |
| `ESP_PRIV_IF` | 5 | Private host↔slave communication |
| `ESP_TEST_IF` | 6 | Transport throughput test |
| `ESP_ETH_IF` | 7 | (invalid) |

---

## Critical Pitfalls (Must Read)

These are the most common errors. Violating any of these will produce a non-working or unstable ESP-Hosted link.

### 1. Init Order Must Be: netif → event loop → hosted init → connect → Wi-Fi

```c
// ❌ WRONG — calling esp_wifi_start before ESP-Hosted transport is up
esp_wifi_init(&cfg);
esp_wifi_start();
esp_hosted_init();            // too late, RPC channel not ready

// ✅ CORRECT
ESP_ERROR_CHECK(esp_netif_init());
ESP_ERROR_CHECK(esp_event_loop_create_default());
// register ESP_HOSTED_EVENT handler here
esp_hosted_init();
esp_hosted_connect_to_slave();
// wait on ESP_HOSTED_EVENT_TRANSPORT_UP (semaphore) before Wi-Fi
xSemaphoreTake(sem_hosted_is_up, portMAX_DELAY);
esp_netif_create_default_wifi_sta();
esp_wifi_init(&cfg);          // weak defs fulfilled by esp_wifi_remote/esp_hosted
```

### 2. Host and Slave Must Select the Same Transport

```c
// ❌ WRONG — host menuconfig: "SPI Full-duplex", but slave menuconfig: "SDIO"
// Result: no INIT event, "Attempt connection with slave: retry[N]" forever

// ✅ CORRECT — both ends same transport
// Host:   Component config -> ESP-Hosted config -> Transport layer = SPI Full-duplex
// Slave:  Example configuration -> Bus Config -> Transport layer = SPI Full-duplex
```

### 3. Remove esp-extconn / disable native Wi-Fi on host

```yaml
# ❌ WRONG — leaving conflicting deps in main/idf_component.yml on ESP32-P4
espressif/esp-extconn:
  version: "~0.1.0"
  rules:
    - if: "target in [esp32p4]"
# esp-extconn and esp_hosted CANNOT coexist

# ✅ CORRECT — comment/remove esp-extconn, then add:
#   idf.py add-dependency "espressif/esp_wifi_remote"
#   idf.py add-dependency "espressif/esp_hosted"
```

For a Wi-Fi-capable host (e.g. ESP32-C3 acting as host), also disable native Wi-Fi:
```c
// ❌ WRONG — leaving native WIFI caps enabled while using esp_hosted
// ✅ CORRECT — edit components/soc/<soc>/include/soc/Kconfig.soc_caps.in
//   and set all WIFI-related configs to 'n'
```

### 4. SDIO Needs External Pull-ups (and classic ESP32 may need eFuse)

```text
// ❌ WRONG — SDIO 4-bit over jumper wires, no pull-ups
//   -> slave enters SPI mode on boot, link never comes up

// ✅ CORRECT
//   - PCB traces, length-matched, < 5 cm
//   - 51 kΩ external pull-ups on CMD, DAT0, DAT1, DAT2, DAT3 (even in 1-bit mode!)
//   - classic ESP32 as slave/master: check eFuse pull-up requirement (one-time, irreversible)
//   - pull-ups still required on DAT2/DAT3 in 1-bit mode to avoid SPI-mode entry
```

### 5. BT Controller Must Be Initialized/Enabled Explicitly (v2.5.2+)

```c
// ❌ WRONG — assuming BT controller is on by default (true only before v2.5.2)
esp_hosted_connect_to_slave();
esp_bt_controller_init(&cfg);  // controller not enabled on slave -> failure

// ✅ CORRECT — enable co-processor controller from host, set MAC first
esp_hosted_connect_to_slave();
uint8_t mac[6];
esp_hosted_iface_mac_addr_get(mac, sizeof(mac), ESP_MAC_BT);   // optional read
// esp_hosted_iface_mac_addr_set(custom_mac, 6, ESP_MAC_BT);   // optional, before enable
esp_hosted_bt_controller_init();
esp_hosted_bt_controller_enable();
// now init BT host stack (NimBLE / BlueDroid)
```

### 6. Wait for TRANSPORT_UP Before Starting Wi-Fi/BT

```c
// ❌ WRONG — racing ahead; RPC may not be wired yet
esp_hosted_init();
esp_hosted_connect_to_slave();
esp_wifi_start();   // can fire before ESP_HOSTED_EVENT_TRANSPORT_UP

// ✅ CORRECT — gate app progress on the transport-up event
// In ESP_HOSTED_EVENT handler, on ESP_HOSTED_EVENT_TRANSPORT_UP:
//     xSemaphoreGive(sem_hosted_is_up);
esp_hosted_init();
esp_hosted_connect_to_slave();
xSemaphoreTake(sem_hosted_is_up, portMAX_DELAY);  // blocks until link is up
```

### 7. SDIO Slave Chip Choices Are Limited

```text
// ❌ WRONG — picking ESP32-C3 or ESP32-S3 as the SDIO *slave*
//   Neither supports SDIO slave mode

// ✅ CORRECT — SDIO slave providers: ESP32, ESP32-C6, ESP32-C5, ESP32-C61
//   SDIO master hosts:  ESP32, ESP32-S3, ESP32-P4
//   For C3/S3 as co-processor, use SPI or UART instead
```

### 8. UART Transport Is Low Throughput — Pick Clock/Baud Deliberately

```text
// ❌ WRONG — expecting multi-Mbit/s over UART at 115200
//   ~0.6 Mbit/s iperf at 921600 baud reference

// ✅ CORRECT
//   - UART only for low-throughput Wi-Fi+BT (Hosted HCI multiplexed)
//   - Reference baud 921600; ESP32-P4+C6 EV board can reach 4 Mbit/s (~3.3 Mbit/s TCP/UDP)
//   - On ESP-IDF v5.5 + UART, IRAM may overflow -> enable
//     "ESP Ringbuf -> Place non-ISR ringbuf functions into flash"
//     (CONFIG_RINGBUF_PLACE_FUNCTIONS_INTO_FLASH=y)
```

### 9. SPI/SDIO Checksum Should Be Enabled

```text
// ❌ WRONG — disabling checksum "for speed" on SPI
//   SPI hardware has no error detection; transient corruption silently breaks RPC

// ✅ CORRECT
//   Host menuconfig: ESP-Hosted config -> SPI Configuration -> Checksum = enable
//   Slave menuconfig: SDIO/SPI Configuration -> Checksum = enable
//   (SDIO has HW CRC, but SPI checksum is strongly recommended)
```

### 10. Custom-Data Callback Signature Changed in v2.12.4

```c
// ❌ WRONG (pre-2.12.4 signature) on >= 2.12.4 firmware
esp_hosted_register_custom_callback(msg_id,
    void (*cb)(uint32_t id, const uint8_t *data, size_t len));  // missing context arg

// ✅ CORRECT (v2.12.4+)
esp_hosted_register_custom_callback(msg_id_exp,
    void (*cb)(uint32_t msg_id_recvd, const uint8_t *data_recvd,
               size_t data_len_recvd, void *local_context),
    local_context);
// Host AND slave must both be >= 2.12.4 to use the new signature
```

### 11. `esp_hosted_*_set_config()` Returns an Error — Check It

```c
// ❌ WRONG — ignoring the warn_unused_result return value
esp_hosted_sdio_set_config(&cfg);   // compiler warns; config may be rejected

// ✅ CORRECT
esp_hosted_transport_err_t ret = esp_hosted_sdio_set_config(&cfg);
if (ret != ESP_TRANSPORT_OK) {
    ESP_LOGE(TAG, "SDIO config rejected: %d", ret);
    return;
}
// then esp_hosted_init() / esp_hosted_connect_to_slave()
```

### 12. Reset Strategy Affects Boot Stability

```text
// ❌ WRONG — host never resets slave, or resets asynchronously to transport bring-up
//   -> stale slave state, spurious INIT events

// ✅ CORRECT — connect Host Reset-Out GPIO to slave EN/RST and let ESP-Hosted
//   reset the slave on connect. Configure reset polarity/strategy in menuconfig:
//   ESP-Hosted config -> Common Slave Reset Strategy
//     = ESP_HOSTED_SLAVE_RESET_ON_EVERY_HOST_BOOTUP (default, most robust)
```

---

## Execution Workflow

| Step | Name | Description |
|------|------|-------------|
| 1 | Plan | Identify host chip, co-processor chip, transport (SPI/SDIO/UART), and BT host stack (NimBLE/BlueDroid). Check chip support table. |
| 2 | Recipe | Pick the matching bring-up recipe (`recipes/bringup_*.md`) and feature recipe. Follow its pin + Kconfig steps exactly. |
| 3 | Slave first | `idf.py create-project-from-example "espressif/esp_hosted:slave"`, `set-target`, menuconfig transport, build, serial-flash. |
| 4 | Host | Take an ESP-IDF Wi-Fi/BT example, add `esp_wifi_remote` + `esp_hosted` deps, remove `esp-extconn`, disable native Wi-Fi if needed, menuconfig transport + slave chip. |
| 5 | Wire | Connect transport pins + Reset + Ground per pin map. SDIO: add pull-ups. Verify with logic analyzer if INIT never arrives. |
| 6 | Build & Flash | `idf.py build` + `idf.py -p <port> flash monitor` on both sides. |
| 7 | Verify | Look for slave "Transport used :: ..." and host "Base transport is set-up" / `ESP_HOSTED_EVENT_TRANSPORT_UP`. |
| 8 | App | Now run Wi-Fi/BT app code on host via `esp_wifi_*` / NimBLE / BlueDroid. |
| 9 | Harden | Add heartbeat monitor + transport-failure recovery (see `recipes/host_events_recovery.md`). |
| 10 | Iterate | Subsequent slave updates via co-processor OTA, not serial. |

### Step 3/4 Detail — Defaulting Config

Generated `sdkconfig`/`sdkconfig.h` are not hand-edited. To persist settings, add lines to `sdkconfig.defaults.<target>` (e.g. `sdkconfig.defaults.esp32p4`) so re-config is automatic.

---

## Failure Strategies

| Situation | Action |
|---|---|
| No INIT event / endless "Attempt connection with slave: retry[N]" | Transport mismatch; verify host and slave menuconfig transport match. Check wiring, Reset GPIO, grounds. Lower SPI/SDIO clock to 5 MHz. |
| `ESP_HOSTED_EVENT_TRANSPORT_FAILURE` recurring | Add heartbeat monitor; on failure call `esp_hosted_deinit()` + re-`init()`/`connect_to_slave()`. Inspect slave reset reason in `ESP_HOSTED_EVENT_CP_INIT`. |
| IRAM overflow building UART slave (IDF v5.5) | Enable `CONFIG_RINGBUF_PLACE_FUNCTIONS_INTO_FLASH=y` (or menuconfig ESP Ringbuf). |
| SDIO link unstable | Confirm 51 kΩ pull-ups on CMD/DAT0–D3; check 3.3 V levels; classic ESP32 — verify eFuse not missing; try SDIO 1-bit to isolate. |
| BT fails to init on slave (v2.5.2+) | Call `esp_hosted_bt_controller_init()`/`enable()` after `esp_hosted_connect_to_slave()`; set MAC before enable. |
| Host Wi-Fi examples conflict on ESP32-P4 | Remove `espressif/esp-extconn` block; ensure `esp_wifi_remote` + `esp_hosted` deps present. |
| Same chip type as host and slave (e.g. two C2) | Apply the "host already has native Wi-Fi" extra disable step (see troubleshooting). |
| Low UART throughput | Expected; switch to SPI/SDIO or raise baud (P4+C6 EV board supports up to 4 Mbit/s). |
| RPC/API not found in docs | Stop and tell the user; only RPCs listed in `docs/implemented_rpcs.md` are real. |

## References

- Scenario recipes → `recipes/` directory
- Host API reference → `resources/api_reference.md`
- Configuration / Kconfig reference → `resources/config_reference.md`
- Consolidated pitfalls → `resources/pitfalls.md`
- Real examples index → `resources/example_list.md`
- Transport sequence / state notes → `resources/state_machine.md`
