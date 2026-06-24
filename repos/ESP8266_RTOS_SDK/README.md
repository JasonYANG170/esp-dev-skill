# ESP8266_RTOS_SDK-skill

面向 [ESP8266_RTOS_SDK](https://github.com/espressif/ESP8266_RTOS_SDK)（乐鑫 ESP8266EX 芯片的 esp-idf 风格 FreeRTOS SDK）的 AI 开发技能（Claude Code / Agent Skill 格式）。本技能让 AI 在编写、配置、构建、调试 ESP8266 固件时，调用真实 API、遵循正确初始化顺序、规避已知陷阱，且不臆造任何不存在的接口。

ESP8266_RTOS_SDK 自 v3.0 起采用与 esp-idf 相同的框架（component 化、`app_main` 入口、`menuconfig`/`sdkconfig`、partition table、event loop），仅面向 ESP8266EX 单核芯片。

## 特性

- **场景化 recipes**：覆盖入门/构建、WiFi（Station/SoftAP/扫描/SmartConfig/ESPNOW）、网络（socket/HTTP/OTA）、外设（GPIO/UART/PWM/I2C）、存储（SPIFFS/NVS/OTA），每个 recipe 含触发意图、前置条件、真实代码、常见错误表、参考示例路径。
- **真实 API 参考**：`resources/api_reference.md` 按模块列出仓库头文件中的真实函数签名、结构体、枚举、宏。
- **配置项速查**：`resources/config_reference.md` 汇总 Kconfig/menuconfig 常用符号（Flash、分区表、WiFi、PHY、日志、Example）。
- **陷阱汇总**：`resources/pitfalls.md` 与 `SKILL.md` 的 Critical Pitfalls 按模块归类，每条带错误/正确对照。
- **示例索引**：`resources/example_list.md` 列出仓库 `examples/` 下全部真实示例路径与一句话描述。
- **零臆造**：所有函数名、结构体、Kconfig 符号、文件路径均来自仓库源码；查不到即视为不存在。

## 适用范围

- 芯片：ESP8266EX（Tensilica L106，32-bit 单核，80MHz）
- 框架：ESP8266_RTOS_SDK（esp-idf style，v3.0+）
- 工具链：xtensa-lx106-elf gcc v8.4.0（esp-2020r3）
- 构建：GNU Make（`make menuconfig` / `make flash`）或 CMake / idf.py
- 不适用：ESP8266 NonOS SDK / RTOS_SDK v2（API 完全不同）、ESP32 系列芯片、Arduino-ESP8266 框架。

## 安装

将本目录克隆/拷贝到 Claude Code 的 skills 目录之一：

- 项目级：`<工程根>/.claude/skills/ESP8266_RTOS_SDK-skill/`
- 用户级：`~/.claude/skills/ESP8266_RTOS_SDK-skill/`（Windows 下 `%USERPROFILE%\.claude\skills\...`）

例如：

```bash
mkdir -p ~/.claude/skills
cp -r ESP8266_RTOS_SDK-skill ~/.claude/skills/
```

之后在对话中提到 ESP8266 / WiFi / OTA / menuconfig 等触发词时，技能会被加载；Claude 会优先读 `SKILL.md` 的场景速查，命中场景先读对应 recipe，再按需查 `resources/`。

## 目录结构

```
ESP8266_RTOS_SDK-skill/
├── SKILL.md                      # 主入口：原则、场景速查、陷阱、执行工作流
├── AGENTS.md                     # 工程约定、include 模式、构建工作流、codegen checklist
├── README.md                     # 本文件
├── CHANGELOG.md                  # 版本变更
├── recipes/                      # 场景 recipe（13 个）
│   ├── hello_world_project.md
│   ├── build_and_flash.md
│   ├── wifi_station.md
│   ├── wifi_softap.md
│   ├── smartconfig.md
│   ├── espnow.md
│   ├── gpio.md
│   ├── uart.md
│   ├── pwm.md
│   ├── i2c.md
│   ├── http_request.md
│   ├── http_ota.md
│   └── spiffs.md
└── resources/                    # 速查文档（真实签名/符号/路径）
    ├── api_reference.md
    ├── config_reference.md
    ├── pitfalls.md
    └── example_list.md
```

## 使用建议

1. 新工程：先读 `recipes/hello_world_project.md` 与 `recipes/build_and_flash.md`。
2. WiFi：按角色读 `recipes/wifi_station.md` / `recipes/wifi_softap.md` / `recipes/espnow.md` / `recipes/smartconfig.md`。
3. 网络/OTA：`recipes/http_request.md` / `recipes/http_ota.md`。
4. 外设：`recipes/gpio.md` / `recipes/uart.md` / `recipes/pwm.md` / `recipes/i2c.md`。
5. 存储：`recipes/spiffs.md`。
6. 任何 API 不确定：查 `resources/api_reference.md`；配置项查 `resources/config_reference.md`；遇到坑查 `resources/pitfalls.md`；找现成代码查 `resources/example_list.md`。

## License

Apache-2.0（与 ESP8266_RTOS_SDK 一致）。
