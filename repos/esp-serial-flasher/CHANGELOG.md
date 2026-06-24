# Changelog

本 Skill 遵循 ESP Serial Flasher 仓库的 v2 API。所有内容均基于仓库真实文档与源码。

## [1.1.0] - 2026-06-18

补齐五个内置 port 中缺失的三个主机平台 recipe（STM32 / Zephyr / Raspberry Pi Pico）。此前仅 `linux_host.md` 覆盖 Linux；现五个内置 port 均有专属 recipe。每个 recipe 的函数/结构体/宏/配置项/路径均核对自仓库 `port/*.h`、`docs/platform-setup.md`、各 `examples/*_example/README.md` 与源码。

### 新增 recipes（3 个）

- **`recipes/stm32_host.md`** — STM32 主机（STM32CubeMX + HAL，`PORT=STM32`）。覆盖 CubeMX 外设配置（UART + 两个 GPIO output）、CMake 集成（`add_subdirectory` + `PORT=STM32`）、预初始化外设模型（`stm32_port_t` 持 `huart` 句柄，init 回调为 NULL）、bin2array 镜像注入、CubeMX user label 生成 `TARGET_BOOT_GPIO_Port` 等符号。来源：`examples/stm32_example/`（README 步骤 1-5）、`port/stm32_port.h`、`docs/platform-setup.md`。
- **`recipes/zephyr_host.md`** — Zephyr 主机（west module + device-tree 驱动 port + 可选 `esf` shell）。覆盖 `zephyr/submanifest/esf.yaml` 声明、`espressif,esp-loader` DTS 节点（`uart`/`reset-gpios`/`boot-gpios`/`default-baudrate`/`higher-baudrate`/`num-trials`/`sync-timeout-ms`）、`prj.conf` 与库 `zephyr/Kconfig`、经 `esp_loader_from_device()` / `esp_loader_config_from_device()` / `esp_loader_connect_args_from_device()` 取 loader（不手填 port）、`CONFIG_ESP_SERIAL_FLASHER_SHELL` 启用 `esf reset/info/images/connect/speed/flash/register` 子命令、`west build` 与 ESP ThreadBR 板 overlay。来源：`examples/zephyr_example/`（README、prj.conf、Kconfig、boards/ overlay、src/main.c、src/shell.c）、`port/zephyr_port.h`、`zephyr/dts/bindings/misc/esp-loader/espressif,esp-loader.yaml`、`zephyr/Kconfig`。
- **`recipes/pi_pico_host.md`** — Raspberry Pi Pico / Pico 2 主机（Pico SDK v2.2.0）。覆盖 RP2040（仅 ARM）与 RP2350（`PICO_PLATFORM` 选 ARM 或 RISC-V，两套交叉编译器不可互换）、`PICO_SDK_PATH`/`PICO_TOOLCHAIN_PATH`、`pi_pico_port_t` 字段（init 回调自动初始化外设）、接线表、bin2array 镜像、`.uf2` BOOTSEL 拖拽烧入、minicom 调试。来源：`examples/pi_pico_example/`（README、CMakeLists、main.c）、`port/pi_pico_port.h`、`docs/platform-setup.md`。

### 更新

- **SKILL.md**：新增「主机平台集成」Scenario Quick Reference 子表（4 个 host recipe）；新增 Core Principle 13（内置 port 的外设初始化模型分三类：Pico/ESP32 自动 init、STM32 预初始化、Zephyr/Linux 不手填）；metadata.version `1.0.0` → `1.1.0`。
- **resources/api_reference.md**：补全 port 实例表的 `pi_pico_port_t`/`pi_pico_uart_ops`、`stm32_port_t`/`stm32_uart_ops`、`zephyr_port_t` 真实符号；新增 `stm32_port_t`、`pi_pico_port_t` 字段定义与 Zephyr `esp_loader_config_t` 结构体、DTS 绑定必填属性。
- **resources/example_list.md**：主机平台示例表扩充为含「内置 port」「关键引用」列，标注各 port 的 `PORT=` 值与对应 recipe。

### Grounding

所有新增签名、字段、配置项、文件路径均来自真实仓库：`port/stm32_port.h`、`port/zephyr_port.h`、`port/pi_pico_port.h`、`docs/platform-setup.md`、`examples/{stm32,zephyr,pi_pico}_example/`（README + 源码 + CMakeLists/Kconfig/overlay）、`zephyr/dts/bindings/misc/esp-loader/espressif,esp-loader.yaml`、`zephyr/Kconfig`。无臆造内容。

## [1.0.0] - 2026-06-18

首个正式版本。基于 `esp-serial-flasher` v2 API（`include/esp_loader.h`、`include/esp_loader_io.h`、`include/esp_loader_error.h`）与 `docs/`、`examples/`、`port/`、`Kconfig` 编写。

### 新增

- **SKILL.md**：核心原则（12 条）、目标芯片支持矩阵、特性×接口支持表、标准烧录地址表、v2 烧录状态机、14 条 Critical Pitfalls（错误做法 vs 正确做法）、执行工作流、失败策略。
- **AGENTS.md**：项目上下文、文件命名、include 模式、标准工程结构、典型 main 模式、各平台构建工作流（ESP-IDF/Zephyr/Pico/Linux/自定义）、bin2array 流程、codegen 清单、Do Not Modify 清单。
- **recipes/（14 个）**：
  - `uart_connect_flash.md` — UART 连接 + 多分区烧录（基础）
  - `flash_partitions.md` — 手写 start/write/finish 流程
  - `connect_with_stub.md` — flasher stub 连接
  - `get_target_info.md` — MAC / flash 容量 / security info
  - `fast_reflash_md5.md` — 已知 MD5 比对快速重烧
  - `deflate_flash.md` — deflate 压缩烧录
  - `read_flash.md` — 读 flash 并校验
  - `load_ram_uart.md` — UART RAM 下载运行
  - `erase_flash.md` — 整片/区域擦除
  - `spi_load_ram.md` — SPI 接口 RAM 下载
  - `sdio_flash.md` — SDIO 烧录（实验性）
  - `usb_cdc_acm.md` — USB CDC-ACM 主机烧录
  - `linux_host.md` — Linux 主机（DTR/RTS / libgpiod）
  - `idf_component_setup.md` — ESP-IDF managed component 集成
  - `custom_port.md` — 自定义主机 port 移植
- **resources/api_reference.md**：全部公共函数签名、枚举、结构体、各平台 port 字段、example_common 助手。
- **resources/config_reference.md**：Kconfig / CMake 变量、port 编译选择、日志级别、stub flash 占用、Zephyr DTS。
- **resources/pitfalls.md**：按主题分组的陷阱与错误码语义。
- **resources/example_list.md**：仓库全部 15 个示例目录索引 + 接线表 + 目标固件准备。
- **README.md** / **CHANGELOG.md**。

### Grounding

所有函数名、结构体、宏、配置项、Kconfig 符号、文件路径与代码片段均来自真实仓库（`include/`、`docs/`、`examples/`、`port/`、`Kconfig`、`idf_component.yml`），无臆造内容。
