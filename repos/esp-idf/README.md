# esp-idf-skill

面向 Espressif **ESP-IDF**（Espressif IoT Development Framework）的 AI 技能（Claude Code / Agent skill 格式）。让 AI 代理能够正确地为 ESP32 全系列 SoC 开发固件——覆盖 Wi-Fi/BLE、外设驱动、NVS/Flash 存储、OTA 升级、分区表、FreeRTOS 任务与 idf.py/CMake 构建系统。

所有 API、结构体、宏、Kconfig 选项、代码片段均取自 ESP-IDF 仓库真实文档与源码，**绝不臆造**。

## 功能特性

- **场景化 recipe（13 篇）**：新建工程、FreeRTOS 任务/队列、GPIO/UART/I2C/SPI/LEDC/GPTimer、Wi-Fi STA/SoftAP、NVS、分区表、OTA、深度睡眠
- **真实 API 速查**：按模块分组的函数签名（`resources/api_reference.md`）
- **Kconfig 配置速查**：分区表/日志/FreeRTOS/Wi-Fi/Flash 等（`resources/config_reference.md`）
- **常见陷阱汇总**：WRONG/CORRECT 对照（`resources/pitfalls.md`）
- **示例工程索引**：仓库 `examples/` 下真实路径表（`resources/example_list.md`）
- **芯片支持矩阵**：esp32 / esp32s2/s3 / esp32c2/c3/c5/c6/c61 / esp32h2/h4/h21 / esp32p4

## 适用范围

- 适用：ESP-IDF 框架的固件开发（C，可选 C++）
- 不适用：ESP8266（用 ESP8266_RTOS_SDK）、Arduino-ESP32、ESP-MATTER/ESP-ADF 等上层框架

## 安装

### 方式一：安装到用户技能目录（全局可用）

```bash
# Claude Code 用户级技能目录
git clone <本仓库> ~/.claude/skills/esp-idf-skill
# 或直接把整个 skills/esp-idf-skill 目录复制过去
```

### 方式二：安装到项目目录（仅本项目可用）

把 `esp-idf-skill` 目录放到项目的 `.claude/skills/` 下：

```
my_project/
└── .claude/
    └── skills/
        └── esp-idf-skill/
            ├── SKILL.md
            ├── AGENTS.md
            ├── recipes/
            └── resources/
```

### 使用

在 Claude Code 中触发：当提到 "ESP-IDF"、"ESP32"、"idf.py"、"app_main"、"Wi-Fi 配网"、"OTA"、"NVS" 等关键词时，技能自动加载。也可直接 `/esp-idf-skill` 调用。

## 目录结构

```
esp-idf-skill/
├── SKILL.md              # 主技能文件（核心原则、芯片矩阵、陷阱、执行流程）
├── AGENTS.md             # 补充约定（工程结构、include、构建、checklist）
├── recipes/              # 13 篇场景 recipe
│   ├── new_project.md
│   ├── freertos_task.md
│   ├── gpio_control.md
│   ├── uart_comm.md
│   ├── i2c_master.md
│   ├── spi_master.md
│   ├── ledc_pwm.md
│   ├── gptimer.md
│   ├── wifi_sta.md
│   ├── softap.md
│   ├── nvs_storage.md
│   ├── partition_table.md
│   ├── ota_update.md
│   └── deep_sleep.md
├── resources/            # 速查文档
│   ├── api_reference.md
│   ├── config_reference.md
│   ├── pitfalls.md
│   └── example_list.md
├── README.md             # 本文件
└── CHANGELOG.md
```

## 许可证

Apache-2.0（与 ESP-IDF 仓库一致）。

## 数据来源

ESP-IDF 仓库本地副本：`D:/esp-skill/espressif-repos/esp-idf`（README、`docs/en/`、`examples/`、`components/*/include`）。
