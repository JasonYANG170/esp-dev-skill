# esp-insights-skill

针对 [esp-insights](https://github.com/espressif/esp-insights)（ESP-Insights 远程诊断 / 可观测性框架）的 AI Skill，面向 Claude Code / Agent Skill 格式。让 AI 代理在 ESP-IDF 项目中正确集成、配置、调试 ESP-Insights —— 所有 API、结构体、宏、Kconfig、代码片段均来自真实仓库文档与头文件，绝不臆造。

## 这是什么

ESP-Insights 是 Espressif 的远程诊断方案：在设备运行时采集错误/告警日志、自定义事件、重启原因、core dump 摘要、指标（metrics）与变量（variables），经 CBOR 编码后通过 HTTPS 或 MQTT(TLS) 上报到 ESP Insights / ESP RainMaker 云端，开发者在 Web 仪表盘查看现场设备健康状态。

本 Skill 提供：

- **场景化配方（recipes）**：HTTPS 接入、MQTT+Claiming、自定义 transport、日志采集、core dump、自定义 metrics/variables、系统 metrics、数据存储调优、运行时控制 —— 每篇含可复制代码、常见错误表、真实参考路径。
- **API 速查（resources/api_reference.md）**：按头文件分组的真实函数签名（含 metadata 1.0/2.0 双分支）。
- **配置参考（resources/config_reference.md）**：三个组件的全部 Kconfig 符号与默认值。
- **陷阱汇总（resources/pitfalls.md）**：从 README/FEATURES/CHANGELOG 提炼的 gotchas。
- **示例索引（resources/example_list.md）**：仓库内真实 example 路径与文件作用。

## 特性

- 中文叙述为主，代码 / API / 技术术语保留英文
- 全部内容 grounded 于仓库 `components/*/include/*.h`、`components/*/Kconfig`、`examples/`、`README.md`、`FEATURES.md`、`CHANGELOG.md`
- 覆盖 ESP32 全系 + ESP-IDF >= v5.1（推荐 v5.5，已兼容 v6.0；4.x 仅 `idf_4_x_compat` 分支）

## 安装

将本 Skill 目录放入 Claude Code 的 skills 目录之一即可被发现：

```bash
# 项目级（随仓库走）
git clone <this-skill-repo> .claude/skills/esp-insights-skill

# 或用户级（所有项目可用）
# Windows: %USERPROFILE%\.claude\skills\esp-insights-skill
# Linux/macOS: ~/.claude/skills/esp-insights-skill
```

确保 `SKILL.md` 与 `AGENTS.md` 位于 `esp-insights-skill/` 根，`recipes/` 与 `resources/` 为子目录。

## 使用范围

**适用：** 在 ESP-IDF 项目集成 ESP-Insights；采集日志/事件/core dump/metrics/variables；配置 HTTPS/MQTT/自定义 transport；调优 RTC 数据存储；启用 command-response。

**不适用：** 非 ESP32 芯片；纯本地调试无网络；ESP-IDF 4.x（除非 `idf_4_x_compat` 分支）；PCB/硬件设计；ESP RainMaker 业务逻辑本身（仅当复用其 transport 时相关）。

详细触发词与原则见 `SKILL.md`。

## License

随仓库 Apache-2.0（见 `LICENSE` 引用）。Skill 文档内容为社区整理。
