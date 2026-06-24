# esp-vision-skill

面向 ESP-VISION（乐鑫低代码端侧 AI 与计算机视觉框架）的 AI Skill。本 Skill 让 AI agent 在 ESP32-P4 / ESP32-S3 / ESP32-S31 开发板上**正确**地编写 MicroPython 视觉脚本、运行 ESP-DL 模型推理、做图像处理与编解码推流，并客制化 ESP-VISION 固件。所有 API、类、常量、配置宏与文件路径均来自 ESP-VISION 仓库的真实文档（`docs/`）、类型存根（`stubs/*.pyi`）、板级配置（`boards/`）与示例（`example/`）—— 不存在则不写，绝不臆造。

## 功能特性

- **场景化配方（recipes/）**：摄像头采集、图像处理（滤波/二值化/色块追踪/形态学）、二维码/条码/AprilTag、ESP-DL 目标检测/姿态估计/分类、LCD 显示、H.264 录制、RTSP 推流、asyncio 多协程应用、客制化固件。
- **API 速查（resources/api_reference.md）**：按模块分组的真实函数/类签名，来源于 `stubs/*.pyi`。
- **配置参考（resources/config_reference.md）**：构建变量、板级 `boardconfig.h` 宏、`imlib_config.h` 算法开关、`board.cmake` 开关、`mpconfigboard.h` 功能宏、`manifest.py`。
- **避坑汇总（resources/pitfalls.md）**：16 条高频错误与正确写法。
- **示例索引（resources/example_list.md）**：`example/` 下全部真实脚本的一行说明。

## 安装

将本 Skill 克隆/复制到 Claude Code 的 skills 目录即可被识别：

```bash
# 项目级（随项目分发）
git clone <本仓库> .claude/skills/esp-vision-skill

# 用户级（全局可用）
# Windows:  %USERPROFILE%\.claude\skills\esp-vision-skill
# macOS/Linux: ~/.claude/skills/esp-vision-skill
```

或直接把 `esp-vision-skill/` 目录放到 `.claude/skills/` 下。Skill 通过 `SKILL.md` 的 frontmatter（`name`/`description`/`tags`/触发词）被自动匹配。

## 支持范围

| 维度 | 范围 |
|---|---|
| 框架 | ESP-VISION（MicroPython v1.28.0 视觉运行时） |
| 芯片 | ESP32-P4、ESP32-S3、ESP32-S31 |
| 开发板 | `ESP32_P4X_EYE`、`ESP32_P4X_FUNCTION_EV_BOARD`、`ESP32_S3_EYE`、`ESP32_S31_KORVO`、`TEMPLATE` |
| ESP-IDF | `release/v5.5`、`release/v6.0`、`master` |
| 构建入口 | `idf.py --board <BOARD> ...`（`idf_ext.py`）或顶层 `Makefile` |

## 目录结构

```
esp-vision-skill/
├── SKILL.md              # 入口：原则 / 何时用 / 配方索引 / 避坑 / 工作流
├── AGENTS.md             # 补充工程约定与工具链（不与 SKILL.md 重复）
├── recipes/              # 场景配方（12 个）
├── resources/            # API / 配置 / 避坑 / 示例索引 速查
├── README.md             # 本文件
└── CHANGELOG.md
```

## 上游权威参考

- ESP-VISION 中文文档：https://docs.espressif.com/projects/esp-vision/zh_CN/latest/
- ESP-VISION 英文文档：https://docs.espressif.com/projects/esp-vision/en/latest/
- MicroPython v1.28.0：https://docs.micropython.org/en/v1.28.0/
- Web IDE：https://esp-vision-ide.espressif.tools/

## 许可证

本 Skill 按其自身许可证分发（见仓库 LICENSE）。引用的 ESP-VISION 仓库代码/文档遵循各自许可证（仓库原始代码为 Apache-2.0，`components/imlib` 为 MIT，子模块各自保留许可证）。
