# esp-mqtt-skill

面向 Espressif **ESP-MQTT** 客户端组件的 AI Skill（Claude Code / Agent skill 格式）。本 Skill 帮助 AI 代理在 ESP-IDF 项目中正确使用 ESP-MQTT 进行 MQTT 通信（TCP / TLS / WebSocket / WebSocket Secure，以及 MQTT v5.0），所有 API、结构体、Kconfig 选项与代码片段均严格来自 esp-mqtt 仓库的真实文档与源码，杜绝臆造。

## 功能特性

- **场景驱动 recipes**：覆盖 TCP 连接、事件处理、发布/订阅/QoS、mTLS 双向认证、服务端 CA 单向校验、PSK 认证、WebSocket/WSS、MQTT 5.0、遗嘱消息（LWT）、outbox 限流管理、自定义 outbox 共 11 个真实场景。
- **API 速查**：`resources/api_reference.md` 列出 `esp_mqtt_client_*`、`esp_mqtt5_*` 全部函数签名、枚举与 `esp_mqtt_client_config_t` 全字段。
- **配置速查**：`resources/config_reference.md` 汇总全部 Kconfig 项与运行时配置字段。
- **陷阱汇总**：`resources/pitfalls.md` 整理 30 条高频陷阱（初始化顺序、`%.*s` 打印、outbox `-2`、TLS 校验、MQTT5 内存释放等）。
- **示例清单**：`resources/example_list.md` 索引仓库 9 个真实示例及其传输方式与依赖。

## 安装

将本 Skill 目录放入 Claude Code 的 skills 目录（项目级 `.claude/skills/` 或用户级 `~/.claude/skills/`）：

```bash
# 项目级（仅当前项目可用）
git clone <this-skill-repo> .claude/skills/esp-mqtt-skill

# 或用户级（所有项目可用）
git clone <this-skill-repo> ~/.claude/skills/esp-mqtt-skill
```

克隆后重启 Claude Code 会话即可识别。Skill 通过 `SKILL.md` 的 description 与 trigger words 自动激活。

## 支持范围

- **目标芯片**：ESP32 系列（ESP32, ESP32-C2/C3/C5/C6/C61, ESP32-H2, ESP32-P4, ESP32-S2/S3）。
- **工具链**：ESP-IDF >= 5.3，CMake，`idf.py`。
- **组件来源**：ESP-IDF Component Manager（`espressif/mqtt`）或本地克隆。
- **协议**：MQTT v3.1.1 与 v5.0；传输 mqtt/mqtts/ws/wss。

## 目录结构

```
esp-mqtt-skill/
├── SKILL.md              # 核心规则、recipe 索引、陷阱、执行工作流
├── AGENTS.md             # 项目约定、构建流程、代码生成 checklist
├── README.md             # 本文件
├── CHANGELOG.md          # 版本变更
├── recipes/              # 11 个场景 recipe
│   ├── tcp_connect.md
│   ├── event_handling.md
│   ├── publish_subscribe.md
│   ├── tls_mutual_auth.md
│   ├── tls_server_cert.md
│   ├── psk_auth.md
│   ├── ws_wss.md
│   ├── mqtt5.md
│   ├── last_will.md
│   ├── outbox_qos.md
│   └── custom_outbox.md
└── resources/            # 速查文档
    ├── api_reference.md
    ├── config_reference.md
    ├── pitfalls.md
    └── example_list.md
```

## 许可

本 Skill 内容依据 esp-mqtt 仓库（Apache-2.0）整理。Skill 本身仅供 AI 辅助开发使用。
