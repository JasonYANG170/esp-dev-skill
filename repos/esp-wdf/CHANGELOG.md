# Changelog

本文件记录 esp-wdf-skill 的版本变更。

格式参考 [Keep a Changelog](https://keepachangelog.com/)，版本号遵循 [Semantic Versioning](https://semver.org/)。

## [1.1.0] - 2026-06-22

### Added
- `recipes/i2c_touch_gpio_irq.md`：I2C + GPIO READY 门控读取 TT21100 电容触摸屏——填补“双外设组合（I2C + GPIO 中断门控）、变长结构化多字节报告两步读取（先读 2 字节长度字，再读该长度字节）、on_init 形态与 VFS I/O 混用”这一常见 HMI 输入模式的空白。所有结构体（`touch_record_t`/`report_hdr_t`/`touch_report_t`/`button_report_t`）、宏（`TT21100_PACK_ID_TOUCH/BUTTON`、`I2C_TT21100_ADDRESS`）、Kconfig（`TT21100_READY_PIN_NUM`、`I2C_SDA_SCL_PIN_PULLUP`）均取自 `examples/peripherals/i2c/i2c_tt21100`。
- `SKILL.md`：Scenario Quick Reference“外设（VFS/ioctl）”表新增 `i2c_touch_gpio_irq.md` 一行。
- `resources/api_reference.md`：第 3.3 节补充 GPIO 电平 `read` 输入、两步变长 I2C 读取模式代码片段，并标注 `esp_gpio_ioctl.h` 注释与位值方向相反的已知陷阱。
- `resources/example_list.md`：充实 `examples/peripherals/i2c/i2c_tt21100` 描述（两步读 + READY 门控 + 自带 Kconfig）。

## [1.0.0] - 2026-06-18

首个发布版本，全部内容基于 esp-wdf 仓库的真实文档（`README.md` / `README_CN.md`）与源码（`components/wamr/app-framework/`、`components/wamr/libc-builtin-extended/include/ioctl/`、`components/extended_wasm_app/`、`examples/`、`Kconfig`、`CMakeLists.txt`）。

### Added
- `SKILL.md`：Skill 主入口，含 12 条核心原则、12 条 Critical Pitfalls（每条含 WRONG/CORRECT 代码）、执行工作流与失败策略。
- `AGENTS.md`：项目上下文、文件命名、include 模式、标准工程结构、入口模板、构建流程、代码生成清单。
- `recipes/`：12 个场景配方——
  - `new_wasm_app.md` 新建并编译 WASM 工程
  - `app_framework_timer.md` App Framework + 定时器
  - `event_pub_sub.md` 事件发布/订阅
  - `request_response.md` 应用间请求/响应
  - `gpio_vfs.md` GPIO（VFS/ioctl）
  - `i2c_sensor.md` I2C 读 BH1750
  - `spi_ledc.md` SPI 主机 + LEDC 调光
  - `uart_filesystem.md` UART 输出 + 文件系统
  - `sockets_network.md` BSD socket 网络
  - `multithread.md` pthread 多线程
  - `lvgl_gui.md` LVGL 图形（访问器 API）
  - `cloud_protocols.md` HTTP / MQTT / RainMaker / 配网
- `resources/api_reference.md`：WAMR App Framework、attr_container、外设 ioctl、socket、LVGL/HTTP/MQTT/RainMaker 真实 API 速查。
- `resources/config_reference.md`：编译器选项、App Framework 导出开关、扩展适配开关、各示例 Kconfig 项与 sdkconfig.defaults 模式。
- `resources/pitfalls.md`：12 条汇总陷阱。
- `resources/example_list.md`：仓库 `examples/` 全部真实示例索引。
- `README.md`：中文介绍、安装说明、支持范围。
