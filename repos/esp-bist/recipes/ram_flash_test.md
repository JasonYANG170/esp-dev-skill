# RAM March 测试与 Flash CRC32 完整性测试

> **适用摘要**: 用 `bist_ram_test_march_a()` / `bist_ram_test_march_x()` 做非破坏式 RAM 完整性测试（IEC 60730 4.2）；用 `bist_flash_test()` 比对运行时 CRC32 与后处理注入的参考值（IEC 60730 4.1）。

## 触发意图

- "RAM 自检 / March A / March X"
- "Flash 完整性 / CRC32"
- "非易失内存校验"
- "耦合故障 / transition 故障"
- "IEC 60730 4.1 / 4.2"

## 前置条件

| 条件 | 要求 |
|---|---|
| 参考工程 | `tests/ram_test/main.c`、`tests/flash_test/main.c` |
| 头文件 | `bist_esp.h`（含 `bist_ram.h`、`bist_flash.h`） |
| 配置 | `CONFIG_ESP_BIST_MEMORY_RAM_TEST=y`、`CONFIG_ESP_BIST_MEMORY_FLASH_TEST=y`、`CONFIG_BIST_FLASH_TEST_CHUNK_SIZE`（默认 `0x1000`）、`CONFIG_ESP_BIST_FLASH_SIZE` |
| 链接器 | RAM 区 `_bist_ram_test_start`..`_dram0_end`；Flash CRC 段 `.crc_section_text` / `.crc_section_data` 与符号 `_crc_section_text_start` / `_crc_section_data_start` |
| 构建 | 必须经 `cmake/project.cmake`，触发 `scripts/calculate_crc32.py` 后处理 |

## 分步说明

### 1. RAM March 测试原理（来自 `bist_ram.h` / `docs/en/module_design_and_coding.rst`）

- 测试前把目标区备份到 `.dram0.safe_ram` 的 `backup_chunk`，并把栈重定位到同段的 256 字节 `ram_test_stack`，确保 march 算法可以测**整段** DRAM（含正常栈）而不破坏自身栈帧。
- **March A（3 步）**：①升序写 0 → ②升序读 0 写 1 → ③升序读 1。
- **March X（6 步）**：①升序写 0 → ②升序读 0 写 1 → ③升序读 1 → ④降序读 1 写 0 → ⑤升序读 0 写 1 → ⑥降序读 1 写 0。比 March A 更全面。

### 2. RAM 单测（来自 `tests/ram_test/main.c`）

```c
#include "bist_esp.h"
#include "unity.h"
void setUp(void) {}
void tearDown(void) {}

void test_BIST_ram_march_a(void) {
    TEST_ASSERT_EQUAL(BIST_ESP_OK, bist_ram_test_march_a());
}
void test_BIST_ram_march_x(void) {
    TEST_ASSERT_EQUAL(BIST_ESP_OK, bist_ram_test_march_x());
}

int main(void) {
    UNITY_BEGIN();
    RUN_TEST(test_BIST_ram_march_a);
    RUN_TEST(test_BIST_ram_march_x);
    return UNITY_END();
}
```

### 3. Flash CRC 原理（来自 `bist_flash.h` / `src/bist/README.md`）

- 多项式 `0x04C11DB7`，标准 CRC-32。
- 后处理脚本 `scripts/calculate_crc32.py`：用 `objcopy --dump-section` 取出 `.flash.text` / `.flash.rodata`，算 CRC，再用 `objcopy --update-section` 注入 `.crc_section_text` / `.crc_section_data`（位于 DROM `drom0_0_seg`）。
- 运行时 `bist_flash_test()` 用同一张 CRC 表按 `CONFIG_BIST_FLASH_TEST_CHUNK_SIZE` 分块重算，与 `_crc_section_text_start` / `_crc_section_data_start` 处的参考值比对。

### 4. Flash 单测（来自 `tests/flash_test/main.c`）

```c
#include "bist_esp.h"
#include "unity.h"
void setUp(void) {}
void tearDown(void) {}

void test_BIST_flash(void) {
    TEST_ASSERT_EQUAL(BIST_ESP_OK, bist_flash_test());
}

int main(void) {
    UNITY_BEGIN();
    RUN_TEST(test_BIST_flash);
    return UNITY_END();
}
```

### 5. 集成进 post-boot 自检（来自 `samples/standalone/main.c`）

```c
static void post_boot_tests(void)
{
    bist_esp_err_t err;

    err = bist_ram_test_march_a();           // 或 march_x，按安全需求选
    if (err == BIST_ESP_RAM_TEST_ERR) { ESP_LOGE(TAG, "RAM test failed"); fail_safe_exit(); }

    err = bist_flash_test();                 // 依赖后处理注入的 CRC
    if (err == BIST_ESP_FLASH_TEST_ERR) { ESP_LOGE(TAG, "Flash test failed"); fail_safe_exit(); }
}
```

### 6. 自定义 `bist.conf`

```
CONFIG_ESP_BIST_MEMORY_RAM_TEST=y
CONFIG_ESP_BIST_MEMORY_FLASH_TEST=y
CONFIG_BIST_FLASH_TEST_CHUNK_SIZE=0x1000
CONFIG_ESP_BIST_FLASH_SIZE=0x400000
```

## 常见错误

| 错误 | 原因 | 解决方法 |
|---|---|---|
| `bist_flash_test()` 全新构建即失败 | 跳过了后处理 CRC 注入 | 必须经 `cmake/project.cmake` 构建（自动跑 `scripts/calculate_crc32.py`） |
| RAM 测试破坏应用数据 | 误以为测试可随时跑 | 库已备份/恢复；但属启动期一次性测试，勿在运行时频繁调用 |
| 自定义链接脚本后 RAM 测试区错位 | 改动了 `_bist_ram_test_start` / `_dram0_end` | 保持 `src/soc/<target>/ld/linker.ld` 原段定义 |
| `.dram0.safe_ram` 被纳入测试区 | 自定义链接脚本把 safe_ram 放进了测试区 | 该段必须位于 `_bist_ram_test_start` **之下** |
| Flash 测试耗时过长 | chunk 太小 | 调大 `CONFIG_BIST_FLASH_TEST_CHUNK_SIZE`（默认 4 KB） |
| CRC 注入后仍比对失败 | 自改了 CRC 表或多项式 | 保持 `0x04C11DB7` 与库内置表一致 |

## 参考

- `tests/ram_test/main.c`、`tests/flash_test/main.c`
- `src/bist/core/memory/include/bist_ram.h`、`bist_flash.h`
- `scripts/calculate_crc32.py`
- `docs/en/module_design_and_coding.rst`（`ram-test`、`flash-test`、`post-build-crc-injection` 节）
- `samples/standalone/main.c`（集成位置）
