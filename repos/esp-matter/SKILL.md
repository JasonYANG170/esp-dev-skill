---
name: esp-matter-skill
description: >-
  AI Skill for developing Matter (over CHIP) firmware on ESP32 using the Espressif esp-matter SDK.
  Used when users need to create, modify, or debug Matter devices, define data models
  (endpoints/clusters/attributes/commands), commission devices with chip-tool, build bridges,
  implement sensors/switches/lighting, configure factory data providers, or enable OTA.
  Trigger words: "esp-matter", "Matter", "CHIP", "commissioning", "fabric", "endpoint",
  "cluster", "attribute", "chip-tool", "matter设备", "入网", "Matter SDK", "connectedhomeip",
  "On/Off Light", "Window Covering", "Door Lock", "data model", "数据模型"
tags:
  - embedded
  - matter
  - CHIP
  - IoT
  - espressif
  - esp32
  - commissioning
  - fabric
  - home-automation
  - zigbee-bridge
  - thread
license: Apache-2.0
compatibility: ESP32 family (esp32, esp32s3, esp32c3, esp32c2, esp32c5, esp32c6, esp32c61, esp32h2, esp32p4); requires ESP-IDF v5.5.4 and the connectedhomeip submodule; built with idf.py
metadata:
  author: Community
  version: "1.1.0"
---

# esp-matter-skill

AI Skill for developing Matter firmware on ESP32 using Espressif's esp-matter SDK. Provides scenario-driven recipes, real API references grounded in the SDK headers and examples, configuration guides, and the common pitfalls that cause non-commissionable or non-working Matter devices.

## Core Principles

1. **Data model is built from a `node`** — Always start with `node::create(&node_config, app_attribute_update_cb, app_identification_cb)`. The node carries root-node device type on endpoint 0; all application endpoints are children.
2. **Device types are created via namespaced `::create()`** — e.g. `extended_color_light::create(node, &config, ENDPOINT_FLAG_NONE, priv_data)`. The available namespaces live in `components/esp_matter/data_model/legacy/esp_matter_endpoint_impl.h` (legacy) or `generated/device_types/` (generated data model).
3. **Attribute changes flow through `app_attribute_update_cb`** — The SDK calls your callback with `callback_type_t` (`PRE_UPDATE`, `POST_UPDATE`, ...). Only act on `PRE_UPDATE` to drive hardware; return `ESP_OK` for attributes you do not handle.
4. **`esp_matter::start(app_event_cb)` must be called after the node and endpoints are created** — Creating the data model before `start()` is mandatory; the node/endpoint objects are plain C++ handles the SDK persists into its database on start.
5. **`nvs_flash_init()` before anything Matter** — The Matter stack reads/writes NVS extensively. Skipping it leads to panics on boot.
6. **Stack lock from non-Matter tasks** — Matter is single-threaded. From app tasks/timers, guard upstream Matter API calls with `esp_matter::lock::ScopedChipStackLock` (RAII). The cleaner pattern for sensor drivers is `chip::DeviceLayer::SystemLayer().ScheduleLambda(...)` to run on the Matter thread.
7. **Custom providers are set before `esp_matter::start()`** — `set_custom_dac_provider()`, `set_custom_commissionable_data_provider()`, `set_custom_device_instance_info_provider()`, `set_custom_device_info_provider()` have no effect if called after start.
8. **Factory/Test provider choice is a Kconfig `choice`** — DAC, Commissionable, Device Instance Info and Device Info providers each have a `choice` under "Component config → ESP Matter". Mixing e.g. `CONFIG_ENABLE_TEST_SETUP_PARAMS=y` with a Factory provider breaks commissioning.
9. **Endpoint flags are bitmaps** — `ENDPOINT_FLAG_NONE` (0) for a normal endpoint. Dynamic/bridged endpoints are added at runtime via `endpoint::resume()`/`aggregator::create()` patterns.
10. **cluster/attribute namespaces mirror the Matter spec** — `cluster::on_off::create()`, `cluster::temperature_measurement::create()`, `cluster::attribute::create_on_off()`, etc. Names come verbatim from `esp_matter_cluster_impl.h` / `esp_matter_attribute_impl.h`.
11. **`priv_data` is your handle back to the driver** — The `void *priv_data` you pass to `<device_type>::create()` is delivered to `app_attribute_update_cb`, so store your `led_driver_handle_t`/bridged-device pointer there.
12. **chip-tool interactive mode is the canonical test path** — `chip-tool interactive start` then `pairing ble-wifi ...` / `onoff toggle <node> <endpoint>`. Default onboarding values for the test build: passcode `20202021`, discriminator `3840`.
13. **A controller/commissioner firmware is a distinct client-only shape** — `CONFIG_ESP_MATTER_CONTROLLER_ENABLE=y` + `CONFIG_ESP_MATTER_ENABLE_MATTER_SERVER=n`; `matter_controller_client::get_instance().init(node_id, fabric_id, port)` then `setup_commissioner()` are called **after** `esp_matter::start()` and **inside** `ScopedChipStackLock`. All control happens via the `matter esp controller ...` shell (pairing / invoke-cmd / read-attr / write-attr / subs-attr / group-settings). Attestation Trust Store (Test/Spiffs/DCL/Custom) and NOC Issuer (Test/Custom) are Kconfig `choice`s under "ESP Matter Controller".

## When to Use

**Applicable:**
- Creating a new Matter device (light, switch, sensor, door lock, window covering, fan, thermostat, etc.)
- Defining or extending a data model (endpoints, clusters, attributes, commands, events)
- Adding a custom cluster / custom attribute / custom command
- Commissioning and controlling a device with chip-tool or the on-device controller console
- Bridging non-Matter devices (Zigbee, BLE Mesh, ESP-NOW, RainMaker) into Matter
- Configuring factory data providers, DAC, and attestation for production
- Matter OTA (plain or encrypted) requestor setup
- RAM/Flash optimization of a Matter firmware

**Not applicable:**
- Raw ESP-IDF questions with no Matter involvement (use the ESP-IDF docs directly)
- Pure connectedhomeip / upstream Matter SDK development on non-ESP32 platforms
- Designing the BLE/Thread/Wi-Fi radio drivers themselves (these are platform layers)
- PCB or antenna design

---

## Scenario Quick Reference (Recipes)

When the user's intent matches a scenario below, **read the corresponding recipe first** — it contains the real call chain, step-by-step instructions, common errors, and code adapted from the repo's own examples.

### Project & Build

| recipe | scenario |
|---|---|
| `recipes/setup_and_build.md` | Clone esp-matter + connectedhomeip, set IDF target, build & flash the light example |
| `recipes/device_data_model.md` | Build a device data model: node, device-type endpoint, cluster, attribute, command |
| `recipes/custom_cluster.md` | Add a vendor custom cluster (XML + codegen + create API) |

### Commissioning & Control

| recipe | scenario |
|---|---|
| `recipes/commissioning_chiptool.md` | Commission a device with chip-tool (BLE-Wi-Fi / BLE-Thread) and send cluster commands |
| `recipes/controller_ondevice.md` | Build an ESP32-S3 as the Matter controller/commissioner and run `matter esp controller invoke/read/write/subscribe/pairing` from firmware |
| `recipes/device_console.md` | Use the on-device `matter` shell for BLE/Wi-Fi/attribute/bridge control |

### Device Types

| recipe | scenario |
|---|---|
| `recipes/lighting.md` | On/Off / Dimmable / Color Temperature / Extended Color light + LED driver callbacks |
| `recipes/sensors.md` | Temperature / Humidity / Occupancy / Contact sensors + ScheduleLambda reporting |
| `recipes/switches_binding.md` | On/Off switch + Binding cluster to control a bound light |
| `recipes/door_lock_window_covering.md` | Door Lock and Window Covering device types |
| `recipes/hvac_appliances.md` | Thermostat (heating/cooling feature) / Refrigerator + TemperatureControlledCabinet / Room Air Conditioner / Pump |

### Integration & Production

| recipe | scenario |
|---|---|
| `recipes/bridge_zigbee.md` | Bridged Node / Aggregator pattern; bridge a Zigbee device into Matter |
| `recipes/factory_data_attestation.md` | Generate factory partition with `esp-matter-mfg-tool`, flash, configure providers |
| `recipes/matter_ota.md` | Matter OTA requestor + encrypted OTA with `esp_matter_ota_requestor_encrypted_init` |

### Optimization, Thread & Low-Power

| recipe | scenario |
|---|---|
| `recipes/optimizations.md` | RAM/Flash optimization Kconfig guide (chip-shell, dynamic endpoints, newlib nano, BLE NimBLE, IRAM→flash, task stacks, cluster exclusion, LTO, SPIRAM) with measured before/after table |
| `recipes/thread_border_router.md` | Build a Thread Border Router on the ESP Thread Border Router board (ESP32-S3 + ESP32-H2 RCP); ThreadBorderRouterManagement dataset + commission Thread end-devices |
| `recipes/icd_device.md` | Intermittently Connected Device (SIT/LIT) on ESP32-H2 / ESP32-C6; polling/idle/active Kconfig + power management + LIT client-registration rule |

---

## Supported Chips & Radio Matrix

The SDK supports the ESP32 family; the radio available determines whether you build over Wi-Fi, Thread, or both. Set with `idf.py set-target <chip>`.

| Target chip | Wi-Fi | Thread (802.15.4) | Notes |
|---|---|---|---|
| esp32 | yes | no | Default target; devkit-c |
| esp32s3 | yes | no | |
| esp32c3 | yes | no | devkit-m |
| esp32c2 | yes | no | |
| esp32c5 | yes | yes | Set `CONFIG_OPENTHREAD_ENABLED` for Thread |
| esp32c6 | yes | yes | Dual-radio; choose Wi-Fi **or** Thread per build |
| esp32c61 | yes | yes | IDF v5.5.4+ |
| esp32h2 | no | yes | Thread-only; ECDSA peripheral for DS attestation |
| esp32p4 | via esp_hosted slave (e.g. esp32c6) | via esp_hosted | Needs ESP32-P4 Function EV Board + ESP-Prog |

> On esp32c5/esp32c6 to switch between Wi-Fi and Thread you toggle `CONFIG_OPENTHREAD_ENABLED`, `CONFIG_ENABLE_WIFI_STATION`, and `CONFIG_USE_MINIMAL_MDNS` together (see `developing.rst`).

---

## Standard Device-Type → `endpoint::<name>::create()` Reference

These namespaces exist in the legacy data model (`CONFIG_ESP_MATTER_ENABLE_GENERATED_DATA_MODEL=n`, the default). All take `(node_t *node, <name>::config_t *config, uint8_t flags, void *priv_data)` and return `endpoint_t *`.

| Device type | Namespace | Typical clusters added |
|---|---|---|
| Root Node (auto on endpoint 0) | `node::create()` | root-node clusters |
| On/Off Light | `endpoint::on_off_light` | OnOff |
| Dimmable Light | `endpoint::dimmable_light` | OnOff, LevelControl |
| Color Temperature Light | `endpoint::color_temperature_light` | OnOff, LevelControl, ColorControl |
| Extended Color Light | `endpoint::extended_color_light` | OnOff, LevelControl, ColorControl (HS+XY+CT) |
| On/Off Light Switch | `endpoint::on_off_light_switch` | OnOff, Binding |
| Dimmer Switch | `endpoint::dimmer_switch` | |
| Color Dimmer Switch | `endpoint::color_dimmer_switch` | |
| Generic Switch | `endpoint::generic_switch` | Switch cluster |
| Fan | `endpoint::fan` | FanControl |
| Door Lock | `endpoint::door_lock` | DoorLock |
| Window Covering | `endpoint::window_covering_device` | WindowCovering (constructor takes `EndProductType`) |
| Temperature Sensor | `endpoint::temperature_sensor` | TemperatureMeasurement |
| Humidity Sensor | `endpoint::humidity_sensor` | RelativeHumidityMeasurement |
| Occupancy Sensor | `endpoint::occupancy_sensor` | OccupancySensing |
| Contact Sensor | `endpoint::contact_sensor` | BooleanState |
| Pump | `endpoint::pump` (ctor: max pressure/speed/flow) | OnOff, PumpConfigurationAndControl |
| Pump Controller | `endpoint::pump_controller` | |
| Aggregator | `endpoint::aggregator` | (container for dynamic bridged endpoints) |
| Bridged Node | `endpoint::bridged_node` | (one per bridged device) |

> The generated data model (`CONFIG_ESP_MATTER_ENABLE_GENERATED_DATA_MODEL=y`, experimental) adds many more device types under `generated/device_types/`. See `SUPPORTED_DEVICE_TYPES.md` for the full list.

---

## Critical Pitfalls (Must Read)

These are the most common errors. Violating any of these produces a device that will not boot, will not commission, or will not report.

### 1. `app_attribute_update_cb` must return `ESP_OK` for unhandled attributes

```cpp
// ❌ WRONG — returning an error for attributes you don't care about aborts the write
esp_err_t app_attribute_update_cb(callback_type_t type, uint16_t endpoint_id, uint32_t cluster_id,
                                  uint32_t attribute_id, esp_matter_attr_val_t *val, void *priv_data) {
    if (cluster_id == OnOff::Id) { ... }
    return ESP_ERR_NOT_SUPPORTED;  // breaks every other attribute on the endpoint
}

// ✅ CORRECT — only act on PRE_UPDATE; return ESP_OK otherwise
esp_err_t app_attribute_update_cb(callback_type_t type, uint16_t endpoint_id, uint32_t cluster_id,
                                  uint32_t attribute_id, esp_matter_attr_val_t *val, void *priv_data) {
    esp_err_t err = ESP_OK;
    if (type == PRE_UPDATE) {
        err = app_driver_attribute_update(endpoint_id, cluster_id, attribute_id, val);
    }
    return err;
}
```

### 2. Endpoints must be created **before** `esp_matter::start()`

```cpp
// ❌ WRONG — creating the endpoint after start() will not register it with the server
esp_matter::start(app_event_cb);
endpoint_t *ep = on_off_light::create(node, &cfg, ENDPOINT_FLAG_NONE, nullptr);  // too late

// ✅ CORRECT — node + endpoints first, then start
node_t *node = node::create(&node_config, app_attribute_update_cb, app_identification_cb);
endpoint_t *ep = on_off_light::create(node, &cfg, ENDPOINT_FLAG_NONE, light_handle);
esp_matter::start(app_event_cb);
```

### 3. NVS must be initialized first

```cpp
// ❌ WRONG — Matter panics reading/writing fabric/access storage
extern "C" void app_main() {
    node::create(...);
    esp_matter::start(app_event_cb);
}

// ✅ CORRECT — nvs_flash_init() is the first call
extern "C" void app_main() {
    nvs_flash_init();
    // ... create node, endpoints ...
    esp_matter::start(app_event_cb);
}
```

### 4. Do not touch the Matter stack from an arbitrary task without the lock

```cpp
// ❌ WRONG — calling attribute::update() from a sensor timer task races the Matter thread
static void sensor_timer_cb(TimerHandle_t xTimer) {
    attribute::update(ep_id, TemperatureMeasurement::Id,
                      TemperatureMeasurement::Attributes::MeasuredValue::Id, &val);
}

// ✅ CORRECT — schedule onto the Matter event loop (preferred for sensor drivers)
static void temp_sensor_notification(uint16_t endpoint_id, float temp, void *user_data) {
    chip::DeviceLayer::SystemLayer().ScheduleLambda([endpoint_id, temp]() {
        attribute_t *attr = attribute::get(endpoint_id, TemperatureMeasurement::Id,
                                           TemperatureMeasurement::Attributes::MeasuredValue::Id);
        esp_matter_attr_val_t val;
        attribute::get_val(attr, &val);
        val.val.i16 = static_cast<int16_t>(temp * 100);
        attribute::update(endpoint_id, TemperatureMeasurement::Id,
                          TemperatureMeasurement::Attributes::MeasuredValue::Id, &val);
    });
}
```

For long or blocking work, the explicit lock is `esp_matter::lock::ScopedChipStackLock lock(portMAX_DELAY);`.

### 5. Reading an attribute: get the handle, then `get_val`

```cpp
// ❌ WRONG — there is no attribute::read() returning the value directly
bool onoff = attribute::read(ep_id, OnOff::Id, OnOff::Attributes::OnOff::Id);

// ✅ CORRECT — two steps: get handle, then get_val
attribute_t *attr = attribute::get(ep_id, OnOff::Id, OnOff::Attributes::OnOff::Id);
esp_matter_attr_val_t val;
attribute::get_val(attr, &val);
bool onoff = val.val.b;
```

### 6. Writing back to the model from the driver uses `attribute::update`, not direct mutation

```cpp
// ❌ WRONG — mutating val does not propagate; calling set_val bypasses callbacks
esp_matter_attr_val_t val;
attribute::get_val(attr, &val);
val.val.b = !val.val.b;
attribute::set_val(attr, &val);  // no reporting, no callbacks

// ✅ CORRECT — update() goes through the data model and reports to subscribers
attribute_t *attr = attribute::get(endpoint_id, OnOff::Id, OnOff::Attributes::OnOff::Id);
esp_matter_attr_val_t val;
attribute::get_val(attr, &val);
val.val.b = !val.val.b;
attribute::update(endpoint_id, OnOff::Id, OnOff::Attributes::OnOff::Id, &val);
```

### 7. `priv_data` carries the driver handle to the callback

```cpp
// ❌ WRONG — using a global driver handle for a multi-endpoint device
static led_driver_handle_t g_handle;  // works for 1 light, breaks for N

// ✅ CORRECT — pass priv_data into create(); cast it back in the callback
endpoint_t *ep = extended_color_light::create(node, &cfg, ENDPOINT_FLAG_NONE, (void *)light_handle);
// in callback:
led_driver_handle_t handle = (led_driver_handle_t)priv_data;
```

### 8. Color value remapping — Matter units are not LED units

```cpp
// ❌ WRONG — passing Matter hue (0-254) straight to a 0-360 LED driver
led_driver_set_hue(handle, val->val.u8);

// ✅ CORRECT — use the example's remap macros from app_priv.h
int value = REMAP_TO_RANGE(val->val.u8, MATTER_HUE, STANDARD_HUE);
led_driver_set_hue(handle, value);
// Temperature is in mireds; invert to kelvin:
uint32_t k = REMAP_TO_RANGE_INVERSE(val->val.u16, STANDARD_TEMPERATURE_FACTOR);
```

### 9. Commissioning with test creds requires `CONFIG_ENABLE_TEST_SETUP_PARAMS=y`

```text
# ❌ WRONG — disabled test params but no factory partition flashed → no passcode/discriminator
CONFIG_ENABLE_TEST_SETUP_PARAMS=n
# (device advertises but chip-tool pairing times out)

# ✅ CORRECT — either keep the test provider, or generate+flash a factory partition
# Option A (dev):
CONFIG_ENABLE_TEST_SETUP_PARAMS=y   # passcode 20202021, discriminator 3840
# Option B (prod): use esp-matter-mfg-tool, flash the factory partition,
#   enable CONFIG_ENABLE_ESP32_FACTORY_DATA_PROVIDER and CONFIG_FACTORY_*_PROVIDER
```

### 10. On esp32c5/c6 you must pick Wi-Fi **or** Thread, not both implicitly

```text
# ❌ WRONG — both Wi-Fi station and Thread enabled without the right mdns config
CONFIG_OPENTHREAD_ENABLED=y
CONFIG_ENABLE_WIFI_STATION=y
CONFIG_USE_MINIMAL_MDNS=y   # mdns wrong for a Thread build

# ✅ CORRECT — Thread build
CONFIG_OPENTHREAD_ENABLED=y
CONFIG_ENABLE_WIFI_STATION=n
CONFIG_USE_MINIMAL_MDNS=n
# (for Wi-Fi build invert all three)
```

### 11. Custom providers set after `start()` are ignored

```cpp
// ❌ WRONG — providers configured too late
esp_matter::start(app_event_cb);
set_custom_dac_provider(my_dac);  // no effect

// ✅ CORRECT — set before start()
set_custom_dac_provider(my_dac);
set_custom_commissionable_data_provider(my_comm);
esp_matter::start(app_event_cb);
```

### 12. Encrypted OTA key buffer must outlive the call

```cpp
// ❌ WRONG — local buffer freed after scope
void start_ota() {
    char key[4096];
    fill_key(key);
    esp_matter_ota_requestor_encrypted_init(key, sizeof(key));  // SDK keeps the pointer
}

// ✅ CORRECT — embed the key as a file/extern symbol so its lifetime is the program
extern const char decryption_key_start[] asm("_binary_esp_image_encryption_key_pem_start");
extern const char decryption_key_end[]   asm("_binary_esp_image_encryption_key_pem_end");
static const char *s_decryption_key = decryption_key_start;
static const uint16_t s_decryption_key_len = decryption_key_end - decryption_key_start;
// after esp_matter::start():
esp_matter_ota_requestor_encrypted_init(s_decryption_key, s_decryption_key_len);
```

### 13. Bridged endpoints need an Aggregator (or Root Node) parent

```cpp
// ❌ WRONG — adding a bridged_node endpoint with no parent aggregator
endpoint_t *bridged = bridged_node::create(node, &cfg, ENDPOINT_FLAG_NONE, nullptr);
// (clients won't discover it correctly)

// ✅ CORRECT — create aggregator first (or reuse root node), then resume/add bridged endpoints
aggregator::config_t agg_cfg;
endpoint_t *agg = aggregator::create(node, &agg_cfg, ENDPOINT_FLAG_NONE, nullptr);
// dynamic bridged endpoints are added at runtime via the bridge add path,
// setting parent_endpoint_id to the aggregator
```

### 14. `erase_flash` before first commissioning

```bash
# ❌ WRONG — leftover NVS from a previous fabric prevents clean commissioning
idf.py flash monitor

# ✅ CORRECT — first flash should wipe storage
idf.py erase_flash flash monitor
```

---

## Execution Workflow

| Step | Name | Description |
|------|------|-------------|
| 1 | Understand | Identify the device type(s), radio (Wi-Fi/Thread), and whether bridging is needed |
| 2 | Recipe | Pick the matching recipe in `recipes/`; follow its call chain and code shape |
| 3 | Query APIs | For specific clusters/attributes not in the recipe, look up `resources/api_reference.md` |
| 4 | Validate | Confirm the device-type namespace, cluster namespace, and attribute `::Id` exist in the headers |
| 5 | Confirm plan | Tell the user: includes, node/endpoint creation order, callback wiring, provider choice |
| 6 | Execute | New project: copy the closest example from `examples/` and modify. Existing project: edit in place |
| 7 | Build | `idf.py set-target <chip>` then `idf.py build` (enable `IDF_CCACHE_ENABLE=1`) |
| 8 | Flash | First time: `idf.py erase_flash flash monitor`; later: `idf.py flash monitor` |
| 9 | Commission | `chip-tool interactive start` → `pairing ble-wifi ...` (or `ble-thread`) |
| 10 | Control | `onoff toggle <node> <endpoint>`, `levelcontrol move-to-level ...`, etc. |

### Step 6 Detail — Project Creation Strategy

Copy the closest example directory rather than writing from scratch:

- On/Off, Dimmable, Color Temperature, Extended Color light �� `examples/light/`
- Light switch with binding → `examples/light_switch/`
- Generic switch → `examples/generic_switch/`
- Temperature/Humidity/Occupancy sensors → `examples/sensors/`
- Door lock → `examples/door_lock/`
- Refrigerator / Room AC / Thermostat-style appliances → `examples/refrigerator/`, `examples/room_air_conditioner/`
- Bridging Zigbee / BLE Mesh / ESP-NOW / RainMaker → `examples/bridge_apps/<name>/`
- Controller / commissioner → `examples/controller/`
- Matter OTA Provider host → `examples/ota_provider/`
- Thread Border Router → `examples/thread_border_router/`
- All device types test app → `examples/all_device_types_app/`
- ICD (Intermittently Connected Device) → `examples/icd_app/`
- Managed-component (no full repo) → `examples/managed_component_light/`

Preserve `main/`, `sdkconfig.defaults*`, `partitions.csv`, and the example's `idf_component.yml`.

---

## Failure Strategies

| Situation | Action |
|---|---|
| Device-type `::create` not found | Confirm legacy vs generated data model; check `esp_matter_endpoint_impl.h` namespaces |
| Commissioning times out | Verify `CONFIG_ENABLE_TEST_SETUP_PARAMS=y` (dev) or factory partition flashed; `erase_flash` |
| Attribute not reporting | Did you use `attribute::update()` (not `set_val`)? Is the endpoint enabled? |
| Crash from sensor task | Wrap Matter calls in `ScheduleLambda` or `ScopedChipStackLock` |
| Build too big | Disable unused clusters under "Select Supported Matter Clusters"; see `resources/config_reference.md` |
| esp32p4 Wi-Fi/BLE missing | esp_hosted slave firmware must be flashed to the connected esp32c6 |
| Multiple fabrics / wrong fabric | Open commissioning window with `kDnssdOnly` after `kFabricRemoved` (see `examples/light/app_main.cpp`) |

## References

- Scenario recipes → `recipes/` directory
- Real API signatures → `resources/api_reference.md`
- Kconfig & config options → `resources/config_reference.md`
- Consolidated gotchas → `resources/pitfalls.md`
- Examples index → `resources/example_list.md`
- Matter device lifecycle → `resources/state_machine.md`
