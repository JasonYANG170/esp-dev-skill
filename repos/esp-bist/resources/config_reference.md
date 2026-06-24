# ESP-BIST Configuration (Kconfig) Reference

All symbols are defined in `src/bist/Kconfig`. Defaults below are the upstream
defaults; a per-application `bist.conf` (Kconfig syntax, **mandatory** even
when empty) can override them.

## Master Enable

| Symbol | Type | Default | Purpose |
|---|---|---|---|
| `CONFIG_ESP_BIST` | bool | `y` | Master enable for the entire library |

## Per-Test Module Enables (menu "ESP-BIST Tests")

| Symbol | Type | Default | Controls compilation of |
|---|---|---|---|
| `CONFIG_ESP_BIST_CPU_REG_TEST` | bool | `y` | `bist_cpu_regs_test` |
| `CONFIG_ESP_BIST_CPU_CSR_REG_TEST` | bool | `y` | `bist_cpu_csr_regs_test` |
| `CONFIG_ESP_BIST_MEMORY_RAM_TEST` | bool | `y` | March A / March X |
| `CONFIG_ESP_BIST_MEMORY_FLASH_TEST` | bool | `y` | `bist_flash_test` |
| `CONFIG_BIST_FLASH_TEST_CHUNK_SIZE` | hex (`depends on ESP_BIST_MEMORY_FLASH_TEST`) | `0x1000` | Flash CRC chunk size (bytes) |
| `CONFIG_ESP_BIST_STACK_TEST` | bool | `y` | Stack overflow tests |
| `CONFIG_ESP_BIST_CLOCK_TEST` | bool | `y` | Clock tests |
| `CONFIG_ESP_BIST_CLOCK_PERCENT_FREQUENCY_DRIFT` | int (`depends on ESP_BIST_CLOCK_TEST`, range 0–100) | `1` | Allowed main XTAL drift (%) |
| `CONFIG_ESP_BIST_GPIO_TEST` | bool | `y` | GPIO plausibility tests |
| `CONFIG_ESP_BIST_PROGRAM_COUNTER_TEST` | bool | `y` | Program Counter test |
| `CONFIG_ESP_BIST_WATCHDOG_TEST` | bool | `y` | `bist_wdt_test` |

## Standalone Build Configs (not under Zephyr)

| Symbol | Type | Default | Purpose |
|---|---|---|---|
| `CONFIG_ESP_BIST_FLASH_SIZE` | hex | `0x400000` | Flash size in bytes (4 MB default) |
| `CONFIG_ESP_BIST_HEAP_SIZE` | hex | `0x1000` | Heap for BIST dynamic allocation; `0` = none |
| `CONFIG_ESP_BIST_STACK_PROTECTION_BLOCK_SIZE` | hex | `0x100` | Stack sentinel block size (256 bytes) |
| `CONFIG_ESP_BIST_WDT_TIMEOUT_US` | int | `50000` | WDT timeout (top of windowed window) |
| `CONFIG_ESP_BIST_WDT_WINDOWED_UNDERFLOW_TIMEOUT_US` | int | `10` | Windowed WDT underflow (bottom of window) |
| `CONFIG_ESP_BIST_METRICS` | bool | `y` | Enable `BIST_METRICS_*` macros |

## How Configuration Is Consumed

1. `bist.conf` in the app dir is auto-included by `cmake/project.cmake`.
2. `ninja -C build menuconfig` opens the interactive Kconfig editor; changes write to `build/config/sdkconfig`.
3. Kconfig values are compiled into a generated `bist_conf.h`, which `bist_core.h` includes (except in Zephyr builds, where `__ZEPHYR__` is defined and `bist_conf.h` is skipped).
4. The same `CONFIG_*` macros are available to application code at compile time (e.g. `CONFIG_ESP_BIST_WDT_TIMEOUT_US` used directly in `samples/standalone/main.c`).

## Example `bist.conf` Files in the Repo

| Path | What it enables |
|---|---|
| `samples/standalone/bist.conf` | Empty → all defaults (every test compiled in) |
| `tests/cpu_reg_test/bist.conf` | Only CPU reg + CSR |
| `tests/ram_test/bist.conf` | Only RAM |
| `tests/flash_test/bist.conf` | Only Flash |
| `tests/clock_test/bist.conf` | Only clock |
| `tests/pc_test/bist.conf` | Only PC |
| `tests/wdt_test/bist.conf` | Only watchdog |
| `tests/digital_io_test/bist.conf` | Only GPIO |
| `tests/cpu_stack_test/bist.conf` | Only stack |
| `tests/windowed_wdt_test/bist.conf` | All tests disabled (driver-level windowed WDT test) |

Single-test pattern (from `tests/cpu_reg_test/bist.conf`):
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

## Zephyr Build Note

When building with Zephyr (`__ZEPHYR__` defined, `ZEPHYR_ESP_BIST_MODULE`), the per-test defaults flip to `default y if !ZEPHYR_ESP_BIST_MODULE` and `bist_conf.h` is not included. The Zephyr integration lives under `samples/zephyr/` and `src/bist/zephyr.cmake`.
