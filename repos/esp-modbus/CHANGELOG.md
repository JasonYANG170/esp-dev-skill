# Changelog

## [1.1.0] - 2026-06-18

### Added
- `recipes/slave_device_id.md` — slave-side Report Slave ID (FC 0x11) recipe: composing the vendor device-ID struct with the `INIT_DEV_ID` macro, when to call `mbc_set_slave_id` (after `mbc_slave_start`), `mbc_get_slave_id` read-back, the Kconfig triplet (`CONFIG_FMB_CONTROLLER_SLAVE_ID_SUPPORT` / `_SLAVE_ID` / `_SLAVE_ID_MAX_SIZE`), and the master FC 0x11 retrieval for cross-reference. Grounded in `examples/serial/mb_serial_slave/main/serial_slave.c`, `examples/serial/mb_serial_master/main/serial_master.c`, and `docs/en/slave_api_overview.rst`.

### Changed
- `SKILL.md` — added `slave_device_id.md` to the "Data model & advanced" scenario table; bumped `metadata.version` 1.0.0 → 1.1.0.
- `resources/example_list.md` — added a Report Slave ID cross-reference section linking the serial slave/master examples to the new recipe and the existing `serial_master.md` §6.
- `resources/api_reference.md` — expanded the Slave ID section with parameter semantics, return codes, the header note that FC 0x11 also applies to TCP slave, and the `MB_FUNC_OTHER_REP_SLAVEID_ENABLED` / `MB_FUNC_OTHER_REP_SLAVEID_BUF` macros from `mb_config.h`.

## [1.0.0] - 2026-06-18

Initial release of the esp-modbus-skill, an AI Skill (Claude Code / Agent format) for the Espressif esp-modbus library (Modbus RTU / ASCII / TCP, component v2.x.x).

### Added
- `SKILL.md` — core principles (12), when-to-use, recipe index, register-type / mode / Kconfig / error-code reference tables, 14 critical pitfalls (WRONG vs CORRECT), execution workflow, failure strategies, references.
- `AGENTS.md` — project context, file naming, include pattern, standard serial/TCP project structure, canonical master/slave init patterns (mirroring the examples), common macros, logging convention, build workflow, code-generation checklist, do-not-modify note.
- `recipes/` — 8 scenario recipes (Chinese prose, real fenced code, common-errors tables, references to real example paths):
  - `serial_slave.md` — Modbus serial slave (RTU/ASCII), RS485 setup, register-area descriptors, event loop, teardown.
  - `serial_master.md` — Modbus serial master, Data Dictionary, polling loop, custom command, Report Slave ID.
  - `tcp_slave.md` — Modbus TCP slave, netif init, multi-connection, keep-alive.
  - `tcp_master.md` — Modbus TCP master, slave IP address table (MDNS / static / IPv6), polling.
  - `data_dictionary.md` — authoring `mb_parameter_descriptor_t`, CID enum, `STR`/`OPTS`/`HOLD_OFFSET` macros, type↔FC pairing.
  - `slave_register_areas.md` — mapping storage structs to Modbus areas via `mbc_slave_set_descriptor`.
  - `custom_handlers.md` — registering/overriding function-code handlers (FC 0x41 echo example, FC 0x04 override).
  - `extended_types.md` — `CONFIG_FMB_EXT_TYPE_SUPPORT`, `PARAM_TYPE_*_ABCD` family, `mb_set_*`/`mb_get_*` endianness helpers.
  - `slave_events.md` — `mbc_slave_check_event` + `mbc_slave_get_param_info` event loop.
- `resources/api_reference.md` — verbatim function signatures from `esp_modbus_common.h` / `esp_modbus_master.h` / `esp_modbus_slave.h` / `mb_endianness_utils.h`, key types, enums, macros.
- `resources/config_reference.md` — all `CONFIG_FMB_*` Kconfig symbols (from `Kconfig`) plus example-level `CONFIG_MB_*` symbols, with defaults/ranges/depends; example `sdkconfig.defaults`; freemodbus exclusion note.
- `resources/pitfalls.md` — 15 consolidated gotchas with repository source-of-truth pointers.
- `resources/example_list.md` — table of every real example/test-app path under `examples/` and `test_apps/`.
- `README.md` (Chinese) and `CHANGELOG.md`.

### Grounding
Every API name, struct field, macro, Kconfig symbol, file path, and code snippet is sourced from the repository's own `docs/en/`, `modbus/mb_controller/common/include/`, `Kconfig`, and `examples/`. No APIs were invented.
