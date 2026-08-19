---
name: esp-serial-flasher-skill
description: >-
  AI Skill for the ESP Serial Flasher portable C library, used to flash Espressif SoCs
  (ESP8266/ESP32/ESP32-S2/S3/C2/C3/C5/C6/H2/P4/C61) from another host MCU over UART, USB CDC-ACM,
  SPI, or SDIO. Use when users need to build host firmware that programs a target ESP device, read
  MAC/flash/security info, load code to RAM, do MD5-verified or deflate-compressed flashing, or
  implement a custom port. Covers the v2 API (esp_loader_t context, per-protocol init, port vtable).
  Trigger words: "ESP Serial Flasher", "esp-serial-flasher", "esp_serial_flasher", "esp_loader",
  "flash ESP from MCU", "cross-MCU flashing", "serial flasher", "ESP 串口烧录", "跨 MCU 烧录",
  "烧录 ESP", "从 MCU 烧录", "RAM 下载", "loader stub", "flasher stub"
license: Apache-2.0
metadata:
  author: Community
  version: "1.1.0"
---

# esp-serial-flasher-skill

ESP Serial Flasher（`esp_loader`）是一个可移植的 C 库，用于从另一台主机 MCU/单板机/PC 对 Espressif SoC（目标芯片）进行烧录与交互。本 Skill 完全基于仓库 `esp-serial-flasher` v2 API 的真实文档与源码（`include/esp_loader.h`、`include/esp_loader_io.h`、`include/esp_loader_error.h`、`docs/`、`examples/`、`port/`、`Kconfig`）编写，所有函数名、结构体、宏、配置项、文件路径与代码片段均来自真实仓库，绝不臆造。

v2 API 的核心范式：**调用者拥有 `esp_loader_t` 上下文** → 选择对应协议的 init 函数（`esp_loader_init_serial` / `esp_loader_init_spi` / `esp_loader_init_sdio`，它们会自动调用 port 的 `init` 回调）→ 用 `esp_loader_connect[_with_stub]` 连接 → 用独立的操作上下文（`esp_loader_flash_cfg_t` / `esp_loader_mem_cfg_t` / `esp_loader_flash_deflate_cfg_t`）执行烧录/RAM 加载。

## Core Principles

1. **v2 一律使用 `esp_loader_t` 上下文** — 所有公共 API 第一个参数都是 `esp_loader_t *loader`；不再有全局/静态状态。每个目标设备声明一个 loader 实例（两台并行烧录就声明两个）。
2. **协议在运行时选择** — 通过调用不同的 init 函数绑定协议与 port vtable：`esp_loader_init_serial()`（UART / USB CDC-ACM / Linux tty，全特性）、`esp_loader_init_spi()`（仅 RAM 下载）、`esp_loader_init_sdio()`（实验性）。不存在 v1 的编译期 `SERIAL_FLASHER_INTERFACE_*` 开关。
3. **port 以 vtable + 嵌入式 base 暴露** — port 结构体第一个成员必须是 `esp_loader_port_t port`，回调用 `container_of` 取回完整结构体。未实现的回调置 `NULL`（SDIO 的 `change_transmission_rate`/`write`/`read`；非 SPI 的 `spi_set_cs`；非 SDIO 的 `sdio_*`）。
4. **`init` 由 `esp_loader_init_*()` 自动调用** — 不要再单独调用 port 初始化；构造好 port 结构体后直接 `esp_loader_init_serial(&loader, &port.port)`。
5. **烧录/加载状态放在独立的 cfg 结构体** — `esp_loader_flash_cfg_t`、`esp_loader_mem_cfg_t`、`esp_loader_flash_deflate_cfg_t` 由调用者填 `offset`/`image_size`/`block_size` 等公开字段，`_state` 子结构由库初始化，禁止手动改。
6. **`esp_loader_flash_finish()` 必须调用** — 它先做 MD5 校验（`skip_verify=false` 时），再发 flash-end 命令；不调用就等于没写完，且静默跳过校验。v1 的 `reboot` 形参已移除。
7. **`esp_loader_connect()` = ROM bootloader；`esp_loader_connect_with_stub()` = flasher stub** — 仅 serial(SLIP) 接口支持 stub。stub 解锁更高波特率、>2MB flash、deflate 压缩写、快速读 flash。host flash 紧张时只用 `esp_loader_connect()`，链接器 GC 会剥掉约 87KB stub rodata。
8. **image_size 必须事先已知** — 烧录前必须知道镜像总大小（读 SD 卡、收到的缓冲区长度、bin2array 生成的符号）。`offset`/`image_size` 必须 4 字节对齐。
9. **ESP8266 特例** — 不支持改波特率（连 `esp_loader_change_transmission_rate` 会返回 `ESP_LOADER_ERROR_UNSUPPORTED_FUNC`，例子里直接跳过）；不支持无 stub 的 MD5 校验，需 `flash_cfg.skip_verify = true`。
10. **SDIO 自动走 stub** — `esp_loader_connect()` 在 SDIO 上会自动上传 `esp-flasher-stub`，支持完整 stub 命令；SDIO 不支持改速率。目前仅 ESP32-C5/C6 作目标。
11. **每个公共函数都返回 `esp_loader_error_t`** — 用 `RETURN_ON_ERROR(x)` 宏（`include/esp_loader.h`）链式检查。不支持的功能返回 `ESP_LOADER_ERROR_UNSUPPORTED_FUNC`，可据此降级。
12. **bootloader 地址因芯片而异** — 用 `examples/common/example_common.c` 的 `bootloader_addresses[]` 表：ESP8266/C3/S3/C2/H2/C6/C61 = `0x0`，ESP32/S2 = `0x1000`，ESP32-C5/P4 = `0x2000`。分区表固定 `0x8000`，应用固定 `0x10000`。
13. **内置 port 的外设初始化模型分三类** — 选择对应 `recipes/*_host.md`：① **Pico/ESP32**：port 的 `init` 回调自动初始化外设（可设 `dont_initialize_peripheral` 跳过）。② **STM32**：port 的 `init` 为 NULL，外设必须由 STM32CubeMX **预先生成并初始化**，port 只持有 `huart` 句柄。③ **Zephyr/Linux**：不手填 port 结构体——Zephyr 经 DTS `espressif,esp-loader` 节点 + `esp_loader_from_device()` 取 loader；Linux 在 `linux_port_t` 里填设备路径。混淆这三类会导致外设未初始化或符号未定义。

## When to Use

**Applicable:**
- 在主机 MCU（ESP32/STM32/RP2040/RP2350/Zephyr/Linux）上构建烧录器固件，对 ESP 目标芯片编程
- UART / USB CDC-ACM / SPI / SDIO 任一接口的连接与烧录
- 多分区烧录（bootloader + 分区表 + app）、MD5 校验、擦除整片/区域
- RAM 下载并运行（load-to-RAM）、读取目标 MAC / flash 容量 / security info
- deflate 压缩烧录（需 stub）、快速重烧（MD5 比对跳过相同分区）
- 移植到新主机平台（自定义 port vtable）
- 从 v1 迁移到 v2 API

**Not applicable:**
- 在 PC 上用 Python esptool 烧录（那是 esptool 项目，非本库）
- 给目标芯片开发应用固件本身（本库只负责“主机烧录目标”）
- 非 Espressif 目标芯片（STM32、Nordic、RISC-V 通用 SoC 等不在支持矩阵内）
- SPI flash 文件系统（LittleFS/FAT）操作——本库是裸 flash 读写

---

## Scenario Quick Reference (Recipes)

当用户意图匹配下列场景时，**先读对应 recipe**，其中包含完整调用链、分步说明、真实代码与常见错误。

### 连接与基础

| recipe | scenario |
|---|---|
| `recipes/uart_connect_flash.md` | UART 主机连接 ESP 目标并多分区烧录（最常用，ROM bootloader） |
| `recipes/connect_with_stub.md` | 用 flasher stub 连接以解锁高波特率/deflate/快速读 |
| `recipes/get_target_info.md` | 读取目标 MAC / flash 容量 / security info / 芯片型号 |

### 烧录与校验

| recipe | scenario |
|---|---|
| `recipes/flash_partitions.md` | 标准多分区烧录（bootloader+分区表+app）含 MD5 自动校验 |
| `recipes/fast_reflash_md5.md` | 用已知 MD5 比对，仅烧录变化分区（快速重烧） |
| `recipes/deflate_flash.md` | deflate 压缩烧录（节省传输时间，需 stub） |
| `recipes/read_flash.md` | 从目标 flash 读取数据并校验 |

### RAM 下载与擦除

| recipe | scenario |
|---|---|
| `recipes/load_ram_uart.md` | 通过 UART 把程序下载到 RAM 并运行（不走 flash） |
| `recipes/erase_flash.md` | 整片擦除 / 区域擦除 |

### 其它接口

| recipe | scenario |
|---|---|
| `recipes/spi_load_ram.md` | 通过 SPI 接口下载 RAM（仅 RAM 下载） |
| `recipes/sdio_flash.md` | 通过 SDIO 接口烧录（实验性，ESP32-C5/C6 目标） |
| `recipes/usb_cdc_acm.md` | 通过 USB CDC-ACM 主机端口烧录（USB Host） |

### 主机平台集成

> 五个内置 port 各有独立的构建/外设模型。ESP32 主机走 `idf_component_setup.md`，其余按下表。

| recipe | scenario |
|---|---|
| `recipes/linux_host.md` | Linux 主机（PC/树莓派）烧录，DTR/RTS 或 libgpiod 复位（`PORT=LINUX`） |
| `recipes/stm32_host.md` | STM32 主机（STM32CubeMX + HAL，`PORT=STM32`），预初始化外设模型（无 init 回调） |
| `recipes/zephyr_host.md` | Zephyr 主机（west module + device-tree 驱动 `espressif,esp-loader` 节点 + 可选 `esf` shell） |
| `recipes/pi_pico_host.md` | Raspberry Pi Pico / Pico 2 主机（Pico SDK，RP2040 / RP2350 ARM 与 RISC-V，uf2 烧入） |

### 集成与移植

| recipe | scenario |
|---|---|
| `recipes/idf_component_setup.md` | 在 ESP-IDF 项目中以 managed component 形式集成 |
| `recipes/custom_port.md` | 为新主机平台实现 port vtable（`PORT=USER_DEFINED`，五个内置 port 都不覆盖时） |

---

## 目标芯片支持矩阵（来自 README.md）

| 目标 | UART | SPI | SDIO | USB CDC ACM |
|:---:|:---:|:---:|:---:|:---:|
| ESP8266 | ✅ | ❌ | ❌ | ❌ |
| ESP32 | ✅ | ❌ | 🚧 | ❌ |
| ESP32-S2 | ✅ | ❌ | ❌ | ❌ |
| ESP32-S3 | ✅ | ✅ | ❌ | ✅ |
| ESP32-C2 | ✅ | ✅ | ❌ | ❌ |
| ESP32-C3 | ✅ | ✅ | ❌ | ✅ |
| ESP32-H2 | ✅ | ✅ | ❌ | ✅ |
| ESP32-C6 | ✅ | ❌ | ✅ | ✅ |
| ESP32-C5 | ✅ | ❌ | ✅ | ✅ |
| ESP32-P4 | ✅ | 🚧 | ❌ | ✅ |
| ESP32-C61 | ✅ | ❌ | 🚧 | ✅ |

图例：✅ 支持 | ❌ 不支持 | 🚧 开发中

## 特性与接口支持（来自 README.md）

| 特性 | UART | USB CDC ACM | SPI | SDIO |
|:---:|:---:|:---:|:---:|:---:|
| Connect (ROM bootloader) | ✅ | ✅ | ✅ | ✅ |
| Connect with stub | ✅ | ✅ | ❌ | ✅ |
| Secure Download Mode | ✅ | ✅ | ❌ | ❌ |
| Flash write | ✅ | ✅ | ❌ | ✅ |
| 压缩写 deflate | 🔶 | 🔶 | ❌ | ✅ |
| Flash read (fast) | 🔶 | 🔶 | ❌ | ✅ |
| Flash read (slow) | ✅ | ✅ | ❌ | ❌ |
| Flash erase (chip) | ✅ | ✅ | ❌ | ✅ |
| Flash erase (region) | ✅ | ✅ | ❌ | ✅ |
| Flash MD5 verify | ✅ | ✅ | ❌ | ✅ |
| RAM download | ✅ | ✅ | ✅ | ✅ |
| Get security info | ✅ | ✅ | ❌ | ✅ |
| Change baud/clock rate | ✅ | ✅ | ❌ | ❌ |

图例：🔶 需先用 `esp_loader_connect_with_stub()` 连接。

## 标准烧录地址（来自 examples/common/example_common.c）

| 镜像 | 地址 | 说明 |
|---|---|---|
| bootloader (ESP8266) | `0x0` | `bootloader_addresses[ESP8266_CHIP]` |
| bootloader (ESP32, ESP32-S2) | `0x1000` | |
| bootloader (ESP32-C5, ESP32-P4) | `0x2000` | |
| bootloader (其余 C/H/S3) | `0x0` | C3/S3/C2/H2/C6/C61 |
| partition table | `0x8000` | `PARTITION_TABLE_ADDRESS` |
| application | `0x10000` | `APPLICATION_ADDRESS` |

## v2 烧录状态机

```
esp_loader_init_serial/spi/sdio   ──►  (port init 自动调用)
        │
        ▼
esp_loader_connect[_with_stub|_secure_download_mode]
        │  (成功后 esp_loader_get_target() 可用)
        ▼
[可选] esp_loader_change_transmission_rate   (serial only, 非 ESP8266/SDIO)
        │
        ▼
esp_loader_flash_start(&flash_cfg)   ──►  内部擦除目标区间 + 初始化 MD5 累加
        │  循环
        ▼
esp_loader_flash_write(&loader,&flash_cfg,payload,chunk)
        │
        ▼
esp_loader_flash_finish(&loader,&flash_cfg)   ──►  MD5 校验(skip_verify=false) + flash-end
        │
        ▼
esp_loader_reset_target(&loader)   ──►  目标重启运行新固件
        │
        ▼
esp_loader_deinit(&loader)   (可选，释放 port 硬件)
```

---

## Critical Pitfalls (Must Read)

下列是最常见的错误。违反任何一条都会导致烧录失败或行为异常。

### 1. v2 必须先 init，且第一个参数永远是 loader

```c
// ❌ WRONG — v1 风格：无上下文、不 init
loader_port_esp32_init(&config);
esp_loader_connect_args_t args = ESP_LOADER_CONNECT_DEFAULT();
esp_loader_connect(&args);

// ✅ CORRECT — v2：声明 port + loader，init 后每个调用都传 &loader
esp32_port_t port = {
    .port.ops    = &esp32_uart_ops,
    .baud_rate   = 115200,
    .uart_port   = UART_NUM_1,
    .uart_rx_pin = GPIO_NUM_5,
    .uart_tx_pin = GPIO_NUM_4,
    .reset_pin   = GPIO_NUM_25,
    .boot_pin    = GPIO_NUM_26,
};
esp_loader_t loader;
esp_loader_init_serial(&loader, &port.port);   // 自动调用 port->ops->init

esp_loader_connect_args_t args = ESP_LOADER_CONNECT_DEFAULT();
esp_loader_connect(&loader, &args);
```

### 2. 传给 init 的是嵌入式 base，不是整个 port 指针

```c
// ❌ WRONG — 传了完整 port 结构体指针类型不匹配
esp_loader_init_serial(&loader, &port);

// ✅ CORRECT — 传嵌入的 esp_loader_port_t base
esp_loader_init_serial(&loader, &port.port);
```

### 3. `esp_loader_flash_finish()` 不能省

```c
// ❌ WRONG — 只 start/write 就结束，flash-end 没发，校验被静默跳过
esp_loader_flash_start(&loader, &flash_cfg);
while (...) esp_loader_flash_write(&loader, &flash_cfg, buf, n);
// 直接 reset_target —— 目标仍停在 loader，且没校验

// ✅ CORRECT — 写完必须 finish（默认做 MD5 校验），再 reset
esp_loader_flash_start(&loader, &flash_cfg);
while (...) esp_loader_flash_write(&loader, &flash_cfg, buf, n);
esp_loader_flash_finish(&loader, &flash_cfg);   // 校验 + flash-end
esp_loader_reset_target(&loader);
```

### 4. ESP8266 不能改波特率、需跳过 MD5 校验

```c
// ❌ WRONG — 对 ESP8266 改波特率会失败；不跳校验 finish 会报 INVALID_MD5
esp_loader_change_transmission_rate(&loader, 230400); // ESP8266: UNSUPPORTED_FUNC
// 且 flash_cfg.skip_verify 保持 false → ESP8266 无 stub MD5 命令

// ✅ CORRECT — 仅非 ESP8266 才改速率；ESP8266 跳过校验
if (esp_loader_get_target(&loader) != ESP8266_CHIP) {
    esp_loader_change_transmission_rate(&loader, 230400);
}
esp_loader_flash_cfg_t flash_cfg = {
    .offset = addr, .image_size = size, .block_size = 1024,
    .skip_verify = (esp_loader_get_target(&loader) == ESP8266_CHIP), // ESP8266 必须 true
};
```

### 5. flash_cfg 的 `_state` 字段不可手填

```c
// ❌ WRONG — 试图初始化 _state._sequence_number / _md5_context
esp_loader_flash_cfg_t flash_cfg = {
    .offset = 0x10000, .image_size = 0x10000, .block_size = 4096,
    ._state._sequence_number = 0,   // 禁止！
};

// ✅ CORRECT — 只填公开字段；_state 由 esp_loader_flash_start() 初始化
esp_loader_flash_cfg_t flash_cfg = {
    .offset     = 0x10000,
    .image_size = 0x10000,
    .block_size = 4096,
    // skip_verify 缺省 false（启用 MD5 校验）
};
esp_loader_flash_start(&loader, &flash_cfg);
```

### 6. image_size 必须事先已知，且 4 字节对齐

```c
// ❌ WRONG — 不知道总大小就烧录；offset / image_size 未对齐
esp_loader_flash_start(&loader, &flash_cfg); // flash_cfg.image_size 未赋值

// ✅ CORRECT — 先取得镜像大小（此处为 bin2array 生成的符号），对齐校验
extern const uint8_t app_bin[];
extern const uint32_t app_bin_size;
esp_loader_flash_cfg_t flash_cfg = {
    .offset     = 0x10000,            // 4 字节对齐
    .image_size = app_bin_size,       // 已知
    .block_size = 1024,
};
```

### 7. deflate 烧录不会内部累积 MD5，需单独 verify

```c
// ❌ WRONG — 以为 deflate_finish 会校验明文 MD5
esp_loader_flash_deflate_start(&loader, &cfg);
// ...deflate_write...
esp_loader_flash_deflate_finish(&loader, &cfg); // 不做 MD5 校验！

// ✅ CORRECT — 用 esp_loader_flash_verify_known_md5 校验明文内容
esp_loader_flash_deflate_finish(&loader, &cfg);
esp_loader_flash_verify_known_md5(&loader, cfg.offset, cfg.image_size, app_bin_md5);
```

### 8. SPI 接口只支持 RAM 下载

```c
// ❌ WRONG — 用 SPI init 后调用 flash 写
esp_loader_init_spi(&loader, &port.port);
esp_loader_flash_start(&loader, &flash_cfg); // 返回 ESP_LOADER_ERROR_UNSUPPORTED_FUNC

// ✅ CORRECT — SPI 只做 RAM 下载
esp_loader_init_spi(&loader, &port.port);
esp_loader_connect(&loader, &connect_args);
load_ram_binary(&loader, app_bin);   // 内部用 esp_loader_mem_start/write/finish
```

### 9. secure download mode 需提供 flash_size，且 ESP32/ESP8266 不支持

```c
// ❌ WRONG — 不传 flash_size，或对 ESP32/8266 调用
esp_loader_connect_secure_download_mode(&loader, &args, 0);          // 0 无意义
esp_loader_connect_secure_download_mode(&loader, &args, size);       // ESP32/8266: UNSUPPORTED_FUNC

// ✅ CORRECT — 提供真实 flash 字节数，仅在支持的芯片上
esp_loader_connect_secure_download_mode(&loader, &args, 4 * 1024 * 1024); // 4MB
```

### 10. change_transmission_rate 只在 serial 接口且连接后有效

```c
// ❌ WRONG — 连接前改速率，或对 SDIO 改速率
esp_loader_change_transmission_rate(&loader, 230400); // 还没 connect
// SDIO：UNSPECIFIED/UNSUPPORTED_FUNC

// ✅ CORRECT — connect 之后、flash_start 之前改
esp_loader_connect(&loader, &args);
if (esp_loader_get_target(&loader) != ESP8266_CHIP) {
    esp_loader_change_transmission_rate(&loader, 230400);
}
```

### 11. RAM 下载要解析镜像 segment，区分 ESP8266 头长度

```c
// ❌ WRONG — 所有芯片用同样偏移读 segment；ESP8266 无扩展头
uint32_t offset = 0x18; // 对 ESP8266 错误

// ✅ CORRECT — ESP8266 头 0x8，其余 0x18（来自 examples/common/example_common.c）
uint32_t offset = (esp_loader_get_target(loader) == ESP8266_CHIP) ? 0x8 : 0x18;
const esp_loader_bin_header_t *header = (const esp_loader_bin_header_t *)bin;
// 解析 header->segments 个 segment，逐个 esp_loader_mem_start/write
esp_loader_mem_finish(loader, &mem_cfg, header->entrypoint);
```

### 12. USB CDC-ACM 不支持改波特率，需先装 USB Host 驱动

```c
// ❌ WRONG — 直接 init_serial，未先装 usb_host / cdc_acm_host；且尝试改速率
esp_loader_init_serial(&loader, &port.port); // cdc_acm 未 install
connect_to_target(&loader, 230400);          // USB CDC 忽略 line coding

// ✅ CORRECT — 先 usb_host_install + cdc_acm_host_install，速率传 0
usb_host_install(&host_config);
cdc_acm_host_install(NULL);
esp_loader_init_serial(&loader, &port.port);
connect_to_target(&loader, 0);   // USB CDC: 速率参数传 0
```

### 13. stub 会占用约 87KB host flash rodata

```c
// ❌ WRONG — host flash 紧张却无条件用 stub
esp_loader_connect_with_stub(&loader, &args); // 拉入全部 11 个 stub ≈ 87KB

// ✅ CORRECT — host flash 紧张时用 ROM bootloader，链接器 GC 剥掉 stub
esp_loader_connect(&loader, &args); // 不引用 stub → --gc-sections 剥除，≈0KB
```

### 14. 自定义 port 未实现的回调必须置 NULL

```c
// ❌ WRONG — 非 SDIO port 也填了 sdio_write/sdio_read 指针，造成误调度
const esp_loader_port_ops_t my_ops = {
    .write = my_write, .read = my_read,
    .sdio_write = my_sdio_write, // 非 SDIO 不该填
};

// ✅ CORRECT — 不用的回调置 NULL（write/read、spi_set_cs、sdio_* 按需）
const esp_loader_port_ops_t my_ops = {
    .init = my_init, .deinit = NULL,
    .enter_bootloader = my_enter_boot, .reset_target = my_reset,
    .start_timer = my_start_timer, .remaining_time = my_remaining,
    .delay_ms = my_delay, .log = NULL, .log_hex = NULL,
    .change_transmission_rate = my_change_rate,
    .write = my_write, .read = my_read,
    .spi_set_cs = NULL, .sdio_write = NULL, .sdio_read = NULL, .sdio_card_init = NULL,
};
```

---

## Execution Workflow

| Step | Name | Description |
|------|------|-------------|
| 1 | Plan | 明确主机平台（ESP-IDF/STM32/Zephyr/Pico/Linux/自定义）、目标芯片、接口（UART/USB/SPI/SDIO）、是否需要 stub/deflate/校验 |
| 2 | Recipe | 匹配 `recipes/` 中的场景，先读对应 recipe 的完整调用链 |
| 3 | Query | 未覆盖的 API 查 `resources/api_reference.md`；配置项查 `resources/config_reference.md`；陷阱查 `resources/pitfalls.md` |
| 4 | Validate | 核对所有函数签名（v2 必带 `&loader`）、port 结构体字段、include 头、`block_size`/对齐、ESP8266 特例 |
| 5 | Confirm | 向用户呈现方案：port 引脚、init 顺序、连接方式（ROM/stub/secure）、烧录分区表、校验策略 |
| 6 | Execute | **新项目**：参考最接近的 `examples/` 示例（见 `resources/example_list.md`）改造；**已有项目**：原地修改 |
| 7 | Check | 检查：finish 是否调用、`_state` 未手填、ESP8266 skip_verify、stub 体积、init 传 `&port.port` |
| 8 | Build | ESP-IDF: `idf.py build`；Zephyr: `west build`；Pico: cmake+make；Linux: `cmake .. && make` |
| 9 | Verify | 串口观察 `Connected to target` / `Progress: 100 %` / `Flash verified`；目标复位后看 slave_monitor 转发的目标日志 |

### Step 6 Detail — 选择起始示例

| 用户意图 | 起始示例（仓库真实路径） |
|---|---|
| UART 多分区烧录 | `examples/esp32_example/` |
| 快速重烧（MD5 比对） | `examples/esp32_fast_reflash_example/` |
| RAM 下载（UART） | `examples/esp32_load_ram_example/` |
| deflate 压缩烧录 | `examples/esp32_deflate_example/` |
| 读目标信息 | `examples/esp32_get_target_info_example/` |
| 读 flash | `examples/esp32_read_flash_example/` |
| stub 烧录 | `examples/esp32_stub_example/` |
| USB CDC-ACM 烧录 | `examples/esp32_usb_cdc_acm_example/` |
| SPI RAM 下载 | `examples/esp32_spi_load_ram_example/` |
| SDIO 烧录 | `examples/esp32_sdio_example/` |
| SDIO RAM 下载 | `examples/esp32_sdio_load_ram_example/` |
| Linux 主机 | `examples/linux_example/` |
| 树莓派 Pico 主机 | `examples/pi_pico_example/` |
| STM32 主机 | `examples/stm32_example/` |
| Zephyr 主机 | `examples/zephyr_example/` |

复制整个示例目录后修改 port 引脚、目标固件（`target-firmware/*.bin` 经 `bin2array.cmake` 转 C 数组）即可。

---

## Failure Strategies

| Situation | Action |
|---|---|
| API/字段在 `resources/` 找不到 | 立即停止，告知用户该 API 可能不存在或属私有头 |
| `ESP_LOADER_ERROR_TIMEOUT` | 检查 TX/RX 是否接反、reset/boot 引脚、共地、降低波特率、缩短线缆 |
| `ESP_LOADER_ERROR_INVALID_RESPONSE` | 降低波特率、缩短线缆、检查供电是否充足 |
| `ESP_LOADER_ERROR_INVALID_TARGET` | 目标芯片或版本不支持，核对支持矩阵 |
| `ESP_LOADER_ERROR_UNSUPPORTED_FUNC` | 接口不支持该功能（SPI 不能 flash；SDIO/USB 不能改速率；ESP8266 无 stub MD5） |
| `ESP_LOADER_ERROR_INVALID_MD5` | 重新烧录；ESP8266 设 `skip_verify=true`；检查镜像是否完整 |
| `ESP_LOADER_ERROR_INVALID_PARAM` | offset/image_size 未 4 字节对齐；secure download mode 的 flash_size 错误 |
| host flash 不够装 stub | 改用 `esp_loader_connect()`（ROM bootloader），靠 `--gc-sections` 剥 stub |
| 不确定连接方式 | 默认 `esp_loader_connect()` + `ESP_LOADER_CONNECT_DEFAULT()`；需高波特率/deflate 才用 `connect_with_stub` |

## References

- 场景 recipes → `recipes/` 目录
- 公共 API 速查 → `resources/api_reference.md`
- 配置项速查 → `resources/config_reference.md`
- 陷阱汇总 → `resources/pitfalls.md`
- 示例清单 → `resources/example_list.md`
- 仓库源码：`include/esp_loader.h`、`include/esp_loader_io.h`、`include/esp_loader_error.h`
- 仓库文档：`docs/configuration.md`、`docs/hardware-connections.md`、`docs/platform-setup.md`、`docs/supporting-new-platform.md`、`docs/migration-v1-to-v2.md`
- 示例公共助手：`examples/common/example_common.{c,h}`
