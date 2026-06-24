# esp-skainet-skill

> 面向 AI 编码助手（Claude Code / Agent Skill）的 ESP-Skainet 智能语音开发技能包。
> 本仓库本身不含任何乐鑫源码，所有 API/配置/代码均从 `esp-skainet` 与 `esp-sr` 真实仓库提取，作为参考依据。

## 这是什么

`esp-skainet-skill` 是一个 Claude Code / Agent Skill，帮助 AI 在 ESP32 / ESP32-S3 / ESP32-P4 上正确开发基于 **ESP-Skainet**（乐鑫智能语音助手）的固件。覆盖：

- **WakeNet** 唤醒词引擎（Hi Lexin / Hi ESP / Alexa / 小爱同学 …）
- **MultiNet** 离线命令词识别（中文 / 英文，最多 200 条）
- **Audio Front-End (AFE)**：AEC 回声消除、BSS 多麦盲源分离、NS 降噪、VAD、AGC
- **VAD** 语音活动检测（WebRTC / vadnet1）
- **DOA** 双麦方向角估计
- **中文 TTS** 语音合成

## 特性

- **场景化 recipes**：10 个真实场景（实时唤醒 / 离线推理 / 中英文命令 / 命令自定义 / AFE 调优 / 深度降噪 / 语音通信 / VAD / DOA / TTS），每个含分步说明、可复制代码、常见错误表。
- **真实 API 速查**：`resources/api_reference.md` 收录 `esp_afe_sr_iface_t`、`esp_wn_iface_t`、`esp_mn_iface_t`、`afe_config_t`、`esp_mn_commands_*` 等全部签名，取自 `esp-sr/include/esp32s3/` 头文件。
- **配置项速查**：`resources/config_reference.md` 列出全部 Kconfig 板子/模型/降噪/VAD 符号与 sdkconfig.defaults 模板。
- **陷阱汇总**：26 条高频坑（模型分区、feed 帧大小、fetch 判空、命令 update、PSRAM、TTS mmap …）。
- **零臆造**：找不到的字段/API 一律不写。

## 安装

将该目录放入 Claude Code 的 skills 目录之一：

- 项目级：`<project>/.claude/skills/esp-skainet-skill/`
- 用户级：`~/.claude/skills/esp-skainet-skill/`

或在调用时直接指向本目录的 `SKILL.md`。

## 使用

向 AI 描述你的语音需求，例如：

- "在 ESP32-S3-Korvo-1 上做一个唤醒词 + 中文命令词的 demo"
- "帮我调 AFE 的 VAD，首字总被截断"
- "怎么动态添加英文命令词"

AI 会按 `SKILL.md` 的 Execution Workflow：先匹配 recipe → 核对配置 → 给出基于真实 example 的代码。

## 支持范围

- **芯片**：ESP32、ESP32-S3（推荐）、ESP32-P4、ESP32-S31
- **工具链**：ESP-IDF v4.4 / v5.x（`idf.py`）
- **依赖**：`espressif/esp-sr` (^2.0.0)，由 example 的 `main/idf_component.yml` 声明
- **语言**：C（FreeRTOS）

## 目录结构

```
esp-skainet-skill/
├── SKILL.md                 # 核心规则、状态机、陷阱、recipe 索引、执行流程
├── AGENTS.md                # 工程约定、include 模式、app_main 模板、构建流程、codegen 清单
├── recipes/                 # 10 个场景 recipe
│   ├── wake_word_afe.md
│   ├── wake_word_raw.md
│   ├── cn_speech_commands.md
│   ├── en_speech_commands.md
│   ├── customize_commands.md
│   ├── afe_config_tuning.md
│   ├── deep_noise_suppression.md
│   ├── voice_communication.md
│   ├── voice_activity_detection.md
│   ├── direction_of_arrival.md
│   └── chinese_tts.md
├── resources/
│   ├── api_reference.md     # 真实 API 签名速查
│   ├── config_reference.md  # Kconfig / sdkconfig / partitions 速查
│   ├── pitfalls.md          # 26 条陷阱汇总
│   └── example_list.md      # examples/ 真实工程索引
├── README.md                # 本文件
└── CHANGELOG.md
```

## 许可

参考来源 `esp-skainet` 为 Apache-2.0（部分示例代码为 Public Domain / CC0）。本 skill 文档按 Apache-2.0 提供。
