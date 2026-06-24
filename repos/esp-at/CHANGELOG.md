# Changelog

本文件记录 esp-at-skill 的变更。

格式基于 [Keep a Changelog](https://keepachangelog.com/zh-CN/1.1.0/)，版本号遵循 [语义化版本](https://semver.org/lang/zh-CN/)。

## [1.0.0] - 2026-06-18

初始发布。

### 新增

- **SKILL.md**：技能主文件，含 12 条核心原则、When to Use、配方索引、芯片支持矩阵、模块 UART 引脚表、Kconfig 速查、AT 指令回调签名、12 条 Critical Pitfalls（每条含 WRONG/CORRECT 代码对照）、Execution Workflow、Failure Strategies、References。
- **AGENTS.md**：补充约定——项目上下文、文件命名、include 模式、标准自定义组件结构、入口初始化模式、构建工作流、自定义指令代码生成检查清单、函数包裹（`__wrap_`）约定、Do Not Modify 清单。
- **recipes/**（8 个场景配方）：
  - `build_and_flash.md`：本地克隆、安装、配置、编译、烧录。
  - `set_port_pin.md`：修改命令口/日志口引脚（CSV/menuconfig/at.py 三种方式）。
  - `customize_partitions.md`：自定义 `at_customize.csv`、生成并烧录 `at_customize.bin`。
  - `override_module_config.md`：用外部目录覆盖模块配置/补丁/分区/工厂参数/BLE 数据。
  - `add_custom_command.md`：添加自定义 AT 指令（四类型、参数解析、可选参数、阻塞、接收原始数据）。
  - `customize_ble_service.md`：自定义 BLE GATT 服务（gatts_data.csv、perm/UUID 语义）。
  - `ota_upgrade.md`：三种 OTA 方案（USEROTA/CIUPDATE/WEBSERVER）。
  - `at_over_spi_sdio.md`：通过 SPI/SDIO 承载 AT 指令。
- **resources/api_reference.md**：基于 `components/at/include/` 头文件的真实 API 速查（状态/睡眠/解析/结果码枚举、`esp_at_cmd_t` 等结构、注册/端口/响应/Wi-Fi/TCP-IP/HTTP/WebSocket/FS/NVS API、指令集初始化宏、错误码编码）。
- **resources/config_reference.md**：基于 `main/Kconfig` 的全部 AT 配置选项速查。
- **resources/pitfalls.md**：按 7 大类汇总 27 条常见陷阱。
- **resources/example_list.md**：`examples/` 下 8 个真实示例索引及关键源码位置。
- **README.md**：中文技能介绍、功能、安装、支持范围、使用方式。
