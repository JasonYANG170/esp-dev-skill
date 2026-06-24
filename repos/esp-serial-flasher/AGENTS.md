# AGENTS.md — Supplementary Agent Guide

> 核心规则、支持矩阵、状态机、陷阱清单、recipes 索引与执行工作流均在 `SKILL.md`。
> 本文件仅记录 `SKILL.md` 未覆盖的工程约定与工具链指引，不重复内容。

## Project Context

- **语言**: C（C99，头文件带 `extern "C"` 守卫，可被 C++ 包含）
- **角色**: 本库是“主机烧录目标”的库，运行在主机 MCU/SBC/PC 上，对 Espressif 目标芯片编程
- **公共 API 头**（稳定，受 semver 保证）: `include/esp_loader.h`、`include/esp_loader_io.h`、`include/esp_loader_error.h`
- **私有头**（勿直接包含）: `private_include/`（`esp_loader_protocol.h`、`esp_targets.h`、`protocol.h`、`slip.h`、`esp_stubs.h`、`loader_log.h`、`md5_hash.h`）
- **工具链 / 构建**:
  - ESP-IDF v5.5+（managed component `espressif/esp-serial-flasher`），`idf.py build`
  - Zephyr v4.4.0 + Zephyr SDK v1.0.1，`west build`
  - Raspberry Pi Pico SDK v2.2.0（RP2040 ARM；RP2350 ARM 或 RISC-V），CMake
  - Linux: gcc/clang + CMake ≥3.22，可选 libgpiod ≥2.0
  - STM32: STM32 HAL + ARM GCC，STM32CubeMX 生成工程
  - 自定义平台: CMake ≥3.22，`set(PORT USER_DEFINED)`

## File Naming

- 源文件: `src/*.c`（核心协议实现），`port/<platform>_port.{c,h}`（每平台一对）
- 公共头: `include/esp_loader*.h`、`include/md5_ctx.h`、`include/serial_io.h`
- 协议实现: `src/protocol_serial.c`、`src/protocol_spi.c`、`src/protocol_sdio.c`、`src/slip.c`
- 目标描述: `src/esp_targets.c` + `private_include/esp_targets.h`
- stub 二进制源（自动生成，勿手改）: `src/stubs/`

## Include Pattern

```c
// 主机应用最小 include（以 ESP-IDF UART port 为例）
#include "esp_loader.h"          // 公共 API（自动包含 esp_loader_error.h / esp_loader_io.h）
#include "esp32_port.h"          // 对应平台的 port 头：esp32_port.h / esp32_spi_port.h /
                                 //   esp32_sdio_port.h / esp32_usb_cdc_acm_port.h /
                                 //   linux_port.h / zephyr_port.h / pi_pico_port.h / stm32_port.h
#include "example_common.h"      // 仅示例辅助（可选），含 connect_to_target / flash_binary / load_ram_binary
```

> 不要包含 `private_include/` 下的头；不要包含 `include/serial_io.h`（旧遗留）。所有稳定 API 都在 `esp_loader.h`。

## Standard Host Project Structure（ESP-IDF 示例）

```
my_flasher/                       # ESP-IDF 项目根
├── CMakeLists.txt
├── main/
│   ├── CMakeLists.txt            # idf_component_register + bin2array.cmake
│   ├── main.c                    # app_main: init port → connect → flash → reset
│   └── target-firmware/          # 目标固件 .bin（经 bin2array 转 C 数组）
│       ├── bootloader.bin
│       ├── partition-table.bin
│       └── app.bin
├── sdkconfig.defaults            # 如 CONFIG_SERIAL_FLASHER_PORT_UART=y
└── idf_component.yml             # 依赖: espressif/esp-serial-flasher
```

非 ESP-IDF（Linux/Pico/自定义）顶层 `CMakeLists.txt` 用 `set(PORT ...)` 选择 port，或 `set(PORT USER_DEFINED)` 自带 port 源（见 `docs/supporting-new-platform.md` Option B）。

## Canonical Init / Main Pattern（v2）

```c
#include "esp_loader.h"
#include "esp32_port.h"        // 换成你的平台 port 头
#include "example_common.h"    // 可选辅助

void app_main(void)
{
    // 1. 构造 port（字段因平台而异）
    esp32_port_t port = {
        .port.ops    = &esp32_uart_ops,
        .baud_rate   = 115200,
        .uart_port   = UART_NUM_1,
        .uart_rx_pin = GPIO_NUM_5,
        .uart_tx_pin = GPIO_NUM_4,
        .reset_pin   = GPIO_NUM_25,
        .boot_pin    = GPIO_NUM_26,
    };

    // 2. 初始化 loader 上下文（自动调用 port->ops->init）
    esp_loader_t loader;
    if (esp_loader_init_serial(&loader, &port.port) != ESP_LOADER_SUCCESS) {
        return; // 硬件初始化失败
    }

    // 3. 连接 + 提速（helper 内部处理 ESP8266 特例）
    if (connect_to_target(&loader, 230400) != ESP_LOADER_SUCCESS) {
        return;
    }

    // 4. 烧录（bootloader 地址随芯片）
    target_chip_t chip = esp_loader_get_target(&loader);
    uint32_t bootloader_addr = get_bootloader_address(chip);
    flash_binary(&loader, bootloader_bin, bootloader_bin_size, bootloader_addr);
    flash_binary(&loader, partition_table_bin, partition_table_bin_size, 0x8000);
    flash_binary(&loader, app_bin, app_bin_size, 0x10000);

    // 5. 复位目标运行新固件
    esp_loader_reset_target(&loader);

    // 6.（可选）释放 port 硬件
    // esp_loader_deinit(&loader);
}
```

`example_common.c` 提供了三个可直接复用的 helper：`connect_to_target` / `connect_to_target_with_stub` / `flash_binary` / `load_ram_binary`，以及 `get_bootloader_address(chip)`。

## Build Workflow（按平台）

### ESP-IDF（managed component）
```bash
idf.py add-dependency "espressif/esp-serial-flasher"
idf.py -p PORT build
idf.py -p PORT flash monitor
```
端口编译在 `menuconfig → ESP serial flasher → Port selection` 或 `sdkconfig.defaults`：`CONFIG_SERIAL_FLASHER_PORT_UART=y`（默认）、`..._SPI=y`、`..._SDIO=y`、`..._USB_CDC_ACM=y`（依赖 `SOC_USB_OTG_SUPPORTED`）。可同时开多个。

### Zephyr
```bash
# west submanifest 加 modules/lib/esp_serial_flasher
west update esp-serial-flasher
west build -b <board> examples/zephyr_example
west flash
```
配置在 `prj.conf`（如 `CONFIG_ESP_SERIAL_FLASHER_UART_BUFSIZE=...`）。

### Raspberry Pi Pico
```bash
cmake -DPICO_BOARD=<board> -DPICO_PLATFORM=<arm|riscv> -DPICO_SDK_PATH=... -B build
cmake --build build
```

### Linux（PC / SBC）
```bash
cd examples/linux_example && mkdir -p build && cd build
cmake ..                              # USB（DTR/RTS 自动复位，无需 libgpiod）
# 或: cmake -DLINUX_PORT_GPIO=ON ..  # SBC 用 libgpiod 控制 reset/boot
make
```
`linux_port_t.gpio_mode` 取 `LINUX_GPIO_NONE` / `LINUX_GPIO_GPIOD` / `LINUX_GPIO_DTR_RTS`。

### 自定义平台（external library）
顶层 `set(PORT USER_DEFINED)`，`add_subdirectory(external/esp-serial-flasher)`，把自写的 `your_port.c` 加入 `flasher` 目标源，include 指向 `include/`。

## bin2array：把目标 .bin 嵌入主机固件

ESP-IDF 示例统一用 `examples/common/bin2array.cmake`，它会生成 `<bin>_bin` / `<bin>_bin_size` / `<bin>_bin_md5`（16 字节）符号：

```cmake
# main/CMakeLists.txt
include(${CMAKE_SOURCE_DIR}/../common/bin2array.cmake)
create_resources(${target_firmware_dir} ${CMAKE_BINARY_DIR}/target_firmware_data.c)
target_sources(${target} PRIVATE ${CMAKE_BINARY_DIR}/target_firmware_data.c)
```
```c
extern const uint8_t app_bin[];
extern const uint32_t app_bin_size;
extern const uint8_t app_bin_md5[];   // 可直接喂给 esp_loader_flash_verify_known_md5
```
> ESP-IDF 自带的 `EMBED_FILES` 不生成 md5，故示例用 bin2array。`MD5` 字段是 16 字节原始哈希（非 hex 字符串），`esp_loader_flash_verify_known_md5` 直接消费它。

## Code Generation Checklist

生成主机烧录代码时逐项核对：

- [ ] 包含 `esp_loader.h` + 平台 port 头；不含私有头 / 旧 `serial_io.h`
- [ ] port 结构体第一个字段是 `esp_loader_port_t port`，且 `.port.ops = &<platform>_<iface>_ops`
- [ ] `esp_loader_init_serial/spi/sdio(&loader, &port.port)` —— 传 base，不是 `&port`
- [ ] 每个 `esp_loader_*` 调用第一个参数都是 `&loader`
- [ ] 连接用 `ESP_LOADER_CONNECT_DEFAULT()`（`sync_timeout=100, trials=10`）
- [ ] `esp_loader_change_transmission_rate` 仅在 serial 接口、连接后、非 ESP8266/SDIO 调用
- [ ] `esp_loader_flash_start` → 循环 `esp_loader_flash_write` → **`esp_loader_flash_finish`**（不可省）
- [ ] `flash_cfg` / `mem_cfg` / `deflate_cfg` 的 `_state` 不手填，由 start 初始化
- [ ] `offset` / `image_size` 4 字节对齐；`image_size` 已知
- [ ] ESP8266 目标：不改波特率、`skip_verify=true`
- [ ] RAM 下载：按 `header->segments` 解析，ESP8266 头偏移 0x8、其余 0x18，最后 `esp_loader_mem_finish(..., header->entrypoint)`
- [ ] SPI 接口只调用 RAM 下载类 API；SDIO 自动走 stub、不改速率
- [ ] USB CDC-ACM：先 `usb_host_install` + `cdc_acm_host_install`，速率参数传 0
- [ ] 自定义 port：未实现回调置 `NULL`；用 `container_of` 取回完整结构体
- [ ] 结束按需 `esp_loader_deinit(&loader)` 释放硬件

## Error Handling Convention

统一用库提供的 `RETURN_ON_ERROR(x)` 宏（`include/esp_loader.h`）做链式检查，或仿照 `example_common.c` 的 `get_error_string()` 打印：

```c
RETURN_ON_ERROR(esp_loader_flash_start(&loader, &flash_cfg));
// 失败时直接 return 该 esp_loader_error_t
```

## Do Not Modify

- 仓库源码：`include/`、`src/`、`port/`、`private_include/`、`src/stubs/`（stub 由 `cmake/gen_stub_sources.py` / `serial_flasher_pull_stubs.cmake` 拉取，勿手改）
- `Kconfig`、`zephyr/Kconfig`、`idf_component.yml`、`CMakeLists.txt`（顶层）
- 本 Skill 的 `SKILL.md` frontmatter（元数据）
