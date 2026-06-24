# esp-adf-skill

面向 [ESP-ADF](https://github.com/espressif/esp-adf)（Espressif Audio Development Framework）的 AI 技能（Claude Code / Agent Skill 格式）。帮助 AI agent 正确地基于 ESP-ADF 进行音频/多媒体固件开发——播放、录音、流媒体、编解码、音频处理、蓝牙、外设、OTA、CLI 等，所有 API 与配置均取自仓库真实文档与源码，杜绝臆造。

适用芯片：ESP32 / ESP32-S2 / ESP32-S3 / ESP32-C3 / ESP32-C5 / ESP32-C6 / ESP32-P4。

## 特性

- **场景配方（recipes/）**：10 个高频场景的逐步实现，含可复制代码与常见错误表
- **API 速查（resources/api_reference.md）**：pipeline / element / stream / codec / peripherals / board / service 真实签名
- **配置参考（resources/config_reference.md）**：`CONFIG_*_BOARD` 板选、IDF 版本矩阵、关键常量
- **陷阱合集（resources/pitfalls.md）**：生命周期、采样率、事件、构建等高频坑
- **示例索引（resources/example_list.md）**：仓库 `examples/` 全部真实路径与一句话说明

## 覆盖范围

- Audio Pipeline / Element / Event 接口
- Stream：i2s / http / fatfs / spiffs / raw / tcp / tone / embed_flash / tts / pwm / algorithm
- Codec：MP3 / AAC / FLAC / OGG / OPUS / AMR(NB/WB) / WAV 解码与编码
- 音频处理：EQ / Downmix / Sonic / Resample / ALC
- 蓝牙：A2DP Sink/Source、HFP
- 外设与服务：Wi-Fi / SD / 触摸 / 按键 / OTA / CLI
- 板级支持：LyraT、LyraTD-MSC、LyraT-Mini、Korvo 系列、S3-BOX、Kaluga、C3-Lyra、C6-DEVKIT、P4-FUNCTION-EV 等

## 安装

### 方式一：项目级技能（推荐）

将本目录放到工程的 `.claude/skills/` 下：

```bash
mkdir -p .claude/skills
cp -r esp-adf-skill .claude/skills/
```

### 方式二：用户级技能（全局）

放到用户技能目录（如 `~/.claude/skills/`）：

```bash
cp -r esp-adf-skill ~/.claude/skills/
```

安装后，Claude Code 在处理 ESP-ADF 相关请求时会自动加载 `SKILL.md`，并按需读取 `recipes/` 与 `resources/`。

## 使用

直接描述需求即可，例如：
- “用 ESP-ADF 从 Flash 播放一段 MP3”
- “做一个网络收音机（HTTP MP3）”
- “录音存 WAV 到 SD 卡”
- “加个均衡器”
- “蓝牙音箱”

技能会先查阅对应 recipe，再按需引用 API 与配置参考。

## 文件结构

```
esp-adf-skill/
├── SKILL.md                 # 入口：原则 / 何时用 / 配方索引 / 状态机 / 陷阱 / 工作流
├── AGENTS.md                # 工程约定：环境、命名、include、构建、checklist
├── recipes/                 # 10 个场景配方
├── resources/               # api_reference / config_reference / pitfalls / example_list
├── README.md
└── CHANGELOG.md
```

## 许可与兼容

- 技能许可：随仓库（ESPRESSIF MIT）
- 需要：ESP-IDF release/v5.1 ~ v5.5 + ESP-ADF master；CMake 构建（`idf.py`）

## 说明

本技能的内容全部基于 ESP-ADF 仓库的真实文档（`docs/`）、头文件（`components/*/include`）与示例（`examples/`）。编解码器（mp3/aac 等）与语音算法（esp-sr）以预编译子模块形式提供，使用前请确保 `git clone --recursive` 完整拉取 `components/esp-adf-libs` 与 `components/esp-sr`。
