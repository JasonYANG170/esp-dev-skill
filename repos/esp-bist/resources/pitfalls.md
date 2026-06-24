# ESP-BIST Consolidated Pitfalls

Every entry is grounded in the repo's headers (`src/bist/**/include/*.h`),
examples (`samples/`, `tests/`), Kconfig (`src/bist/Kconfig`), and docs
(`docs/en/*.rst`). Read this before writing any BIST integration code.

---

## 1. Return-value discipline

Every `bist_*` test returns `bist_esp_err_t`. The library NEVER calls your
fail-safe handler — you must check every return value and route non-`BIST_ESP_OK`
to a fail-safe path (`fail_safe_exit()` infinite loop or custom).

```c
// WRONG
bist_cpu_regs_test();

// CORRECT
bist_esp_err_t err = bist_cpu_regs_test();
if (err == BIST_ESP_CPU_TEST_ERR) { fail_safe_exit(); }
```
Source: `samples/standalone/main.c` (runtime_tests / post_boot_tests).

---

## 2. Stack overflow: init before check

`bist_cpu_stack_overflow_check()` can only ever report `BIST_ESP_STACK_TEST_OVERFLOW`
after `bist_cpu_stack_overflow_init()` has written the `0xDEADBEEF` sentinel at
the linker symbol `_stack_overflow_protection_start`.

```c
// WRONG
while (1) { bist_cpu_stack_overflow_check(); }   // never trips

// CORRECT
bist_cpu_stack_overflow_init();
while (1) { if (bist_cpu_stack_overflow_check() == BIST_ESP_STACK_TEST_OVERFLOW) fail_safe_exit(); }
```
Source: `src/bist/core/cpu/include/bist_cpu_stack.h`; `samples/standalone/main.c`.

---

## 3. `bist_wdt_test()` is a two-boot sequence

First boot triggers the MWDT reset; success is only observable on the next boot
(reset reason == `RESET_REASON_CORE_MWDT0`). Do not expect it to "pass" in a
single run.
Source: `src/bist/core/wdt/include/bist_wdt.h`; `tests/wdt_test/main.c`.

---

## 4. `wdt_init()` minimum is one MWDT tick

`wdt_init(timeout_us)` returns `-1` for any value below one RTC tick
(~31 µs at 32768 Hz; documented minimum 500 µs). The repo's own test asserts
`wdt_init(1) == -1`.
Source: `src/bist/drivers/include/wdt.h`; `tests/wdt_test/main.c`.

---

## 5. Windowed WDT needs BOTH bounds

`wdt_init(timeout_us)` sets the top of the window; `wdt_init_windowed(underflow_us)`
sets the bottom. Without the windowed call, feeding too early goes undetected.
Source: `src/bist/README.md` (Windowed Watchdog); `samples/standalone/main.c`.

---

## 6. First-feed-then-immediate-feed is by-design underflow

After `wdt_init_windowed()`, the first `wdt_feed()` clears `window_open_flag`;
a second feed before the underflow window fires the violation. This is the
repo's `test_BIST_WINDOWED_WDT_UNDERFLOW`, not a bug.
Source: `tests/windowed_wdt_test/main.c`.

---

## 7. Flash CRC depends on the post-build injection

`bist_flash_test()` compares runtime CRC32 against values injected by
`scripts/calculate_crc32.py` into `.crc_section_text` / `.crc_section_data`
(via `cmake/project.cmake`'s post-build hook). Skip the hook and the test fails.
Source: `src/bist/core/memory/include/bist_flash.h`;
`docs/en/module_design_and_coding.rst` (post-build CRC injection).

---

## 8. RAM test region is linker-defined

March A/X operate on `_bist_ram_test_start` .. `_dram0_end`. The backup buffer
(`backup_chunk`) and 256-byte safe stack (`ram_test_stack`) live in
`.dram0.safe_ram`, which MUST sit below `_bist_ram_test_start` so the march can
test the full DRAM (including the normal stack) without trashing its own frame.
Source: `docs/en/module_design_and_coding.rst` (ram-test); `docs/en/software_architecture.rst`.

---

## 9. GPIO tests run BEFORE app GPIO config

`bist_gpio_output_test` / `bist_gpio_input_test` reset and reconfigure the pin.
Run them in post-boot, before `configure_led()` / `configure_button()`.
Source: `samples/standalone/main.c` (post_boot_tests → configure_* ordering).

---

## 10. Invalid GPIO numbers are rejected

`bist_gpio_output_test(SOC_GPIO_PIN_COUNT)` and `bist_gpio_output_test(-1)` both
return `BIST_ESP_IO_TEST_ERR`. Always pass a real, board-routed `gpio_num_t`.
Source: `src/bist/core/io/include/bist_gpio.h`; `tests/digital_io_test/main.c`.

---

## 11. SoC-dependent CSR masks — do not override

CSR write masks are chip-specific and already exclude write-once / reserved bits:
- PMPCFG: `0x1D1D1D1D` (C3/C6/H2), `0x0D0D0D0D` (C5; also excludes A[1])
- PMPADDR: `0xFFFFFFFF` (C3/C6/H2), `0x3FFFFFE0` (C5, 25-bit, 128-byte granularity)
- PMA addr `0x3FFFFFE0`; PMA cfg NOT tested (PMA_L write-once)
- `mexstatus` (`0x7E1`) and `mhint` (`0x7C5`) tested on C5 only
Do not "improve" the masks — write-once bits would lock until power-on reset.
Source: `src/bist/core/cpu/include/bist_cpu_csr_regs.h`; `src/bist/README.md`.

---

## 12. 32 kHz crystal test is skipped on C6/H2/C5

`bist_ext_crystal_fail_test()` returns `BIST_ESP_OK` (skipped) on SoCs without
`SOC_XT_WDT_SUPPORTED`. For clock validation there, rely on `bist_main_crystal_test()`.
Source: `src/bist/core/clock/include/bist_clock_fail.h`.

---

## 13. XT WDT callback must be IRAM and guarded

The MWDT callback runs from IRAM before reset; mark it `IRAM_ATTR` and keep it
short. `esp_xt_wdt_register_callback` (and its header `esp_xt_wdt.h`) only exist
when `SOC_XT_WDT_SUPPORTED` — guard with `#if`.
Source: `samples/standalone/main.c`.

---

## 14. `bist.conf` is mandatory

Every app directory must contain `bist.conf` (Kconfig syntax). It can be empty
(defaults apply, as in `samples/standalone/bist.conf`) but must exist, and is
what selects which `CONFIG_ESP_BIST_*_TEST` modules compile in.
Source: `README.md`; every `tests/*/bist.conf`.

---

## 15. PC test needs an active watchdog

`tests/pc_test/main.c` calls `wdt_init(10000)` before `bist_pc_test()` and
`wdt_deinit()` after. If you call `bist_pc_test()` with no WDT running you may
get an unexpected result.
Source: `tests/pc_test/main.c`.

---

## 16. Logging uses early-log variants

`bist_log.h` redefines `ESP_LOG*` to `ESP_EARLY_LOG*` so output works before
the full printf/UART stack is up. Prefer `ESP_LOGI/ESP_LOGE` in app code rather
than calling the early variants directly.
Source: `src/bist/drivers/include/bist_log.h`.

---

## 17. Do not modify SR code

The BIST library (`src/bist/**`) and SoC support (`src/soc/**/ld/linker.ld`,
`scripts/calculate_crc32.py`) are safety-relevant (SR) per
`docs/en/software_safety_requirements.rst`. Modifying them can invalidate the
IEC 60730 safety case. Treat as read-only.
