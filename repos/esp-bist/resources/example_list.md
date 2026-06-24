# ESP-BIST Examples & Samples Index

Real paths under the upstream repo (`D:/esp-skill/espressif-repos/esp-bist/`).
Use these as starting points; do not fabricate example names.

## Samples (full integration)

| Path | Description |
|---|---|
| `samples/standalone/` | Canonical IEC 60730 standalone application: post-boot tests (crystal/RAM/Flash/stack/IO) + runtime tests (CPU/CSR/stack/PC) + windowed WDT + fail-safe. Contains `main.c`, `CMakeLists.txt`, `bist.conf`, `gdbinit`. |
| `samples/zephyr/` | Zephyr RTOS integration on ESP32-C6 HP+LP dual core. HP ping/pong host (`src/main.c`), LP remote running BIST (`remote/src/main.c`: `bist_cpu_regs_test`/`bist_cpu_csr_regs_test`/`bist_ram_test_march_x` with `ulp_lp_core_intr_disable()/enable()`). Built via `west build --sysbuild`; `Kconfig.sysbuild` auto-binds `REMOTE_BOARD`; `remote/boards/esp32c6_devkitc_lpcore.overlay` wires mbox/SHM/LP UART. |

## Tests (Unity single-test harnesses, each with `main.c`, `CMakeLists.txt`, `bist.conf`)

| Path | Description | BIST API exercised |
|---|---|---|
| `tests/cpu_reg_test/` | CPU register integrity + CSR | `bist_cpu_regs_test`, `bist_cpu_csr_regs_test` |
| `tests/cpu_stack_test/` | Stack overflow detection stress | `bist_cpu_stack_overflow_test` |
| `tests/ram_test/` | RAM March A + March X | `bist_ram_test_march_a`, `bist_ram_test_march_x` |
| `tests/flash_test/` | Flash CRC32 integrity | `bist_flash_test` |
| `tests/clock_test/` | 32 kHz XT WDT + 40 MHz drift | `bist_ext_crystal_fail_test`, `bist_main_crystal_test` |
| `tests/pc_test/` | Program Counter integrity | `bist_pc_test` (with `wdt_init`/`wdt_deinit`) |
| `tests/wdt_test/` | MWDT reset + sub-tick rejection | `bist_wdt_test`, `wdt_init(1)` negative case |
| `tests/digital_io_test/` | GPIO output/input + invalid-GPIO | `bist_gpio_output_test`, `bist_gpio_input_test` |
| `tests/windowed_wdt_test/` | Windowed WDT: normal / underflow / consecutive | `wdt_init`, `wdt_init_windowed`, `wdt_feed`, `wdt_is_underflow_detected` |

Most test dirs also ship a Pytest harness:
- QEMU: `pytest_qemu_<name>.py`
- Device: `pytest_device_<name>.py`

(`clock_test/` has no pytest harness in the tree; the others listed do.)

## Key Source Layout (for reference, not copy targets)

| Path | Contents |
|---|---|
| `src/bist/include/` | `bist_esp.h`, `bist_esp_types.h`, `bist_core.h`, `bist_metrics.h` |
| `src/bist/core/cpu/include/` | `bist_cpu_regs.h`, `bist_cpu_csr_regs.h`, `bist_cpu_stack.h`, `bist_pc.h` |
| `src/bist/core/memory/include/` | `bist_ram.h`, `bist_flash.h` |
| `src/bist/core/clock/include/` | `bist_clock_fail.h` |
| `src/bist/core/wdt/include/` | `bist_wdt.h` |
| `src/bist/core/io/include/` | `bist_gpio.h` |
| `src/bist/drivers/include/` | `wdt.h`, `gpio.h`, `esp_timer.h`, `bist_log.h` |
| `src/bist/Kconfig` | All `CONFIG_ESP_BIST_*` symbols |
| `src/soc/<target>/ld/linker.ld` | `_bist_ram_test_start`, `_dram0_end`, `_stack_overflow_protection_start`, `.pc_test_X`, `.crc_section_text/data` |
| `scripts/calculate_crc32.py` | Post-build CRC32 injector (polynomial `0x04C11DB7`) |
| `cmake/project.cmake` | Build entry; registers post-build CRC hook |
| `cmake/<target>.cmake` | Toolchain/target files: `esp32c3`, `esp32c5`, `esp32c6`, `esp32h2` |

## Docs (read these for authoritative detail)

| Path | Contents |
|---|---|
| `docs/en/index.rst` | Doc tree root + revision history (v1.0.0, Jan 23 2026) |
| `docs/en/get_started.rst` | Supported SoCs table, build/flash/QEMU, MCUboot |
| `docs/en/software_safety_requirements.rst` | IEC 60730 component → fault → module mapping |
| `docs/en/software_architecture.rst` | Three-layer arch, IRAM placement, `.dram0.safe_ram` |
| `docs/en/module_design_and_coding.rst` | Per-test algorithms, exec-time/cycle tables, CRC injection |
| `docs/en/application_guide.rst` | Standalone integration pattern, execution order, timing |
| `docs/en/api.rst` | API doc tree (includes the `inc/*.inc` generated files) |
| `docs/en/software_validation.rst` | QEMU/GDB fault-injection flow per test; expected PASS/FAIL outputs; CI JUnit references |
| `docs/en/test_traceability_matrix.rst` | IEC 60730 component ID → design → test → results, 4-level traceability |
| `docs/en/coverage_analysis.rst` | Module + requirements coverage matrices; known hardware-only limits |
| `docs/en/tool_qualification.rst` | T1/T2/T3 tool classification; GCC/ld/cppcheck/QEMU/GDB qualification |
| `docs/en/safety_case_summary.rst` | 6 safety claims × evidence × status; residual-risk argument |
| `docs/en/mcp_server.rst` | MCP server: 6 tools, setup (Cursor/VS Code), `ingest.py`, `mcp_data_drift` CI |
| `README.md`, `src/bist/README.md` | Repo + library overviews |

## MCP Server (AI assistant integration)

| Path | Contents |
|---|---|
| `mcp-server/server.py` | FastMCP server exposing 6 tools: `search_bist_docs`, `get_api_reference`, `get_architecture_info`, `search_kconfig_options`, `search_source_code`, `get_supported_socs` |
| `mcp-server/ingest.py` | Deterministic 5-step offline pipeline → `data/{docs,api,kconfig,source,socs}.json`; path-independent via `_relativize` |
| `mcp-server/search.py` | `BISTSearchEngine` (BM25Okapi) + tokenizer |
| `mcp-server/requirements.txt` | `rank_bm25>=0.2.2` |
| `mcp-server/mcp.json` | Optional platform entry point hint |
| `.cursor/mcp.json` | Workspace-level Cursor registration (`${workspaceFolder}`) |
| `.vscode/mcp.json` | Workspace-level VS Code registration + `dev.watch` auto-restart |

## Validation Infrastructure (QEMU/GDB fault injection)

| Path | Contents |
|---|---|
| `conftest.py` | `QEMU_RISCV` (`-icount 3`, debug `-s -S`), `GDB_RISCV` (`:1234`), `qemu_instance`/`qemu_debug_instance`/`gdb_instance` fixtures, `--executable` option |
| `tests/idf_targets.py` | 4-target parametrization: `esp32c3`, `esp32c5`, `esp32c6`, `esp32h2` |
| `tests/cpu_reg_test/pytest_qemu_cpu_reg_test.py` | CPU reg (32 params) + CSR fault injection via `tb testRegA_<name>` |
| `tests/cpu_stack_test/pytest_qemu_cpu_stack_test.py` | Stack overflow fault injection (`count_max` reduction) |
| `tests/ram_test/pytest_qemu_ram_test.py` | March A/X fault injection via linker symbol `_bist_ram_test_start` |
| `tests/flash_test/pytest_qemu_flash_test.py` | Flash CRC fault injection (`crc_section_len`) |
| `tests/pc_test/pytest_qemu_pc_test.py` | PC bit-flip + pointer-zero (WDT reset reason 1/7) |
| `tests/wdt_test/pytest_qemu_wdt_test.py` | WDT timeout stretch (`bist_test_wdt_timeout`) |
| `tests/windowed_wdt_test/pytest_qemu_windowed_wdt_test.py` | Windowed WDT normal/underflow/consecutive |
| `samples/standalone/gdbinit` | OpenOCD `:3333` debug script (QEMU uses `:1234`) |
