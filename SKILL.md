---
name: esp-dev-skill
description: >-
  Use for Espressif ESP32 / ESP8266 firmware and component development across
  ESP-IDF, Arduino-ESP32, ESP8266 RTOS SDK, Matter, Zigbee, Thread, BLE/NimBLE,
  Wi-Fi, ESP-NOW, MQTT, USB, audio, AI, security, and other ESP repositories.
  Routes work to local repo modules, recipes, source, examples, and API references.
license: Apache-2.0
metadata:
  author: Community
  version: "1.1.0"
  compatibility: >-
    Targets ESP32, ESP32-S2/S3, ESP32-C2/C3/C5/C6/C61, ESP32-H2/H4/H21, ESP32-P4, and ESP8266.
    Build via ESP-IDF, Arduino-ESP32, ESP8266 RTOS SDK, or the selected Espressif component workflow.
  tags:
    - embedded
    - espressif
    - esp32
    - esp8266
    - esp-idf
    - arduino
    - firmware
    - wifi
    - bluetooth
    - matter
    - zigbee
---

# esp-dev-skill

Use this skill for Espressif firmware, SDK, and component work. The `repos/` tree contains routed modules for ESP-IDF, Arduino-ESP32, ESP8266 RTOS SDK, Matter/Zigbee/Thread, BLE/NimBLE, MQTT, USB, audio, AI, security, and supporting libraries.

## Route First

1. Identify the exact SoC, board, framework, ESP-IDF/Arduino/component version, protocol or peripheral, and whether the user has an existing project.
2. Read `resources/repo_index.md` to choose the repo module. If uncertain, default to `repos/esp-idf/` for ESP32-series IDF firmware and `repos/ESP8266_RTOS_SDK/` for ESP8266 RTOS firmware.
3. Read `resources/recipe_index.md` only when searching by scenario across repos.
4. For implementation, read the selected `repos/<repo>/SKILL.md`, then the matching recipe and source/example files.
5. Read `resources/development_workflow.md` and `resources/source_strategy.md` before broad searches across `repos/`.

## Evidence Order

Before emitting code or build steps, verify names in this order:

1. User project files: `sdkconfig`, `sdkconfig.defaults`, `idf_component.yml`, `CMakeLists.txt`, `partitions.csv`, `platformio.ini`, Arduino board/core settings, and existing source.
2. Selected repo source, headers, examples, Kconfig files, and component manifests.
3. Selected repo `resources/`.
4. Selected repo recipes.
5. Generic ESP knowledge only after target and version support are checked.

If sources conflict, follow the user's checked-out project and state the version mismatch.

## Coding Rules

- Keep ESP-IDF, Arduino-ESP32, ESP8266 RTOS SDK, and standalone component workflows distinct.
- Do not generalize across ESP32 variants. Wi-Fi, Classic Bluetooth, BLE, 802.15.4, USB, PSRAM, camera/LCD, LP core, and AI/DSP support differ by SoC and version.
- Verify every API, Kconfig symbol, component name, and dependency against the selected version.
- Prefer official examples and managed-component manifests over invented project structure.
- Include required `menuconfig`, partition table, NVS, certificate, model, or component dependency changes with generated code.

## Key References

- `resources/repo_index.md` - repo/module selection.
- `resources/recipe_index.md` - scenario search across all modules.
- `resources/development_workflow.md` - shared implementation workflow.
- `resources/source_strategy.md` - narrow search strategy for the large repo collection.
- `resources/recipe_quality.md` - required metadata and evidence standard for recipes.
- `scripts/validate_mcu_skill.py` - structural and recipe-quality smoke check.
- `scripts/check_recipe_symbols.py` - optional symbol-evidence scan for a selected repo or recipe.
- `resources/glossary.md` - terms and stack names.
- `repos/<repo>/SKILL.md` - selected module guidance.
- `repos/<repo>/recipes/*.md` and `repos/<repo>/resources/` - scenario and API evidence.

## Stop Conditions

Ask for target SoC and framework version when they materially change the answer. If a repo module marks a topic as a gap, or a symbol cannot be found in the selected project/source/reference, mark it unverified instead of inventing an API.
