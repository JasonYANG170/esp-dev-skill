# Changelog

本技能的所有重要变更记录于此文件。格式参考 [Keep a Changelog](https://keepachangelog.com/)。

## [1.1.0] - 2026-06-18

### 新增
- **recipes/deep_sleep_end_device.md**：Deep Sleep End Device 完整 recipe。覆盖与 light sleep 的流程差异（整芯片断电、wake-from-reset、协议栈重新 init + rejoin）、`esp_deep_sleep_start()` + RTC timer + EXT1 双唤醒源、`RTC_DATA_ATTR` 跨睡眠时间戳、`EZB_BDB_SIGNAL_DEVICE_REBOOT` 分支、oneshot timer 延迟断电、partitions.csv 与 sdkconfig 关键项。来源：`examples/sleepy_devices/deep_sleep_end_device/`、`docs/en/faq.rst`。
- **SKILL.md**：新增 Core Principle 13（deep sleep 与 light sleep 是两套流程）；Scenario Quick Reference「设备角色与组网」表加入 `deep_sleep_end_device.md`；版本号 1.0.0 → 1.1.0。
- **resources/api_reference.md**：新增「Deep Sleep 电源管理」章节，收录 `esp_deep_sleep_start`、`esp_sleep_enable_timer_wakeup`、`esp_sleep_enable_ext1_wakeup`、`esp_sleep_get_wakeup_causes`、`gpio_wakeup_enable`、`rtc_gpio_*` 等示例实际调用的 ESP-IDF 签名，及 deep sleep sdkconfig 关键项。
- **resources/example_list.md**：扩充 `examples/sleepy_devices/deep_sleep_end_device/` 条目说明（唤醒源、wake-from-reset、关键 sdkconfig）。

### 依据
- 代码来源：`examples/sleepy_devices/deep_sleep_end_device/main/deep_sleep_end_device.c`、`deep_sleep_end_device.h`、`sdkconfig.defaults`、`partitions.csv`。
- 工具来源：`examples/utils/switch_driver/`（`CONFIG_GPIO_EXT1_WAKEUP_SOURCE` Kconfig）、`examples/utils/alarm_timer/`。
- 文档来源：`docs/en/faq.rst`（Zigbee Light Sleep Mode 章节对比）。

## [1.0.0] - 2026-06-18

首个正式版本，面向 ESP Zigbee SDK v2.x（`esp-zigbee-lib >= 2.0.0`，`ezb_` API）。

### 新增
- **SKILL.md**：12 条核心原则、When to Use、10 篇 Recipes 索引、芯片/角色/BDB 模式/信道掩码/ZCL callback 参考表、14 条带"错误/正确"代码对照的关键陷阱、执行工作流、失败策略。
- **AGENTS.md**：API 分层说明（v2.x `ezb_` vs v1.x `compat/`）、标准项目结构、include 模式、Zigbee 主任务与 signal handler 骨架、锁规约、构建流程、代码生成 checklist。
- **recipes/（10 篇）**：协调器灯（formation/steering）、路由器/终端开关（ZDO match/bind）、休眠终端（light sleep）、ZHA 数据模型、属性写本地 + 主动上报、ZCL 命令发送、ZCL Core Action 接收、网关/RCP、Touchlink、OTA 升级、自定义 cluster。
- **resources/api_reference.md**：按平台层 / Core / App Signals / BDB / AF / ZHA / NWK / APS / Security / ZDO / ZCL / Touchlink 分组的真实 API 速查。
- **resources/config_reference.md**：`idf_component.yml`、Kconfig/sdkconfig、`partitions.csv`、示例私有配置宏、内存配置、抓包密钥。
- **resources/pitfalls.md**：34 条陷阱汇总（API/启动/角色/数据模型/属性/存储/休眠/调试）。
- **resources/example_list.md**：仓库 `examples/` 全部子目录索引（HA 设备、休眠、Touchlink、OTA、网关、自定义、all_device_types_app、utils、测试脚本）。
- **README.md** / **CHANGELOG.md**。

### 依据
- 仓库版本：`ESP_ZIGBEE_VER` 2.0.1（`components/esp-zigbee-lib/include/esp_zigbee_version.h`）。
- 文档来源：`docs/en/`（`introduction.rst`、`developing.rst`、`faq.rst`、`api-reference/`）。
- 代码来源：`components/esp-zigbee-lib/include/ezbee/` 头文件、`examples/` 各示例。
