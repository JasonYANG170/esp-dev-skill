# ESP Recipe Quality Standard

Use this standard when creating or revising Espressif recipes. The goal is to bind code to an exact SoC, framework version, and component source.

## Required Metadata

Every recipe should state near the top:

- Applicable SoCs and excluded SoCs.
- ESP-IDF, Arduino-ESP32, ESP8266 RTOS SDK, or component version.
- Required dependencies and whether they come from Component Registry or local source.
- Example path or source files used as evidence.
- Validation level: `compiled`, `source-matched`, `example-derived`, or `draft`.

## Required Implementation Evidence

For each nontrivial code block, cite or name:

- Header/source file containing each API.
- Kconfig symbol source when `CONFIG_*` values are required.
- `idf_component.yml`, `CMakeLists.txt`, Arduino library, or build setting changes.
- Required partition/NVS/certificate/model files.
- Target-specific feature assumptions such as BLE, Classic Bluetooth, USB, 802.15.4, PSRAM, camera, LCD, or LP core.

## Review Checklist

- No ESP32-family feature claim without exact SoC support.
- No ESP-IDF v4/v5 API mixing.
- No Arduino and ESP-IDF project structure mixing unless using Arduino as an IDF component.
- No missing dependency declaration for managed components.
- Build, flash, monitor, and menuconfig commands match the selected workflow.

## Optional Symbol Scan

Run `scripts/check_recipe_symbols.py --scope-glob "repos/<repo>/recipes/*.md"` when revising a repo, or `--recipe-glob "repos/<repo>/recipes/<recipe>.md"` for one file. Treat findings as review leads; C++ methods, user callbacks, and generated APIs can require manual judgment.
