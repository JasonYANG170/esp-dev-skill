# esp-now-skill

面向乐鑫 [ESP-NOW](https://github.com/espressif/esp-now) 无连接 Wi-Fi 通信组件的 AI 技能（Claude Code / Agent skill 格式）。本组件在 ESP-IDF 原生 `esp_now` 协议之上封装了数据收发、设备控制与绑定、安全握手、批�� OTA、Wi-Fi 配网、无线调试与节点间时间同步等高级能力。本技能提供场景化 recipes、完整 API 速查、Kconfig 配置参考与常见陷阱，所有函数名、结构体、宏与示例代码均取自 `espressif/esp-now` 仓库真实源码，杜绝臆造。

## 特性

- **场景驱动**：11 个 recipe 覆盖收发、控制、安全、OTA、配网、调试、时间同步、低功耗、存储工具等真实用例
- **API 速查**：`resources/api_reference.md` 按模块（espnow / ctrl / security / ota / prov / time / utils / debug）汇总真实签名
- **配置参考**：`resources/config_reference.md` 列出全部 Kconfig 选项（安全、light sleep、control 自动信道、OTA 重传、任务栈/优先级、NVS、调试日志等）
- **陷阱汇总**：`resources/pitfalls.md` 集中 39 条常见错误与修正
- **示例索引**：`resources/example_list.md` 列出仓库全部示例及下载命令
- **中文为主**：说明文字用中文，代码/API/技术术语保留英文

## 适用范围

- 目标芯片：ESP32 / ESP32-C2 / C3 / C6 / S2 / S3（推荐）
- 工具链：ESP-IDF >= v4.4，`idf.py` 构建
- 组件：`espressif/esp-now`（v2.5.x，经 ESP Component Registry 下发）

不适用：原生 `esp_now_*` 低层 API 问题（请查 ESP-IDF 文档）、非 ESP32 系列、BLE Mesh、Wi-Fi socket。

## 安装

将本技能目录放入 Claude Code 的 skills 目录之一：

- 项目级：`<project>/.claude/skills/esp-now-skill`
- 用户级：`~/.claude/skills/esp-now-skill`

```shell
# 克隆到用户级 skills 目录
git clone <your-skill-repo> ~/.claude/skills/esp-now-skill

# 或直接复制本目录
cp -r /path/to/esp-now-skill ~/.claude/skills/
```

放置后，Claude Code 在匹配到 ESP-NOW 相关意图时会自动加载本技能。

## 目录结构

```
esp-now-skill/
├── SKILL.md                    # 技能主文件：原则、场景索引、陷阱、执行流程
├── AGENTS.md                   # 补充约定：工程结构、构建流程、代码生成 checklist
├── README.md                   # 本文件
├── CHANGELOG.md                # 变更记录
├── recipes/                    # 场景化方案（11 个）
│   ├── get_started_send_recv.md
│   ├── unicast_and_group.md
│   ├── control_initiator.md
│   ├── control_responder.md
│   ├── coin_cell_switch.md
│   ├── security_handshake.md
│   ├── ota_batch_upgrade.md
│   ├── provisioning_wifi.md
│   ├── wireless_debug.md
│   ├── time_sync.md
│   └── storage_utils.md
└── resources/                  # 速查文档
    ├── api_reference.md
    ├── config_reference.md
    ├── pitfalls.md
    └── example_list.md
```

## 许可证

本技能内容遵循 Apache-2.0（与上游组件一致）。引用的代码片段版权归原作者所有（示例代码标注为 Public Domain / CC0，源自仓库 examples）。
