# Changelog

All notable changes to this **Skill** (not the upstream esp-insights repo) are documented here.
Upstream repo history lives at `D:/esp-skill/espressif-repos/esp-insights/CHANGELOG.md`.

## [1.0.0] - 2026-06-18

Initial release of `esp-insights-skill`.

### Added
- `SKILL.md` — YAML frontmatter (name/description/trigger words/tags/license Apache-2.0/compatibility ESP32 + ESP-IDF >=5.1/version 1.0.0) and body: overview, 12 core principles, when-to-use, recipe index, component/transport matrix, key Kconfig reference, reporting lifecycle, 14 critical pitfalls (wrong/correct code blocks), execution workflow, failure strategies, references.
- `AGENTS.md` — project context (C / ESP32 / ESP-IDF), file naming, include pattern, HTTPS auth-key embed pattern, standard project structure, canonical init pattern adapted from `examples/minimal_diagnostics`, build workflow, codegen checklist, do-not-modify note.
- `recipes/` (10 recipes):
  - `https_quickstart.md` — HTTPS minimal integration
  - `mqtt_transport.md` — MQTT(TLS) + RainMaker Claiming + fctry partition
  - `custom_transport.md` — custom transport via register+enable
  - `log_capture.md` — error/warn capture + ESP_DIAG_EVENT + per-tag level
  - `core_dump.md` — core dump Kconfig/partition/ELF/upload
  - `custom_metrics.md` — custom metrics (1.0/2.0 APIs)
  - `custom_variables.md` — custom variables + network variables
  - `system_metrics.md` — heap/wifi metrics + network variables
  - `data_store_tuning.md` — RTC/RAM store sizing + low-mem events
  - `runtime_control.md` — reporting on/off, manual send, command-response
- `resources/`:
  - `api_reference.md` — real signatures grouped by header (esp_insights / esp_diagnostics / metrics / variables / system_metrics / network_variables / data_store), including metadata 1.0 vs 2.0 dual branches
  - `config_reference.md` — all Kconfig symbols from the three components + example sdkconfig.defaults + idf_component.yml deps
  - `pitfalls.md` — consolidated gotchas by theme
  - `example_list.md` — real example/component/doc paths with one-line descriptions
- `README.md` — Chinese intro, install instructions, scope.
- `CHANGELOG.md` — this file.

### Grounding
- All function names, structs, typedefs, macros, enums, Kconfig symbols, file paths, and code snippets sourced from `D:/esp-skill/espressif-repos/esp-insights`: `components/*/include/*.h`, `components/*/Kconfig`, `components/*/idf_component.yml`, `examples/*/`, `README.md`, `FEATURES.md`, `CHANGELOG.md`, `docs/en/*.rst`.
- No fabricated APIs. Where the repo is thin (e.g., Flash data store is documented as unsupported), the skill states that rather than inventing usage.
