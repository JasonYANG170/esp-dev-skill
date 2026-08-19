# ESP Development Workflow

Use this reference when writing, porting, or reviewing Espressif firmware or component code.

## Evidence Order

1. User project files: `sdkconfig`, `sdkconfig.defaults`, `idf_component.yml`, `CMakeLists.txt`, `partitions.csv`, `platformio.ini`, Arduino core version, and existing source.
2. The selected ESP repo module under `repos/<repo>/`.
3. Source headers, examples, Kconfig files, and component manifests for the exact version in use.
4. The module's `resources/` and recipe files.
5. Generic ESP-IDF or Arduino knowledge only after version and target compatibility are checked.

## Compatibility Checks

Before generating code, verify:

- `IDF_TARGET` or Arduino board target.
- ESP-IDF, Arduino-ESP32, component, or SDK version.
- Component dependency name and version if using `idf.py add-dependency`.
- SoC feature support: Wi-Fi, BLE, 802.15.4, USB, PSRAM, camera/LCD, LP core, AI/DSP acceleration.
- Kconfig symbol existence in the selected repo/version.

## Code Generation Rules

- Prefer the closest official example and adapt it.
- Keep ESP-IDF, Arduino, ESP8266 RTOS SDK, and standalone component workflows distinct.
- Do not assume a feature exists on every ESP32-family chip.
- Mention required menuconfig/Kconfig, partition, NVS, certificate, model, or managed-component changes with the code.
- When a child repo module gives broad support claims, validate them against the user's exact SoC and version before relying on them.
