---
name: esp-bist-skill
description: >-
  AI Skill for Espressif ESP-BIST (Built-In Self Test) safety library development. Used when
  users need to create, integrate, or debug IEC 60730 Class B self-test firmware on Espressif
  RISC-V SoCs, including CPU/CSR/PC integrity tests, RAM March A/X, Flash CRC, clock monitoring,
  stack overflow detection, and watchdog/windowed-WDT integration.
  Trigger words: "ESP-BIST", "BIST", "IEC 60730", "Class B", "自检", "内置自测", "安全关键", "safety-critical", "March A", "March X", "windowed watchdog", "CRC32", "ESP32-C3", "ESP32-C6", "ESP32-H2", "ESP32-C5"
tags:
  - embedded
  - esp-bist
  - BIST
  - safety-critical
  - IEC-60730
  - Class-B
  - RISC-V
  - espressif
  - self-test
  - watchdog
  - CRC32
license: LGPL-3.0-or-later
compatibility: Target SoCs ESP32-C3 / ESP32-C5 / ESP32-C6 / ESP32-H2 (and C61/H4/P4 per docs); CMake + Ninja build on top of ESP-IDF + MCUboot 2.2.0; RISC-V toolchain (riscv32-esp-elf)
metadata:
  author: Community
  version: "1.1.0"
---

# esp-bist-skill

AI Skill for the Espressif ESP-BIST (Built-In Self Test) library. ESP-BIST is a CMake library providing IEC 60730-1 Annex H (Class B) compliant hardware self-test routines for Espressif RISC-V SoCs: CPU registers, Configuration & Status Registers (CSR), RAM (March A/X), Flash (CRC32), Program Counter, clock sources, stack overflow, watchdog, and digital I/O. This skill provides scenario recipes, a grounded API reference, the real Kconfig option map, consolidated pitfalls, and the canonical post-boot + runtime integration pattern taken from `samples/standalone`.

## Core Principles

1. **Never invent APIs** — Every `bist_*` function, error code, and config symbol MUST be verified against `resources/api_reference.md` / `resources/config_reference.md`, which are transcribed from the repo's headers (`src/bist/**/include/*.h`) and `src/bist/Kconfig`. If a name is not in those files, it does not exist.
2. **One umbrella header** — Application code includes `"bist_esp.h"` (which pulls `bist_esp_types.h` + `bist_core.h`). The watchdog / GPIO / timer driver wrappers need their own headers: `"wdt.h"`, `"gpio.h"`, `"esp_timer.h"`.
3. **Post-boot vs runtime split** — Expensive or destructive tests (RAM March, Flash CRC, stack-overflow stress, watchdog reset, crystal) run ONCE at startup. Cheap idempotent tests (CPU regs, CSR, PC, stack-overflow *check*) run every main-loop iteration. Mirror `samples/standalone/main.c`.
4. **Every test returns `bist_esp_err_t`** — Never ignore the return value. A non-`BIST_ESP_OK` result MUST route to a fail-safe handler (`fail_safe_exit()` infinite loop, or custom). The library never calls your fail-safe for you.
5. **WDT timeout ≥ one RTC tick (≈ 500 µs / 31 µs at 32768 Hz)** — `wdt_init()` returns `-1` for any value below one MWDT tick. The repo's own test asserts `wdt_init(1) == -1`.
6. **Watchdog test is a two-boot sequence** — `bist_wdt_test()` first triggers a reset; success is only detectable on the NEXT boot by checking the reset reason is `RESET_REASON_CORE_MWDT0`. Do not expect it to "pass" in a single run.
7. **RAM test is non-destructive but needs the linker region** — March A/X back up the region into a `.dram0.safe_ram` backup buffer and relocate the stack to a 256-byte safe stack before testing. The tested region is bounded by linker symbols `_bist_ram_test_start` .. `_dram0_end`, defined in `src/soc/<target>/ld/linker.ld`.
8. **Flash CRC needs the post-build step** — `bist_flash_test()` compares runtime CRC32 of `.flash.text` / `.flash.rodata` against reference values injected by `scripts/calculate_crc32.py` into `.crc_section_text` / `.crc_section_data`. If you bypass `cmake/project.cmake`'s post-build hook, the stored CRC is zero and the test fails with `BIST_ESP_FLASH_TEST_ERR`.
9. **Stack overflow uses a 0xDEADBEEF sentinel** — `bist_cpu_stack_overflow_init()` writes the sentinel at the linker symbol `_stack_overflow_protection_start`. You MUST call init before `bist_cpu_stack_overflow_check()` will ever report `BIST_ESP_STACK_TEST_OVERFLOW`.
10. **CSR/PMP masks are chip-specific and safety-locked** — Never write test patterns to PMP Lock or PMA_L bits; they are write-once until power-on reset. The library's masks already exclude them (e.g. PMPCFG mask `0x1D1D1D1D` on C3/C6/H2, `0x0D0D0D0D` on C5). Do not "improve" the masks.
11. **Windowed WDT detects BOTH too-early and too-late feeds** — `wdt_init_windowed(underflow_us)` defines the bottom of the window; `wdt_init(timeout_us)` defines the top. `wdt_is_underflow_detected()` returns true if you fed before the underflow window. On C6/H2 (no XT WDT) the 32 kHz external-crystal test is skipped and returns `BIST_ESP_OK`.
12. **`bist.conf` is mandatory** — Every application directory must contain a `bist.conf` (Kconfig syntax). It can be empty (defaults apply) but must exist, and is what compiles individual `CONFIG_ESP_BIST_*_TEST` modules in or out.

## When to Use

**Applicable:**
- Building safety-critical firmware on ESP32-C3 / C5 / C6 / C61 / H2 / H4 / P4 targeting IEC 60730-1 Annex H Class B (or IEC 60335-1 Annex R)
- Integrating ESP-BIST post-boot and runtime self-tests into a new or existing bare-metal/IDF critical firmware
- Writing or adapting a `samples/standalone`-style application (windowed WDT, fail-safe exit, runtime test loop)
- Debugging a specific BIST test failure (`BIST_ESP_*_ERR`) and mapping it back to the IEC 60730 component
- Configuring `bist.conf` / `CONFIG_ESP_BIST_*` Kconfig options (WDT timeouts, clock drift %, heap size, flash chunk size)
- Adding performance-metrics collection via the RISC-V Performance Counter CSRs (`BIST_METRICS_*`)

**Not applicable:**
- General ESP-IDF application development with no safety/self-test requirement (use the ESP-IDF skill instead)
- Non-Espressif or non-RISC-V targets (BIST is RISC-V CSR/PMP/PMA specific)
- Xtensa-based ESP32 / S2 / S3 / C6* classic cores (the library targets the RISC-V families listed in `get_started.rst`)
- PCB / schematic design or analog circuit verification

---

## Scenario Quick Reference (Recipes)

When user intent matches a scenario, **read the corresponding recipe first** — it carries the complete call chain, real code adapted from the repo, and a common-errors table.

### Project Setup & Integration

| recipe | scenario |
|---|---|
| `recipes/project_setup.md` | New BIST application: CMakeLists, `bist.conf`, MCUboot bootloader, build/flash/QEMU |
| `recipes/standalone_integration.md` | Full IEC 60730 integration pattern — post-boot tests + runtime loop + windowed WDT + fail-safe (mirrors `samples/standalone`) |
| `recipes/zephyr_integration.md` | Zephyr RTOS on ESP32-C6 HP+LP dual core: `west build --sysbuild`, mbox ping/pong IPC, LP-core BIST with `ulp_lp_core_intr_disable()/enable()` (mirrors `samples/zephyr`) |

### CPU & Control-Flow Tests

| recipe | scenario |
|---|---|
| `recipes/cpu_register_csr_test.md` | CPU general-purpose registers + CSR (trap / PMP / PMA / mexstatus) integrity |
| `recipes/pc_and_stack_test.md` | Program Counter test (IRAM/Flash/RTC placement) + stack overflow init/check/stress |
| `recipes/watchdog_windowed_test.md` | WDT reset test + windowed WDT (underflow / overflow / consecutive feeds) |

### Memory & Clock Tests

| recipe | scenario |
|---|---|
| `recipes/ram_flash_test.md` | RAM March A / March X + Flash CRC32 integrity (post-build CRC injection) |
| `recipes/clock_test.md` | External 32 kHz crystal (XT WDT) + main 40 MHz crystal drift measurement |
| `recipes/gpio_test.md` | GPIO output / input plausibility test + invalid-GPIO handling |

### Utilities

| recipe | scenario |
|---|---|
| `recipes/metrics_collection.md` | RISC-V Performance Counter metrics: cycles, instructions, hazards via `BIST_METRICS_*` |

### AI Assistant & Tooling

| recipe | scenario |
|---|---|
| `recipes/mcp_server_setup.md` | Wire the shipped ESP-BIST MCP server (6 tools, FastMCP + BM25) into Cursor / VS Code; regenerate `data/*.json` via `ingest.py`; satisfy the `mcp_data_drift` CI job |

### Safety Certification

| recipe | scenario |
|---|---|
| `recipes/safety_validation_workflow.md` | IEC 60730 Class B evidence workflow: QEMU `-icount 3` + GDB `:1234` fault injection driven by `pytest_qemu_*`, JUnit XML → traceability matrix → coverage → tool qualification → safety case summary |

---

## Supported SoCs

Source: `README.md`, `src/bist/README.md`, `docs/en/get_started.rst`.

| SoC | CPU / Arch | Max Freq | SRAM | XT WDT (32 kHz test) | CSR count |
|---|---|---|---|---|---|
| ESP32-C3 | Single-core RISC-V | 160 MHz | 400 + 8 RTC KB | Yes (`SOC_XT_WDT_SUPPORTED`) | 25 |
| ESP32-C5 | RISC-V + LP RISC-V | 240 MHz | 384 HP + 16 LP KB | No (skipped) | 39 (+ mexstatus, mhint) |
| ESP32-C6 | RISC-V + LP RISC-V | 160 MHz | 512 HP + 16 LP KB | No (skipped) | 37 (PMA addr) |
| ESP32-H2 | Single-core RISC-V | 96 MHz | 320 + 4 RTC KB | No (skipped) | 37 (PMA addr) |
| ESP32-C61 / H4 / P4 | RISC-V families | per datasheet | per datasheet | per chip | per chip |

> Note: The two README files list slightly different active sets (README.md names C3/C5/C6/H2; `get_started.rst` additionally documents C61/H4/P4 as supported targets). The CI toolchain files in `cmake/` cover `esp32c3`, `esp32c5`, `esp32c6`, `esp32h2`. Prefer those four for guaranteed build support.

## Key Configuration (Kconfig)

Full list in `resources/config_reference.md`. All symbols live in `src/bist/Kconfig`.

| Symbol | Type | Default | Purpose |
|---|---|---|---|
| `CONFIG_ESP_BIST` | bool | `y` | Master enable for the whole library |
| `CONFIG_ESP_BIST_CPU_REG_TEST` | bool | `y` | Compile `bist_cpu_regs_test` |
| `CONFIG_ESP_BIST_CPU_CSR_REG_TEST` | bool | `y` | Compile `bist_cpu_csr_regs_test` |
| `CONFIG_ESP_BIST_MEMORY_RAM_TEST` | bool | `y` | Compile March A / March X |
| `CONFIG_ESP_BIST_MEMORY_FLASH_TEST` | bool | `y` | Compile `bist_flash_test` |
| `CONFIG_BIST_FLASH_TEST_CHUNK_SIZE` | hex | `0x1000` | Flash CRC chunk size (bytes) |
| `CONFIG_ESP_BIST_STACK_TEST` | bool | `y` | Compile stack overflow tests |
| `CONFIG_ESP_BIST_CLOCK_TEST` | bool | `y` | Compile clock tests |
| `CONFIG_ESP_BIST_CLOCK_PERCENT_FREQUENCY_DRIFT` | int | `1` | Allowed main XTAL drift (%) |
| `CONFIG_ESP_BIST_GPIO_TEST` | bool | `y` | Compile GPIO plausibility tests |
| `CONFIG_ESP_BIST_PROGRAM_COUNTER_TEST` | bool | `y` | Compile PC test |
| `CONFIG_ESP_BIST_WATCHDOG_TEST` | bool | `y` | Compile `bist_wdt_test` |
| `CONFIG_ESP_BIST_FLASH_SIZE` | hex | `0x400000` | Flash size (bytes) |
| `CONFIG_ESP_BIST_HEAP_SIZE` | hex | `0x1000` | Heap size for BIST ops |
| `CONFIG_ESP_BIST_STACK_PROTECTION_BLOCK_SIZE` | hex | `0x100` | Stack sentinel block size |
| `CONFIG_ESP_BIST_WDT_TIMEOUT_US` | int | `50000` | WDT timeout (top of window) |
| `CONFIG_ESP_BIST_WDT_WINDOWED_UNDERFLOW_TIMEOUT_US` | int | `10` | Windowed WDT underflow (bottom) |
| `CONFIG_ESP_BIST_METRICS` | bool | `y` | Enable `BIST_METRICS_*` macros |

## Error Code Reference (`bist_esp_err_t`)

Source: `src/bist/include/bist_esp_types.h`.

| Value | Name | Meaning |
|---|---|---|
| 0 | `BIST_ESP_OK` | Test passed |
| 1 | `BIST_ESP_CPU_TEST_ERR` | CPU register test failed |
| 2 | `BIST_ESP_CPU_CSR_TEST_ERR` | CSR test failed |
| 3 | `BIST_ESP_RAM_TEST_ERR` | RAM March A/X mismatch |
| 4 | `BIST_ESP_FLASH_TEST_ERR` | Flash CRC mismatch |
| 5 | `BIST_ESP_PC_TEST_ERR` | Program Counter mismatch |
| 6 | `BIST_ESP_CLOCK_TEST_ERR` | Clock failure / drift out of tolerance |
| 7 | `BIST_ESP_STACK_TEST_ERR` | Stack overflow detection mechanism failed |
| 8 | `BIST_ESP_STACK_TEST_OVERFLOW` | Stack sentinel corrupted (overflow detected) |
| 9 | `BIST_ESP_WDT_TEST_ERR` | Watchdog did not trigger expected reset |
| 10 | `BIST_ESP_IO_TEST_ERR` | GPIO output/input plausibility failed |

## Standalone Execution Flow (state machine)

Adapted from `docs/en/application_guide.rst` and `samples/standalone/main.c`:

```
Boot
 │
 ▼
Post-Boot Tests  ── fail ──►  fail_safe_exit()  (infinite loop)
 │ pass
 ▼
bist_cpu_stack_overflow_init()        (write 0xDEADBEEF sentinel)
 │
 ├─ esp_xt_wdt_register_callback(fail_safe_exit)   (only if SOC_XT_WDT_SUPPORTED)
 ├─ wdt_register_callback(wdt_callback)
 ▼
configure_gpio()  →  wdt_init(TIMEOUT_US)  →  wdt_init_windowed(UNDERFLOW_US)
 │
 ▼
┌─── Main Loop ──────────────────────────────────────┐
│  application_logic()                               │
│  runtime_tests():                                  │
│     bist_cpu_regs_test()       → fail_safe_exit    │
│     bist_cpu_csr_regs_test()   → fail_safe_exit    │
│     bist_cpu_stack_overflow_check() → fail_safe_exit │
│     bist_pc_test()             → fail_safe_exit    │
│  wdt_feed()   (must be inside the windowed window) │
└───────────────────── loop ─────────────────────────┘
```

## Critical Pitfalls (Must Read)

These are the most common errors. Each shows a real WRONG vs CORRECT pattern grounded in the repo's headers, examples, and docs.

### 1. Ignoring the `bist_esp_err_t` return value

```c
// ❌ WRONG — fire-and-forget; a fault passes silently
bist_cpu_regs_test();
bist_cpu_csr_regs_test();

// ✅ CORRECT — route every failure to a fail-safe handler
bist_esp_err_t err = bist_cpu_regs_test();
if (err == BIST_ESP_CPU_TEST_ERR) {
    ESP_LOGE(TAG, "CPU register test failed");
    fail_safe_exit();
}
```

### 2. Calling `bist_cpu_stack_overflow_check()` without `init()`

```c
// ❌ WRONG — sentinel never written, check can never report overflow
while (1) {
    if (bist_cpu_stack_overflow_check() == BIST_ESP_STACK_TEST_OVERFLOW) {
        fail_safe_exit();
    }
}

// ✅ CORRECT — init the 0xDEADBEEF sentinel once after post-boot tests
bist_cpu_stack_overflow_init();   // writes _stack_overflow_protection_start
while (1) {
    if (bist_cpu_stack_overflow_check() == BIST_ESP_STACK_TEST_OVERFLOW) {
        fail_safe_exit();
    }
}
```

### 3. Expecting `bist_wdt_test()` to pass in a single boot

```c
// ❌ WRONG — first execution triggers a reset; "OK" only appears next boot
int main(void) {
    if (bist_wdt_test() != BIST_ESP_OK) fail_safe_exit();  // always resets first time
}

// ✅ CORRECT — two-boot design: trigger reset, then verify reset reason on next boot
//   The library checks reset reason == RESET_REASON_CORE_MWDT0 internally.
//   Integrate into post-boot tests (see samples/standalone) and accept the reboot.
static void post_boot_tests(void) {
    bist_esp_err_t err = bist_wdt_test();
    if (err == BIST_ESP_WDT_TEST_ERR) fail_safe_exit();
}
```

### 4. WDT timeout below one RTC tick

```c
// ❌ WRONG — 1 us is below one MWDT tick (~31 us @ 32768 Hz); init refuses
wdt_init(1);   // returns -1, watchdog never starts

// ✅ CORRECT — minimum accepted is 500 us (one MWDT tick)
if (wdt_init(500) != 0) {
    ESP_LOGE(TAG, "WDT init failed");
    fail_safe_exit();
}
// Or use the configured default:
wdt_init(CONFIG_ESP_BIST_WDT_TIMEOUT_US);
```

### 5. Forgetting the windowed WDT lower bound

```c
// ❌ WRONG — only top of window set; underflow (too-early feed) undetected
wdt_init(50000);
while (1) { /* ... */ wdt_feed(); }   // feeding in a tight loop looks "fine"

// ✅ CORRECT — both bounds; feed only after the underflow window elapses
wdt_init(50000);                              // top of window
wdt_init_windowed(2000);                      // bottom of window (2 ms)
while (1) {
    /* application work taking >= underflow_us */
    wdt_feed();
    if (wdt_is_underflow_detected()) fail_safe_exit();
}
```

### 6. Calling `wdt_feed()` immediately after `wdt_init_windowed()` then again too soon

```c
// ❌ WRONG — the first feed clears window_open_flag; an immediate second feed
//            fires the underflow violation (this is the repo's own negative test)
wdt_init_windowed(2000);
wdt_feed();
ets_delay_us(100);   // << underflow window
wdt_feed();          // wdt_is_underflow_detected() == true

// ✅ CORRECT — wait >= underflow_timeout_us between feeds
ets_delay_us(UNDERFLOW_TIMEOUT_US + FEED_MARGIN_US);
wdt_feed();
TEST_ASSERT_FALSE(wdt_is_underflow_detected());
```

### 7. Flash CRC "fails" because the post-build injection step was skipped

```c
// ❌ WRONG — custom build system that never runs scripts/calculate_crc32.py
//            _crc_section_text_start / _crc_section_data_start stay zero
bist_flash_test();   // → BIST_ESP_FLASH_TEST_ERR

// ✅ CORRECT — build through cmake/project.cmake, which runs post-build:
//   python scripts/calculate_crc32.py <elf> .flash.text  .crc_section_text
//   python scripts/calculate_crc32.py <elf> .flash.rodata .crc_section_data
// Then at runtime the stored CRCs are non-zero and bist_flash_test() returns OK.
```

### 8. Passing an invalid GPIO number to the IO test

```c
// ❌ WRONG — SOC_GPIO_PIN_COUNT / -1 are out of range
bist_gpio_output_test(SOC_GPIO_PIN_COUNT);   // → BIST_ESP_IO_TEST_ERR
bist_gpio_input_test(-1, 0);                 // → BIST_ESP_IO_TEST_ERR

// ✅ CORRECT — use a real, board-routed GPIO; input test needs external level
#define LED_GPIO 7
#define BTN_GPIO 9
bist_gpio_output_test(LED_GPIO);                 // expect BIST_ESP_OK
bist_gpio_input_test(BTN_GPIO, 1);               // button tied high = expect 1
```

### 9. Running GPIO tests AFTER the application has reconfigured the pins

```c
// ❌ WRONG — IO tests reset and reconfigure pins; do them before app GPIO setup
configure_led();        // sets GPIO7 as output
bist_gpio_output_test(LED_GPIO);   // pin state fight with app config

// ✅ CORRECT — order matches samples/standalone: tests first, then app config
post_boot_tests();      // includes bist_gpio_output_test / input_test
configure_led();
configure_button();
```

### 10. Expecting the 32 kHz external-crystal test to actually run on C6/H2

```c
// ❌ WRONG — assuming the test will catch a dead 32 kHz crystal on C6/H2
// (those SoCs lack SOC_XT_WDT_SUPPORTED; the test returns BIST_ESP_OK = "skipped")
if (bist_ext_crystal_fail_test() != BIST_ESP_OK) fail_safe_exit();

// ✅ CORRECT — guard with SOC_XT_WDT_SUPPORTED; rely on main crystal test otherwise
bist_esp_err_t err = bist_ext_crystal_fail_test();   // skipped → OK on C6/H2
if (err == BIST_ESP_CLOCK_TEST_ERR) fail_safe_exit();
err = bist_main_crystal_test();                       // always validates 40 MHz
if (err == BIST_ESP_CLOCK_TEST_ERR) fail_safe_exit();
```

### 11. Defining `wdt_callback` without `IRAM_ATTR`

```c
// ❌ WRONG — MWDT callback runs before reset from IRAM context; a flash callback
//            may fault if flash cache is disabled
void wdt_callback(void *args) { ESP_LOGE(TAG, "WDT"); }

// ✅ CORRECT — place the callback in IRAM (as samples/standalone does)
void IRAM_ATTR wdt_callback(void *args) {
    ESP_LOGE(TAG, "User WDT callback triggered");
}
wdt_register_callback(wdt_callback, NULL);
```

### 12. Omitting the mandatory `bist.conf` file

```bash
# ❌ WRONG — empty app dir, no bist.conf; build fails or uses no module selection
my_app/
└── main.c

# ✅ CORRECT — bist.conf is mandatory even when empty (defaults apply)
my_app/
├── CMakeLists.txt
├── bist.conf        # can be empty, or pin CONFIG_ESP_BIST_*_TEST=y/n
└── main.c
```

---

## Execution Workflow

| Step | Name | Description |
|------|------|-------------|
| 1 | Plan | Identify target SoC, which IEC 60730 components apply, post-boot vs runtime split |
| 2 | Recipe | If the scenario matches a recipe in `recipes/`, follow its call chain and code |
| 3 | Query | For any API/config not in a recipe, look it up in `resources/api_reference.md` or `resources/config_reference.md` |
| 4 | Validate | Verify the function name exists in `resources/api_reference.md`; verify the Kconfig symbol exists in `resources/config_reference.md`; if absent, do NOT invent it |
| 5 | Confirm | Present: includes (`bist_esp.h` + driver headers), `bist.conf`, init order, runtime loop, fail-safe path |
| 6 | Execute | For a new app, copy the closest reference: `samples/standalone/` for full integration, or `tests/<name>/` for a single test, then modify |
| 7 | Build | `cmake -DSOC_TARGET=<target> -B build -GNinja && ninja -C build` (after `. $IDF_PATH/export.sh`) |
| 8 | Run | `ninja -C build qemu` (QEMU) or `ninja -C build flash` + `ninja -C build monitor` (device) |
| 9 | Verify | Confirm each test prints `BIST_ESP_OK`; on a deliberate fault (fault-injection via GDB) confirm `fail_safe_exit()` triggers |
| 10 | Certify | Use `docs/en/test_traceability_matrix.rst` + `docs/en/safety_case_summary.rst` for IEC 60730 evidence |

### Step 6 Detail — Project Creation Strategy

- **Full IEC 60730 integration** → copy `samples/standalone/` (has `main.c`, `CMakeLists.txt`, `bist.conf`, post-boot + runtime + windowed WDT)
- **Single-test harness (Unity)** → copy `tests/<test_name>/` (e.g. `tests/cpu_reg_test/`, `tests/ram_test/`, `tests/windowed_wdt_test/`)
- **Zephyr integration** → start from `samples/zephyr/`
- Always edit the copied `bist.conf` to enable exactly the `CONFIG_ESP_BIST_*_TEST` modules you need (see any `tests/*/bist.conf` for the pattern).

## Failure Strategies

| Situation | Action |
|---|---|
| A `bist_*` name is not in `resources/api_reference.md` | Stop; the API does not exist in this version of the library. Inform the user. |
| `wdt_init()` returns `-1` | The timeout is below one MWDT tick (≈ 500 µs / 31 µs). Raise it to at least 500 µs. |
| `bist_flash_test()` fails on a fresh build | Confirm the post-build CRC injection (`scripts/calculate_crc32.py`) ran via `cmake/project.cmake`; check `_crc_section_text_start` is non-zero. |
| `bist_cpu_stack_overflow_check()` never trips | `bist_cpu_stack_overflow_init()` was not called, or the linker symbol `_stack_overflow_protection_start` is mis-placed. |
| `bist_ext_crystal_fail_test()` "passes" on C6/H2 with no crystal | Expected — on SoCs without `SOC_XT_WDT_SUPPORTED` the test is skipped (`BIST_ESP_OK`). Use `bist_main_crystal_test()` for clock validation. |
| Windowed WDT flags underflow unexpectedly | Application loop is faster than `CONFIG_ESP_BIST_WDT_WINDOWED_UNDERFLOW_TIMEOUT_US`. Either raise the underflow value or add a deliberate delay before `wdt_feed()`. |
| `wdt_init_windowed()` "first feed then immediate feed" underflows | This is by design (the repo's `test_BIST_WINDOWED_WDT_UNDERFLOW` proves it). Wait ≥ underflow window between feeds. |
| Unsure which CSR mask applies | It is chip-specific and already encoded in the library (PMPCFG `0x1D1D1D1D` on C3/C6/H2, `0x0D0D0D0D` on C5; PMA not tested for cfg due to write-once Lock bit). Do not override. |
| Build cannot find `esp_xt_wdt.h` | That header comes from the ESP-IDF HAL (`components/`), not the BIST tree. Guard its use with `#if SOC_XT_WDT_SUPPORTED`. |

## References

- Scenario recipes → `recipes/` directory
- API quick reference (every `bist_*` signature) → `resources/api_reference.md`
- Kconfig / config reference → `resources/config_reference.md`
- Consolidated pitfalls → `resources/pitfalls.md`
- Real example & sample paths index → `resources/example_list.md`
- Upstream repo → https://github.com/espressif/esp-bist
