# AGENTS.md — Supplementary Agent Guide (esp-bist-skill)

> Core rules, recipe index, error codes, pitfalls, and execution workflow live in `SKILL.md`.
> This file covers **only** conventions and tooling not already in `SKILL.md`. Do not duplicate.

## Project Context

- **Language**: C (`-std=gnu17`)
- **Target**: Espressif RISC-V SoCs — ESP32-C3, ESP32-C5, ESP32-C6, ESP32-H2 (C61 / H4 / P4 documented in `docs/en/get_started.rst`)
- **Build system**: CMake + Ninja, on top of ESP-IDF (set `IDF_PATH`, run `. $IDF_PATH/export.sh`)
- **Bootloader**: MCUboot 2.2.0 (`/opt/mcuboot` in the dev container)
- **Toolchain**: `riscv32-esp-elf-gcc` / `riscv32-esp-elf-gdb`
- **License**: LGPL-3.0-or-later (see repo `LICENSE`)

## Code Generation Conventions

### File Naming
- Application entry: `main.c` (every `tests/<name>/` and `samples/standalone/` uses `main.c`)
- Unity harnesses keep `setUp()` / `tearDown()` stubs even when empty
- Per-app config: `bist.conf` (Kconfig syntax, **mandatory** even if empty)
- Per-app build: `CMakeLists.txt`

### Include Pattern
```c
// Umbrella — pulls bist_esp_types.h + bist_core.h (all bist_* test prototypes)
#include "bist_esp.h"

// Logging (re-maps ESP_LOG* to ESP_EARLY_LOG* so logs work pre-printf)
#include "bist_log.h"

// Driver wrappers (only when driving WDT / GPIO / timer directly)
#include "wdt.h"
#include "gpio.h"
#include "esp_timer.h"

// Optional: metrics via RISC-V Performance Counter CSRs
#include "bist_metrics.h"

// IDF/HAL helpers commonly used alongside BIST
#include "rom/ets_sys.h"          // ets_delay_us
#include "esp_attr.h"             // IRAM_ATTR
#include "soc/soc_caps.h"         // SOC_XT_WDT_SUPPORTED, SOC_GPIO_PIN_COUNT
#include "esp_xt_wdt.h"           // only when SOC_XT_WDT_SUPPORTED
```

### Standard Application Structure (mirrors `samples/standalone/`)
```
my_bist_app/
├── CMakeLists.txt        # set BIST_ROOT_DIR, APP_NAME, APP_SOURCES; include(${BIST_ROOT_DIR}/cmake/project.cmake)
├── bist.conf             # Kconfig defaults; mandatory (can be empty)
├── main.c                # main(): post_boot_tests() -> init -> runtime loop
└── (optional) gdbinit    # fault-injection breakpoints for QEMU debug
```

### Canonical `main()` Pattern
From `samples/standalone/main.c` (abbreviated):
```c
#include "bist_esp.h"
#include "bist_log.h"
#include "wdt.h"
#include "gpio.h"
#include "rom/ets_sys.h"
#include "esp_attr.h"
#include "soc/soc_caps.h"

static void fail_safe_exit(void) { ESP_LOGE(TAG, "Fail safe exit"); while (1); }
void IRAM_ATTR wdt_callback(void *args) { ESP_LOGE(TAG, "User WDT callback triggered"); }

int main(void) {
    post_boot_tests();                                 // RAM/Flash/stack/IO/clock/WDT (once)
    bist_cpu_stack_overflow_init();                    // write 0xDEADBEEF sentinel
#if SOC_XT_WDT_SUPPORTED
    esp_xt_wdt_register_callback((esp_xt_callback_t)fail_safe_exit, NULL);
#endif
    wdt_register_callback(wdt_callback, NULL);
    runtime_tests();                                   // CPU/CSR/stack/PC (verify path)
    configure_led(); configure_button();
    wdt_init(CONFIG_ESP_BIST_WDT_TIMEOUT_US);
    wdt_init_windowed(CONFIG_ESP_BIST_WDT_WINDOWED_UNDERFLOW_TIMEOUT_US);
    while (1) { set_led(get_button()); runtime_tests(); wdt_feed(); }
    return 0;
}
```

### Unity Single-Test Harness Pattern
From `tests/cpu_reg_test/main.c` (shape repeated across all `tests/*/main.c`):
```c
#include "bist_esp.h"
#include "unity.h"
void setUp(void) {}
void tearDown(void) {}

void test_BIST_Cpu_Regs(void) {
    TEST_ASSERT_EQUAL(BIST_ESP_OK, bist_cpu_regs_test());
}
int main(void) {
    UNITY_BEGIN();
    RUN_TEST(test_BIST_Cpu_Regs);
    return UNITY_END();
}
```

### Logging Convention
`bist_log.h` redefines the standard `ESP_LOG*` macros to `ESP_EARLY_LOG*` variants, so logging works before the printf/UART subsystem is fully up. Prefer `ESP_LOGI/ESP_LOGE` (not the raw early variants) in application code.

### WDT Callback Convention
MWDT callbacks execute in IRAM just before reset. They must be:
- Marked `IRAM_ATTR`
- Short and deterministic (no flash reads, no malloc)
- Registered via `wdt_register_callback(cb, arg)` BEFORE `wdt_init()`

## Build Workflow

1. Set up IDF: `. $IDF_PATH/export.sh`
2. (Once) Build MCUboot bootloader:
   ```bash
   cd /opt/mcuboot/boot/espressif
   cmake -DCMAKE_TOOLCHAIN_FILE=tools/toolchain-<target>.cmake \
         -DMCUBOOT_TARGET=<target> -DESP_HAL_PATH=$IDF_PATH -B build -GNinja
   ninja -C build
   ninja -C build flash_boot -DESP_PORT=/dev/ttyUSB0
   ```
3. Build the BIST app (inside the app dir):
   ```bash
   cmake -DSOC_TARGET=<target> -B build -GNinja
   ninja -C build
   ```
4. Configure: `ninja -C build menuconfig` (edits Kconfig; defaults come from `bist.conf`)
5. Run on QEMU: `ninja -C build qemu` (or `qemu_debug` + `riscv32-esp-elf-gdb ... -ex "target remote :1234"`)
6. Flash device: `ninja -C build flash -DESP_PORT=/dev/ttyUSBx`
7. Monitor: `ninja -C build monitor` (exit with `Ctrl+]`)
8. Test suite (Pytest + Unity + GDB fault injection):
   - QEMU: `pytest pytest_qemu_* --executable=<app> --soc-target=<target> --junitxml=build/tests/report.xml`
   - Device: `pytest pytest_device_* --junitxml=build/tests/report.xml`

### `bist.conf` Patterns
- **Full standalone**: empty file (all `CONFIG_ESP_BIST_*_TEST` default to `y`)
- **Single test** (e.g. CPU regs only — from `tests/cpu_reg_test/bist.conf`):
  ```
  CONFIG_ESP_BIST_CPU_REG_TEST=y
  CONFIG_ESP_BIST_CPU_CSR_REG_TEST=y
  CONFIG_ESP_BIST_MEMORY_RAM_TEST=n
  CONFIG_ESP_BIST_MEMORY_FLASH_TEST=n
  CONFIG_ESP_BIST_STACK_TEST=n
  CONFIG_ESP_BIST_CLOCK_TEST=n
  CONFIG_ESP_BIST_GPIO_TEST=n
  CONFIG_ESP_BIST_PROGRAM_COUNTER_TEST=n
  CONFIG_ESP_BIST_WATCHDOG_TEST=n
  ```

## Code Generation Checklist

- [ ] `#include "bist_esp.h"` present (single umbrella header)
- [ ] Every `bist_*` call's return value checked and routed to `fail_safe_exit()` on error
- [ ] Post-boot tests (RAM/Flash/stack stress/WDT/clock/IO) run ONCE before the main loop
- [ ] Runtime tests (CPU regs / CSR / stack check / PC) run every loop iteration
- [ ] `bist_cpu_stack_overflow_init()` called after post-boot tests, before runtime checks
- [ ] `wdt_init(CONFIG_ESP_BIST_WDT_TIMEOUT_US)` + `wdt_init_windowed(...)` called before the loop
- [ ] `wdt_feed()` called inside the loop, AFTER the underflow window has elapsed
- [ ] WDT callback marked `IRAM_ATTR` and registered before `wdt_init()`
- [ ] XT WDT callback guarded by `#if SOC_XT_WDT_SUPPORTED`
- [ ] `bist.conf` exists in the app directory (even if empty)
- [ ] `CMakeLists.txt` includes `${BIST_ROOT_DIR}/cmake/project.cmake` (drives post-build CRC injection)
- [ ] No invented API names — every `bist_*` / config symbol verified against `resources/`

## Do Not Modify

- `src/bist/**` — the BIST library itself is safety-relevant (SR) code; modifying it can invalidate the IEC 60730 safety case. Treat as read-only.
- `src/soc/<target>/ld/linker.ld` — defines `_bist_ram_test_start`, `_dram0_end`, `_stack_overflow_protection_start`, `.crc_section_text` / `.crc_section_data`, and the `pc_test_X` sections. Moving these breaks the tests.
- `scripts/calculate_crc32.py` — the post-build CRC injector; its polynomial (`0x04C11DB7`) must match the runtime CRC table.
- `SKILL.md` frontmatter — skill metadata.
- `resources/` — quick-reference docs transcribed from the repo; edit only when the upstream API changes.
