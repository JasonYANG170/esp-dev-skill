# 项目搭建：构建并运行第一个 BIST 应用

> **适用摘要**: 从零创建一个 ESP-BIST 应用工程，配置 CMake/Ninja 构建、MCUboot 引导、`bist.conf`，并在 QEMU 或真机上运行。

> Version: selected repo/component version; align with the user project and dependency manifest.
> Evidence: `repos/esp-bist/resources/`, source/examples in `repos/esp-bist/`, and this recipe path `repos/esp-bist/recipes/project_setup.md`.
> Validation: draft metadata added from repository routing; verify APIs, Kconfig symbols, and component dependencies against the selected version.

## 触发意图

- "新建 BIST 工程"
- "如何构建 esp-bist"
- "BIST 应用怎么配置"
- "menuconfig / bist.conf"
- "QEMU 运行 BIST"

## 前置条件

| 条件 | 要求 |
|---|---|
| 参考工程 | `samples/standalone/` 或 `tests/cpu_reg_test/` |
| 工具链 | ESP-IDF（`. $IDF_PATH/export.sh`）、CMake ≥ 3.22、Ninja、`riscv32-esp-elf-gcc` |
| 引导 | MCUboot 2.2.0（dev container 中位于 `/opt/mcuboot`） |
| 目标 SoC | `esp32c3` / `esp32c5` / `esp32c6` / `esp32h2` 之一 |

## 分步说明

### 1. 克隆仓库并进入 IDF 环境

```bash
git clone https://github.com/espressif/esp-bist.git
cd esp-bist
. $IDF_PATH/export.sh
```

### 2.（一次性）构建并烧录 MCUboot 引导

```bash
cd /opt/mcuboot/boot/espressif
cmake -DCMAKE_TOOLCHAIN_FILE=tools/toolchain-esp32c3.cmake \
      -DMCUBOOT_TARGET=esp32c3 -DESP_HAL_PATH=$IDF_PATH -B build -GNinja
ninja -C build
ninja -C build flash_boot -DESP_PORT=/dev/ttyUSB0
```
将 `esp32c3` 替换为目标 SoC。

### 3. 复制最接近的参考工程

```bash
# 完整 IEC 60730 集成 → 用 samples/standalone
cp -r samples/standalone ../my_bist_app

# 单项测试（Unity）→ 用 tests/<name>，例如 cpu_reg_test
cp -r tests/cpu_reg_test ../my_cpu_reg_app
```

### 4. 工程文件骨架（来自 `tests/cpu_reg_test/CMakeLists.txt`）

```cmake
cmake_minimum_required(VERSION 3.22)

set(BIST_ROOT_DIR ${CMAKE_CURRENT_LIST_DIR}/../../)
set(APP_NAME my_bist_app)

set(APP_SOURCES
    main.c
)

include(${BIST_ROOT_DIR}/cmake/project.cmake)

project(${APP_NAME} LANGUAGES C ASM)
```

> `cmake/project.cmake` 会自动加载工具链、BIST 库，并注册后处理钩子，运行 `scripts/calculate_crc32.py`（Flash CRC 测试依赖此步骤）。

### 5. `bist.conf`（**必填**，可为空）

单项测试示例（来自 `tests/cpu_reg_test/bist.conf`）：
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

### 6. 构建 / 配置 / 运行

```bash
cmake -DSOC_TARGET=esp32c3 -B build -GNinja
ninja -C build
ninja -C build menuconfig        # 交互式修改 Kconfig
ninja -C build qemu              # QEMU 运行
ninja -C build qemu_debug        # QEMU 调试（另开终端连 gdb :1234）
ninja -C build flash -DESP_PORT=/dev/ttyUSB0
ninja -C build monitor           # Ctrl+] 退出
```

### 7. 运行测试套件（Pytest + Unity + GDB 注入）

```bash
pytest pytest_qemu_* --executable=my_bist_app --soc-target=esp32c3 \
      --junitxml=build/tests/report.xml
pytest pytest_device_* --junitxml=build/tests/report.xml
```

## 常见错误

| 错误 | 原因 | 解决方法 |
|---|---|---|
| 构建报找不到 `esp_xt_wdt.h` | 该头来自 IDF HAL，非 BIST 树 | 用 `#if SOC_XT_WDT_SUPPORTED` 包裹；或在 C6/H2 上不引用 |
| `bist.conf` 不存在 | 应用目录缺该文件，构建失败 | 必须存在，即使为空 |
| `bist_flash_test()` 在自建系统中失败 | 跳过了后处理 CRC 注入 | 必须经 `cmake/project.cmake` 构建 |
| `wdt_init(1)` 返回 `-1` | 低于一个 MWDT tick（≈ 31 µs） | 至少传 500 µs |
| QEMU 跑不起来 | `SOC_TARGET` 与工具链不匹配 | 用 `cmake/esp32c3.cmake` 等支持的四个目标 |

## 参考

- `samples/standalone/CMakeLists.txt`、`samples/standalone/bist.conf`
- `tests/cpu_reg_test/CMakeLists.txt`、`tests/cpu_reg_test/bist.conf`
- `cmake/project.cmake`、`scripts/calculate_crc32.py`
- 上游 `README.md`、`docs/en/get_started.rst`
