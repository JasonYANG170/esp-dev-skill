# esp-wdf-skill

面向 [Espressif ESP-WDF](https://github.com/espressif/esp-wdf)（WebAssembly Development Framework）的 AI Skill（Claude Code / Agent Skill 格式）。ESP-WDF 是运行于 ESP32 系列芯片上的 WebAssembly 应用开发框架，配套运行时为 ESP-WASMachine。本 Skill 让 AI 代理能够基于仓库真实文档与源码，正确地创建、编译、调试 ESP-WDF 的 WASM 应用——所有 API、结构体、宏、配置项均来自 esp-wdf 仓库，绝不臆造。

## 功能特性

- **场景化配方（recipes/）**：覆盖 10 个真实场景——新建工程、App Framework 定时器、事件发布/订阅、请求/响应、GPIO、I2C 传感器、SPI/LEDC、UART/文件系统、BSD socket 网络、pthread 多线程、LVGL 图形、云服务（HTTP/MQTT/RainMaker/配网）。
- **真实 API 速查（resources/api_reference.md）**：WAMR App Framework API（`api_timer_*`、`api_publish_event`、`api_register_resource_handler` 等）、attr_container、外设 VFS/ioctl、LVGL 访问器，签名取自仓库头文件。
- **配置参考（resources/config_reference.md）**：编译器选项、App Framework 导出开关、扩展适配开关、各示例 Kconfig 项与典型 `sdkconfig.defaults`。
- **常见陷阱（resources/pitfalls.md）**：沙箱越界、入口形态二选一、网络三件齐备、LVGL 加锁等 12 条。
- **示例索引（resources/example_list.md）**：仓库 `examples/` 下全部真实示例路径与一句话描述。

## 安装

将本 Skill 克隆/复制到 Claude Code 的 skills 目录即可。

**项目级**（仅当前项目可用）：

```
<project>/.claude/skills/esp-wdf-skill/
```

**用户级**（所有项目可用）：

```
~/.claude/skills/esp-wdf-skill/
```

目录结构：

```
esp-wdf-skill/
├── SKILL.md              # Skill 主入口（frontmatter + 正文）
├── AGENTS.md             # 补充代理约定
├── README.md             # 本文件
├── CHANGELOG.md          # 版本变更
├── recipes/              # 场景配方（10 个）
│   ├── new_wasm_app.md
│   ├── app_framework_timer.md
│   ├── event_pub_sub.md
│   ├── request_response.md
│   ├── gpio_vfs.md
│   ├── i2c_sensor.md
│   ├── spi_ledc.md
│   ├── uart_filesystem.md
│   ├── sockets_network.md
│   ├── multithread.md
│   ├── lvgl_gui.md
│   └── cloud_protocols.md
└── resources/            # 速查文档
    ├── api_reference.md
    ├── config_reference.md
    ├── pitfalls.md
    └── example_list.md
```

安装后重启 Claude Code，触发词（如 "ESP-WDF"、"WASM 应用"、"WAMR"、"on_init"、"api_register_resource_handler" 等）即可激活本 Skill。

## 支持范围

- **目标芯片**：ESP32 系列（由 ESP-WASMachine 宿主支持的具体 BSP 决定，如 esp-box、esp32_p4_function_ev_board）
- **构建产物**：`.wasm`（字节码）/ `.aot`（AOT，需 wamrc）
- **覆盖模块**：WAMR App Framework（timer/event/request）、外设 VFS/ioctl（GPIO/UART/I2C/SPI/LEDC）、文件系统、BSD socket、pthread、LVGL、ESP HTTP Client、ESP-MQTT、ESP-RainMaker、Wi-Fi Provisioning

## 许可

Apache-2.0（与 esp-wdf 仓库组件许可一致）。
