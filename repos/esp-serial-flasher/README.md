# esp-serial-flasher-skill

面向 [ESP Serial Flasher](https://github.com/espressif/esp-serial-flasher)（`esp_loader`）的 AI Skill（Claude Code / Agent skill 格式）。本 Skill 让 AI 代理能够**完全基于仓库真实文档与源码**，正确地为主机平台开发烧录 ESP 目标芯片的固件——覆盖 UART / USB CDC-ACM / SPI / SDIO 接口、ROM/stub/secure 连接、多分区烧录、deflate 压缩、RAM 下载、目标信息读取、快速重烧、Linux/STM32/Zephyr/Pico 主机以及自定义 port 移植。

所有 API、结构体、宏、配置项、文件路径、代码片段均来自 `esp-serial-flasher` v2 的 `include/`、`docs/`、`examples/`、`port/`、`Kconfig`，绝不臆造。

## 特性

- **场景驱动 recipes**：14 个 recipe 覆盖连接、烧录、校验、RAM 下载、擦除、各接口（SPI/SDIO/USB/Linux）、ESP-IDF 集成与自定义 port 移植
- **API 速查**：`resources/api_reference.md` 列出全部公共函数签名、枚举、结构体与各平台 port 字段
- **配置速查**：`resources/config_reference.md` 覆盖 Kconfig / CMake 变量、stub 体积、Zephyr DTS
- **陷阱汇总**：`resources/pitfalls.md` 整合 v2 范式、ESP8266 特例、SDIO/SPI/USB 注意事项、错误码语义
- **示例索引**：`resources/example_list.md` 列出仓库全部真实示例及接线
- **Critical Pitfalls**：SKILL.md 给出 14 条最常见的错误做法 vs 正确做法对照

## 目标与主机支持

- **目标芯片**：ESP8266、ESP32、ESP32-S2/S3、ESP32-C2/C3/C5/C6/H2、ESP32-P4、ESP32-C61
- **主机平台**：ESP-IDF v5.5+、STM32 HAL、Zephyr v4.4.0、Raspberry Pi Pico SDK v2.2.0、Linux（libgpiod ≥2.0 可选）、任意 CMake ≥3.22 自定义平台
- **接口**：UART、USB CDC-ACM、SPI（仅 RAM 下载）、SDIO（实验性，ESP32-C5/C6 目标）

## 安装

把本 Skill 目录放入 Claude Code 的 skills 目录即可。两种方式：

### 方式 A：项目级（仅当前项目可用）
复制到项目的 `.claude/skills/`：
```bash
mkdir -p .claude/skills
cp -r esp-serial-flasher-skill .claude/skills/
```

### 方式 B：用户级（所有项目可用）
复制到用户 skills 目录（如 `~/.claude/skills/`）：
```bash
cp -r esp-serial-flasher-skill ~/.claude/skills/
```

安装后，当对话涉及“跨 MCU 烧录 ESP”“ESP Serial Flasher”“esp_loader”“从 MCU 烧录”“RAM 下载”“flasher stub”等意图时，Skill 会被自动触发。

## 目录结构

```
esp-serial-flasher-skill/
├── SKILL.md                  # 主入口：核心原则、支持矩阵、状态机、陷阱、工作流
├── AGENTS.md                 # 工程约定：include 模式、项目结构、构建、codegen 清单
├── recipes/                  # 14 个场景 recipe（中文）
│   ├── uart_connect_flash.md
│   ├── flash_partitions.md
│   ├── connect_with_stub.md
│   ├── get_target_info.md
│   ├── fast_reflash_md5.md
│   ├── deflate_flash.md
│   ├── read_flash.md
│   ├── load_ram_uart.md
│   ├── erase_flash.md
│   ├── spi_load_ram.md
│   ├── sdio_flash.md
│   ├── usb_cdc_acm.md
│   ├── linux_host.md
│   ├── idf_component_setup.md
│   └── custom_port.md
├── resources/                # 速查文档
│   ├── api_reference.md
│   ├── config_reference.md
│   ├── pitfalls.md
│   └── example_list.md
├── README.md                 # 本文件
└── CHANGELOG.md
```

## 使用范围

**适用**：在主机 MCU/SBC/PC 上构建烧录器固件、对 ESP 目标编程；多分区烧录与 MD5 校验；RAM 下载运行；读目标 MAC/flash/security info；deflate 压缩烧录；快速重烧；移植到新主机平台；v1→v2 迁移。

**不适用**：PC 上用 Python esptool 烧录；为目标芯片开发应用固件本身；非 Espressif 目标；SPI flash 文件系统操作。

## 许可证

本 Skill 内容基于 Apache-2.0 许可的 ESP Serial Flasher 仓库编写，仅供 AI 辅助开发参考。
