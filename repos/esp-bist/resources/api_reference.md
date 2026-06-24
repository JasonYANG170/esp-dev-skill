# ESP-BIST API Quick Reference

Every signature below is transcribed verbatim from the repo headers under
`src/bist/**/include/*.h` and `src/bist/drivers/include/*.h`. If a name is not
here, it does not exist in this version of the library.

## Umbrella Header

```c
#include "bist_esp.h"
//  → includes:
//     bist_esp_types.h   (error codes, ASM/BIST_ADD_LABEL macros)
//     bist_core.h        (all bist_* test prototypes below)
//     bist_conf.h        (generated from Kconfig / bist.conf)
```

Source: `src/bist/include/bist_esp.h`, `src/bist/core/include/bist_core.h`.

## Common Types & Error Codes

Source: `src/bist/include/bist_esp_types.h`.

```c
#define ASM(x)                __asm volatile(x)
#define BIST_ADD_LABEL(label) ASM(#label ":")

typedef enum {
    BIST_ESP_OK                  = 0,
    BIST_ESP_CPU_TEST_ERR        = 1,   // CPU register test failed
    BIST_ESP_CPU_CSR_TEST_ERR    = 2,   // CSR test failed
    BIST_ESP_RAM_TEST_ERR        = 3,   // RAM March A/X mismatch
    BIST_ESP_FLASH_TEST_ERR      = 4,   // Flash CRC mismatch
    BIST_ESP_PC_TEST_ERR         = 5,   // Program Counter mismatch
    BIST_ESP_CLOCK_TEST_ERR      = 6,   // Clock failure / drift
    BIST_ESP_STACK_TEST_ERR      = 7,   // Stack detection mechanism failed
    BIST_ESP_STACK_TEST_OVERFLOW = 8,   // Stack sentinel corrupted
    BIST_ESP_WDT_TEST_ERR        = 9,   // Watchdog did not reset as expected
    BIST_ESP_IO_TEST_ERR         = 10,  // GPIO plausibility failed
} bist_esp_err_t;
```

## CPU Tests

Source: `src/bist/core/cpu/include/bist_cpu_regs.h`, `bist_cpu_csr_regs.h`.

```c
// Test all 32 RISC-V GP registers (X1-X31) with 0xAAAAAAAA / 0x55555555.
bist_esp_err_t bist_cpu_regs_test(void);
//   → BIST_ESP_OK | BIST_ESP_CPU_TEST_ERR

// Test CSRs (trap / PMP / PMA[C6,H2,C5] / mexstatus+mhint [C5 only]).
// CSR counts: C3=25, C6/H2=37, C5=39.
bist_esp_err_t bist_cpu_csr_regs_test(void);
//   → BIST_ESP_OK | BIST_ESP_CPU_CSR_TEST_ERR
```

## Stack Tests

Source: `src/bist/core/cpu/include/bist_cpu_stack.h`.

```c
// Write 0xDEADBEEF sentinel at _stack_overflow_protection_start.
bist_esp_err_t bist_cpu_stack_overflow_init(void);     // → BIST_ESP_OK

// Verify sentinel intact (call after init).
bist_esp_err_t bist_cpu_stack_overflow_check(void);
//   → BIST_ESP_OK | BIST_ESP_STACK_TEST_OVERFLOW

// Stress: bounded recursion (max 20000 iters, 128-byte frames) to force overflow.
bist_esp_err_t bist_cpu_stack_overflow_test(void);
//   → BIST_ESP_OK (overflow detected) | BIST_ESP_STACK_TEST_ERR (detection failed)

// Bytes of stack still containing the fill pattern (high watermark).
uint32_t bist_get_stack_high_watermark(void);
```

## Program Counter Test

Source: `src/bist/core/cpu/include/bist_pc.h`.

```c
// Calls 4 functions placed in IRAM / Flash / RTC; compares returned address.
// NOTE: requires wdt_init() to be active (see tests/pc_test/main.c).
bist_esp_err_t bist_pc_test(void);   // → BIST_ESP_OK | BIST_ESP_PC_TEST_ERR
```

## RAM Tests (March A / March X)

Source: `src/bist/core/memory/include/bist_ram.h`.

```c
// 3-step March A; backs up to .dram0.safe_ram; region _bist_ram_test_start.._dram0_end.
bist_esp_err_t bist_ram_test_march_a(void);   // → BIST_ESP_OK | BIST_ESP_RAM_TEST_ERR

// 6-step March X (more comprehensive).
bist_esp_err_t bist_ram_test_march_x(void);   // → BIST_ESP_OK | BIST_ESP_RAM_TEST_ERR
```

## Flash Test (CRC32)

Source: `src/bist/core/memory/include/bist_flash.h`.

```c
// Recompute CRC32 of .flash.text + .flash.rodata in CONFIG_BIST_FLASH_TEST_CHUNK_SIZE
// chunks, compare to stored values injected by scripts/calculate_crc32.py.
bist_esp_err_t bist_flash_test(void);   // → BIST_ESP_OK | BIST_ESP_FLASH_TEST_ERR
```

## Clock Tests

Source: `src/bist/core/clock/include/bist_clock_fail.h`.

```c
// External 32.768 kHz crystal via XT WDT (200-cycle timeout).
// Skipped (returns BIST_ESP_OK) on SoCs without SOC_XT_WDT_SUPPORTED.
bist_esp_err_t bist_ext_crystal_fail_test(void);   // → BIST_ESP_OK | BIST_ESP_CLOCK_TEST_ERR

// Main 40 MHz crystal drift vs 32 kHz reference over 500 cycles.
// Tolerance = CONFIG_ESP_BIST_CLOCK_PERCENT_FREQUENCY_DRIFT (default 1%).
bist_esp_err_t bist_main_crystal_test(void);       // → BIST_ESP_OK | BIST_ESP_CLOCK_TEST_ERR
```

## Watchdog Test

Source: `src/bist/core/wdt/include/bist_wdt.h`.

```c
// Two-boot sequence: first boot triggers MWDT reset (100 us timeout, 1000 us wait);
// next boot verifies reset reason == RESET_REASON_CORE_MWDT0.
bist_esp_err_t bist_wdt_test(void);   // → BIST_ESP_OK | BIST_ESP_WDT_TEST_ERR
```

## GPIO Tests

Source: `src/bist/core/io/include/bist_gpio.h`.

```c
// Output: reset pin → output mode → set 0 read back → set 1 read back → reset.
bist_esp_err_t bist_gpio_output_test(gpio_num_t gpio_num);
//   → BIST_ESP_OK | BIST_ESP_IO_TEST_ERR (also for invalid gpio_num / -1)

// Input: reset pin → input mode → read level → compare to expected_level → reset.
bist_esp_err_t bist_gpio_input_test(gpio_num_t gpio_num, bool expected_level);
//   → BIST_ESP_OK | BIST_ESP_IO_TEST_ERR
```

## WDT Driver Wrapper

Source: `src/bist/drivers/include/wdt.h`.

```c
int  wdt_init(uint32_t timeout_us);                       // ≥ 500 us; 0 on success, -1 if too small
void wdt_deinit(void);
void wdt_feed(void);
void wdt_register_callback(void (*callback)(void *), void *arg);
int  wdt_init_windowed(uint32_t underflow_timeout_us);    // 0 / -1
void wdt_windowed_deinit(void);
bool wdt_is_underflow_detected(void);
```

## GPIO Driver Wrapper (selected)

Source: `src/bist/drivers/include/gpio.h`.

```c
typedef void (*gpio_isr_t)(void *arg);

typedef struct {
    uint64_t           pin_bit_mask;
    gpio_mode_t        mode;
    gpio_pullup_t      pull_up_en;
    gpio_pulldown_t    pull_down_en;
#if SOC_GPIO_SUPPORT_PIN_HYS_FILTER
    gpio_hys_ctrl_mode_t hys_ctrl_mode;
#endif
} gpio_config_t;

esp_err_t gpio_config(const gpio_config_t *pGPIOConfig);
esp_err_t gpio_reset_pin(gpio_num_t gpio_num);
esp_err_t gpio_set_level(gpio_num_t gpio_num, uint32_t level);
int      gpio_get_level(gpio_num_t gpio_num);
esp_err_t gpio_set_direction(gpio_num_t gpio_num, gpio_mode_t mode);
esp_err_t gpio_set_pull_mode(gpio_num_t gpio_num, gpio_pull_mode_t pull);
esp_err_t gpio_pullup_en(gpio_num_t gpio_num);
esp_err_t gpio_pullup_dis(gpio_num_t gpio_num);
esp_err_t gpio_pulldown_en(gpio_num_t gpio_num);
esp_err_t gpio_pulldown_dis(gpio_num_t gpio_num);
```

## esp_timer Driver Wrapper (selected)

Source: `src/bist/drivers/include/esp_timer.h`.

```c
typedef struct esp_timer* esp_timer_handle_t;
typedef void (*esp_timer_cb_t)(void *arg);

typedef enum { ESP_TIMER_TASK, ESP_TIMER_ISR, ESP_TIMER_MAX } esp_timer_dispatch_t;

typedef struct {
    esp_timer_cb_t        callback;
    void                 *arg;
    esp_timer_dispatch_t  dispatch_method;
    const char           *name;
    bool                  skip_unhandled_events;
} esp_timer_create_args_t;

esp_err_t esp_timer_early_init(void);
esp_err_t esp_timer_init(void);
esp_err_t esp_timer_deinit(void);
esp_err_t esp_timer_create(const esp_timer_create_args_t *create_args,
                           esp_timer_handle_t *out_handle);
esp_err_t esp_timer_start_once(esp_timer_handle_t timer, uint64_t timeout_us);
esp_err_t esp_timer_start_periodic(esp_timer_handle_t timer, uint64_t period);
esp_err_t esp_timer_restart(esp_timer_handle_t timer, uint64_t timeout_us);
esp_err_t esp_timer_stop(esp_timer_handle_t timer);
esp_err_t esp_timer_delete(esp_timer_handle_t timer);
int64_t   esp_timer_get_time(void);
int64_t   esp_timer_get_next_alarm(void);
int64_t   esp_timer_get_next_alarm_for_wake_up(void);
esp_err_t esp_timer_get_period(esp_timer_handle_t timer, uint64_t *period);
esp_err_t esp_timer_get_expiry_time(esp_timer_handle_t timer, uint64_t *expiry);
esp_err_t esp_timer_dump(FILE *stream);
void      esp_timer_isr_dispatch_need_yield(void);
bool      esp_timer_is_active(esp_timer_handle_t timer);
esp_err_t esp_timer_new_etm_alarm_event(esp_etm_event_handle_t *out_event);
```

## Logging

Source: `src/bist/drivers/include/bist_log.h`. Re-maps `ESP_LOG*` to `ESP_EARLY_LOG*` so logs work pre-printf.

```c
#define ESP_LOGE(tag, fmt, ...) ESP_EARLY_LOGE(tag, fmt, ##__VA_ARGS__)
#define ESP_LOGW(tag, fmt, ...) ESP_EARLY_LOGW(tag, fmt, ##__VA_ARGS__)
#define ESP_LOGI(tag, fmt, ...) ESP_EARLY_LOGI(tag, fmt, ##__VA_ARGS__)
#define ESP_LOGD(tag, fmt, ...) ESP_EARLY_LOGD(tag, fmt, ##__VA_ARGS__)
#define ESP_LOGV(tag, fmt, ...) ESP_EARLY_LOGV(tag, fmt, ##__VA_ARGS__)
```

## Performance Metrics

Source: `src/bist/include/bist_metrics.h`.

```c
#define CSR_PCER 0x7e0
#define CSR_PCMR 0x7e1
#define CSR_PCCR 0x7e2

typedef enum {
    BIST_METRICS_MODE_CYCLE         = 0,
    BIST_METRICS_MODE_INST          = 1,
    BIST_METRICS_MODE_LD_HAZARDS    = 2,
    BIST_METRICS_MODE_JMP_HAZARDS   = 3,
    BIST_METRICS_MODE_IDLE          = 4,
    BIST_METRICS_MODE_LOAD          = 5,
    BIST_METRICS_MODE_STORE         = 6,
    BIST_METRICS_MODE_JMP_UNCOND    = 7,
    BIST_METRICS_MODE_BRANCH        = 8,
    BIST_METRICS_MODE_BRANCH_TAKEN  = 9,
    BIST_METRICS_MODE_INST_COMP     = 10,
} bist_metrics_mode_t;

typedef struct {
    uint32_t start_value;
    uint32_t end_value;
    bool     valid;
} bist_metrics_t;

#define BIST_METRICS_INIT(mode_)
#define BIST_METRICS_BEGIN(metrics_)
#define BIST_METRICS_END(metrics_)
#define BIST_METRICS_PRINT(name_, metrics_)   // "METRICS: <name> Mode: <mode> delta=<count>"
```

## External (IDF HAL, referenced by samples — not in BIST tree)

- `esp_xt_wdt.h` — `esp_xt_wdt_register_callback(cb, arg)`; only when `SOC_XT_WDT_SUPPORTED`. Header comes from the ESP-IDF HAL, not from `src/bist/`.
- `rom/ets_sys.h` — `ets_delay_us(us)`.
- `esp_attr.h` — `IRAM_ATTR`.
- `soc/soc_caps.h` — `SOC_XT_WDT_SUPPORTED`, `SOC_GPIO_PIN_COUNT`, `SOC_CPU_HAS_PMA`.
- `hal/gpio_ll.h`, `hal/gpio_types.h` — `gpio_num_t`, `gpio_mode_t`, etc.
- `ulp_lp_core_interrupts.h` — `ulp_lp_core_intr_disable()`, `ulp_lp_core_intr_enable()`. Used by `samples/zephyr/remote/src/main.c` to gate LP-core interrupts around `bist_cpu_regs_test()` / `bist_cpu_csr_regs_test()` / `bist_ram_test_march_x()` so the test window is not preempted. Header comes from the ESP-IDF LP-core HAL, not from `src/bist/`.

## Zephyr mbox API (external, referenced by `samples/zephyr` — not in BIST tree)

The Zephyr integration sample uses Zephyr's mailbox driver for HP↔LP IPC. Signatures from `<zephyr/drivers/mbox.h>` (Zephyr tree, not BIST):

```c
typedef uint32_t mbox_channel_id_t;
struct mbox_dt_spec { const struct device *dev; mbox_channel_id_t channel_id; };
struct mbox_msg { const void *tx_data; uint32_t tx_size; };

#define MBOX_DT_SPEC_GET(node_id, name) ...

int       mbox_send_dt(const struct mbox_dt_spec *spec, const struct mbox_msg *msg);  // msg=NULL → signal-only
int       mbox_register_callback_dt(const struct mbox_dt_spec *spec,
                                    void (*cb)(const struct device *, mbox_channel_id_t, void *, struct mbox_msg *),
                                    void *user_data);
int       mbox_set_enabled_dt(const struct mbox_dt_spec *spec, bool enabled);
uint32_t  mbox_max_channels_get_dt(const struct mbox_dt_spec *spec);
int       mbox_mtu_get_dt(const struct mbox_dt_spec *spec);                            // max bytes per TX msg
```

Device-tree binding (`samples/zephyr/remote/boards/esp32c6_devkitc_lpcore.overlay`): a `vnd,mbox-consumer` node with `mboxes = <&mbox0 1>, <&mbox0 0>; mbox-names = "tx", "rx";` and `chosen { zephyr,ipc = &mbox0; zephyr,ipc_shm = &ipc_shm; }`.

## MCP Server Tools (Python, `mcp-server/server.py` — not C, not on-target)

The shipped MCP server exposes six `@mcp.tool()`-decorated Python functions consumed by AI assistants (Cursor / VS Code). This is a developer-productivity tool, excluded from the safety-qualified scope. Signatures transcribed from `mcp-server/server.py`:

```python
def search_bist_docs(query: str, soc_target: str = "") -> str
    # BM25 search over RST docs + READMEs; returns ranked chunks with source file refs.

def get_api_reference(name: str, soc_target: str = "") -> str
    # Look up function/type/module; returns signature + Doxygen + params + returns + Kconfig guards.

def get_architecture_info(topic: str, soc_target: str = "") -> str
    # Fetch architecture / module design / memory model / safety docs chapters.

def search_kconfig_options(query: str) -> str
    # Search Kconfig symbols: name, type, default, depends_on, help text.

def search_source_code(query: str, file_type: str = "all") -> str
    # Locate C function bodies in src/bist; file_type: "header" | "source" | "all".

def get_supported_socs(soc_target: str = "") -> str
    # SoC matrix (CPU, freq, SRAM, PMA, XT WDT, CSR count) for C3/C5/C6/H2.
```

Engine: `BISTSearchEngine` (`mcp-server/search.py`) over `rank_bm25.BM25Okapi`; tokenizer lowercases and strips non-`[a-z0-9_]`. Data lives in `mcp-server/data/{docs,api,kconfig,source,socs}.json`, regenerated by `mcp-server/ingest.py ..` (deterministic, path-independent).

## Validation Fixtures (Python, `conftest.py` — not C, not on-target)

The QEMU/GDB fault-injection harness is driven by these pytest fixtures/classes from the repo-root `conftest.py`:

```python
class QEMU_RISCV:
    # qemu-system-riscv32 -nographic -icount 3 -machine <target> -drive file=...,if=mtd,format=raw
    # debug mode adds: -s -S  (gdbserver on :1234, halted start)
    def start(self, debug=False) -> tuple[process, queue.Queue]
    def stop(self, qemu_process) -> None

class GDB_RISCV:
    # riscv32-esp-elf-gdb build/<app>.elf --command=<script.gdb>
    def attach(self, script: str) -> process   # writes script to gdb_script_temp.gdb, runs it
    def stop(self, gdb_process=None) -> None

@pytest.fixture
def qemu_instance(request, target)        # normal-mode QEMU
@pytest.fixture
def qemu_debug_instance(request, target)  # debug-mode QEMU (-s -S)
@pytest.fixture
def gdb_instance(request)                 # GDB_RISCV bound to --executable
# CLI: --executable=<app_name_no_ext>
```

Target parametrization (`tests/idf_targets.py`): `("esp32c3", "esp32c5", "esp32c6", "esp32h2")`; run one with `--target=<soc>`.
