# STM32 主机烧录（STM32CubeMX / STM32 HAL 内置 port）

> **适用摘要**: 用 STM32（任意 HAL 系列）作主机经 UART 烧录 ESP 目标，走内置 `stm32_port`（`PORT=STM32`）。与 `custom_port.md` 的 USER_DEFINED 不同：STM32 port 不实现 init 回调，而是要求外设由 CubeMX **预先生成并初始化**，调用者只填 `huart` 句柄和 BOOT/RESET 的 GPIO 端口/引脚。无现成工程，按 STM32CubeMX 流程生成。

> Version: selected repo/component version; align with the user project and dependency manifest.
> Evidence: `repos/esp-serial-flasher/resources/`, source/examples in `repos/esp-serial-flasher/`, and this recipe path `repos/esp-serial-flasher/recipes/stm32_host.md`.
> Validation: draft metadata added from repository routing; verify APIs, Kconfig symbols, and component dependencies against the selected version.

## 触发意图

- "STM32 烧录 ESP"
- "STM32CubeMX 集成 esp-serial-flasher"
- "用 stm32_port"
- "PORT=STM32"
- "从 STM32 给 ESP 烧录"

## 前置条件

| 条件 | 要求 |
|---|---|
| 主机 | STM32 任意 HAL 系列（C0/F0/F1/F2/F3/F4/F7/G0/G4/H5/H7/L0/L1/L4/L5/U0/U5/WB/WL）|
| 工具链 | STM32CubeMX（生成 CMake 工程）+ arm-none-eabi-gcc + CMake ≥3.22 + Ninja |
| 参考示例 | `examples/stm32_example/`（仅 README 指南，无现成工程） |
| 参考头 | `port/stm32_port.h`、`docs/platform-setup.md` |

## 分步说明

### 1. STM32CubeMX 配置外设（生成 CMake 工程）

STM32 port 的设计前提是 **外设已由 CubeMX 初始化完成**（port 的 init 回调为 NULL，不复位外设）。在 CubeMX 中启用：

- 一个 **UART**（与 ESP 通信，TX/RX，波特率 115200，8N1，无流控）
- 第二个 **UART**（调试日志，可选）
- 两个 **GPIO output**（控制 ESP 的 `BOOT` 与 `RESET`，初始电平 High）

> 在 **Project Manager** 中将 *Toolchain / IDE* 设为 **CMake**，再 **Generate Code**。

### 2. CMake 集成（在 CubeMX 生成的顶层 CMakeLists.txt 里加）

CubeMX 生成的顶层 `CMakeLists.txt` 中，在 `add_subdirectory(cm_s` 或 `add_subdirectory(cmake/stm32cubemx)` **之后** 加入（来自 `examples/stm32_example/README.md` 的真实片段）：

```cmake
set(FLASHER_DIR /path/to/esp-serial-flasher)   # 改成你的 checkout

set(PORT STM32)                                  # 关键：选内置 STM32 port
add_subdirectory(${FLASHER_DIR} ${CMAKE_BINARY_DIR}/flasher_lib)
```

再链接库到可执行文件：

```cmake
target_link_libraries(${CMAKE_PROJECT_NAME}
    stm32cubemx
    flasher
)
```

### 3. 构造 stm32_port（预初始化外设模型）

`stm32_port_t`（`port/stm32_port.h`）与 ESP32/Linux port 的关键差别：**没有 baudrate / 引脚号 / init 回调**——UART 已由 HAL 初始化好，port 只持有 `huart` 句柄与两个 GPIO 描述。

```c
#include "stm32_port.h"
#include "esp_loader.h"

stm32_port_t port = {
    .port.ops     = &stm32_uart_ops,
    .huart        = &huart2,                    // CubeMX 生成的 UART 句柄
    .port_boot    = TARGET_BOOT_GPIO_Port,      // CubeMX user label 生成
    .pin_num_boot = TARGET_BOOT_Pin,
    .port_rst     = TARGET_RESET_GPIO_Port,
    .pin_num_rst  = TARGET_RESET_Pin,
};

esp_loader_t loader;
esp_loader_init_serial(&loader, &port.port);    // STM32 port init=NULL，不重新初始化外设
```

> `TARGET_BOOT_GPIO_Port` / `TARGET_BOOT_Pin` 等符号是 CubeMX 在你给 GPIO pin 赋 user label 时自动生成的 `main.h` 宏。若不用 label，在 `USER CODE BEGIN PD` 区手动 `#define` 这些常量。

`stm32_port.h` 会用 `__has_include` 按家族探测正确的 HAL 头（如 `stm32h7xx_hal.h`），因此只需把对应系列的 HAL Drivers include 目录传给编译器即可，无需手动选系列。

### 4. 连接 + 烧录（USER CODE 区）

`examples/stm32_example/README.md` 给出的核心片段（放进 CubeMX 的 `USER CODE BEGIN 2`）：

```c
esp_loader_connect_args_t connect_cfg = ESP_LOADER_CONNECT_DEFAULT();
if (esp_loader_connect(&loader, &connect_cfg) != ESP_LOADER_SUCCESS) {
    Error_Handler();
}

if (esp_loader_get_target(&loader) != ESP8266_CHIP) {
    esp_loader_change_transmission_rate(&loader, 921600);
}
```

烧录循环本身复用 `examples/common/example_common.c` 的 `flash_binary(&loader, bin, size, offset)` 助手（STM32 README 明确指向 `examples/esp32_example` 参考完整流程）。

### 5. 提供目标固件镜像（bin2array）

STM32 工程通常把镜像编译进固件。用仓库的 `bin2array.cmake` 把 `.bin` 目录转成 C 数组：

```cmake
include(${FLASHER_DIR}/examples/common/bin2array.cmake)
create_resources(/path/to/binaries
                 ${CMAKE_CURRENT_BINARY_DIR}/target_firmware_data.c)
set_property(SOURCE ${CMAKE_CURRENT_BINARY_DIR}/target_firmware_data.c PROPERTY GENERATED 1)

target_sources(${CMAKE_PROJECT_NAME} PRIVATE
    ${CMAKE_CURRENT_BINARY_DIR}/target_firmware_data.c
)
```

生成的 `extern` 符号（如 `bootloader_bin` / `bootloader_bin_size`）可直接在代码里用。镜像也可从外部 flash 或通信接口动态获取——只要烧录前已知 `image_size`。

### 6. 构建

```bash
cmake -B build -G Ninja
cmake --build build
```

若生成了 CubeMX presets：

```bash
cmake --preset Debug
cmake --build --preset Debug
```

## 常见错误

| 错误 | 原因 | 解决方法 |
|---|---|---|
| `No STM32 HAL header found` 编译错误 | HAL Drivers include 目录未传给编译器 | 在 CubeMX 工程的 include 路径里加入 `Drivers/STM32xxxx_HAL_Driver/Inc` |
| 连接超时 | UART 波特率不是 115200，或 TX/RX 接反 | CubeMX 里 ESP 通信 UART 设 115200 8N1；核对 TX↔RX 交叉接线 |
| BOOT/RESET 引脚无效 | 用了物理引脚而非 CubeMX label | 给 GPIO 赋 user label，或在 `USER CODE BEGIN PD` 手动 `#define` |
| `TARGET_BOOT_GPIO_Port` 未定义 | 没给 GPIO 设 label | CubeMX 里设 user label（如 `TARGET_BOOT`），或手动定义 |
| 烧录卡在 0% | 外设没真正初始化（port init=NULL 假设已初始化） | 确认 CubeMX 生成的 `MX_USART2_UART_Init()` 已在 main 里调用 |
| 镜像符号找不到 | 没把生成的 `.c` 加入 target_sources | 用 `set_property(... GENERATED 1)` 并 `target_sources` |

## 参考项目

- `examples/stm32_example/` — STM32 主机集成指南（README 步骤 1-5，无现成工程）
- `port/stm32_port.h` — `stm32_port_t`、`stm32_uart_ops`、`__has_include` 系列探测
- `docs/platform-setup.md` — STM32 Setup 章节（"chip-specific, generated by STM32CubeMX"）
- `examples/common/example_common.c` / `bin2array.cmake` — `flash_binary` 助手与 bin→array 转换
- `docs/hardware-connections.md` — UART 接线
