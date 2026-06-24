# esp-sr-skill

面向 Espressif **ESP-SR** 离线语音识别框架的 AI Skill（Claude Code / Agent skill 格式）。

ESP-SR 是乐鑫提供的嵌入式语音识别组件，包含 **AFE（音频前端）**、**WakeNet（唤醒词）**、**MultiNet（命令词）**、**VADNet（语音活动检测）**、**AEC（回声消除）**、**中文 TTS** 等模块，覆盖 ESP32 / ESP32-S3 / ESP32-S31 / ESP32-P4 / ESP32-C3 / ESP32-C5 / ESP32-C6 等芯片。

本 skill 把仓库真实的头文件、文档、`test_apps` 用法整理成场景化 recipe 与 API 参考，让 AI agent 能**不臆造接口**地基于 ESP-SR 开发固件。

## 功能特性

- **场景化 recipe**：AFE 语音识别主线、模型分区配置、V1→V2 迁移、WakeNet/MultiNet/VADNet/AEC/TTS 独立用法、自定义命令词
- **完整 API 参考**：AFE / WakeNet / MultiNet / VADNet / AEC / DOA / TTS 全部真实函数签名、结构体、枚举
- **配置参考**：menuconfig 选项、Kconfig 符号、分区表、阈值范围、afe_config_t 字段
- **踩坑汇总**：20 条来自实战与头文件注释的常见错误及正确写法
- **状态机与数据流**：唤醒/命令词/VAD 状态转换、AFE pipeline 数据流
- **示例索引**：仓库自带 `test_apps/` 全部可编译工程路径

## 安装

将本 skill 目录放入 Claude Code 的 skills 目录即可。两种方式：

### 方式一：项目级（推荐，随工程走）

把 `esp-sr-skill` 目录放到你的 ESP-IDF / esp-skainet 工程下：

```
your_project/
└── .claude/
    └── skills/
        └── esp-sr-skill/      ← 整个目录放这里
```

### 方式二：用户级（全局可用）

放到用户 skills 目录：

- **Windows**: `%USERPROFILE%\.claude\skills\esp-sr-skill\`
- **macOS / Linux**: `~/.claude/skills/esp-sr-skill/`

放入后 Claude Code 会自动加载，当对话涉及 ESP-SR / WakeNet / MultiNet / 唤醒词 / 命令词 等关键词时触发。

## 目录结构

```
esp-sr-skill/
├── SKILL.md                    # 主入口：原则、recipe 索引、踩坑、工作流
├── AGENTS.md                   # 工程约定、include 模式、构建流程、checklist
├── recipes/                    # 场景化方案
│   ├── afe_sr_pipeline.md      # AFE 语音识别主线
│   ├── model_partition.md      # 模型选择与烧录
│   ├── migration_v1_v2.md      # V1→V2 迁移
│   ├── wakenet_standalone.md   # 单独 WakeNet
│   ├── multinet_commands.md    # MultiNet 命令词
│   ├── custom_commands.md      # 自定义命令词
│   ├── vadnet.md               # VADNet
│   ├── aec_usage.md            # AEC 回声消除
│   └── chinese_tts.md          # 中文 TTS
├── resources/                  # 速查文档
│   ├── api_reference.md        # 全部 API 签名
│   ├── config_reference.md     # Kconfig / 分区 / 配置字段
│   ├── pitfalls.md             # 踩坑汇总
│   ├── example_list.md         # test_apps 示例索引
│   └── state_machine.md        # 状态机与数据流
├── README.md                   # 本文件
└── CHANGELOG.md                # 版本记录
```

## 支持范围

- **芯片**：ESP32 / ESP32-S2 / ESP32-S3 / ESP32-S31 / ESP32-P4 / ESP32-C3 / ESP32-C5 / ESP32-C6
- **框架**：ESP-IDF >= 5.0
- **模块**：AFE、WakeNet（wn9/wn9l/wn9s）、MultiNet（mn2/mn5q8/mn6/mn7，中/英）、VADNet、WebRTC VAD、AEC（SR/FD/VOIP）、NS/NSNet、DOA、中文 TTS
- **许可**：ESPRESSIF MIT License（与仓库 `LICENSE` 一致）

## 数据来源声明

本 skill 所有 API、结构体、宏、模型名、配置项均来自 `esp-sr` 仓库的：
- `include/<target>/*.h`、`src/include/*.h`、`esp-tts/esp_tts_chinese/include/*.h`
- `docs/en/*.rst`、`docs/zh_CN/*.rst`
- `test_apps/`（可编译的真实用法）
- `Kconfig.projbuild`、`idf_component.yml`、`CMakeLists.txt`、`README.md`

未在仓库中出现的接口一律不写入。文档偏薄的领域（如 NS 独立 API 细节）以 recipe 数量更少、引用更克制的方式处理，避免臆造。
