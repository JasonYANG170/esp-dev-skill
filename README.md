**[English](README_EN.md)** | 中文

# esp-dev-skill

Espressif 全生态统一 AI 技能。把 **41 个** ESP 框架/SDK 仓库的开发知识（**554 个**场景 recipe + API/配置/陷阱参考）聚合为一个可安装的 skill，所有内容均基于各仓库的真实文档与源码。

支持芯片：ESP32、ESP32-S2/S3、ESP32-C2/C3/C5/C6/C61、ESP32-H2/H4/H21、ESP32-P4、ESP8266。

## 功能特性

- 基于 41 个 Espressif 官方仓库的真实文档与源码
- 覆盖核心框架：ESP-IDF、Arduino-ESP32、ESP8266 RTOS SDK
- 覆盖协议与连接：Matter、Zigbee、Thread、BLE (NimBLE)、ESP-NOW、MQTT、Modbus、USB
- 覆盖 AI/语音/视觉：ESP-SR、ESP-DL、ESP-Skainet、ESP-DSP、ESP-Vision、ESP-Who
- 覆盖领域框架：ADF/GMF 音频、Brookesia、Claw、IoT Solution、RainMaker
- 覆盖安全：mbedTLS、TF-PSA-Crypto、安全认证管理
- 每个子技能含 recipe（完整调用链）、API 速查、配置参考、常见陷阱
- 提供跨仓库 recipe 总索引（554 个场景一表通查）

## 安装说明

### 1. 拉取仓库到 skill 目录

根据你使用的 AI Agent 文档，找到或创建存放 Skill 的目录：

```bash
git clone <repo-url> esp-dev-skill
```

或直接复制目录：

```bash
cp -r esp-dev-skill /path/to/skills/
```

例如：

> **Claude Code**
> **项目作用域**：位于项目根目录下的 `.claude/skills`
> **用户作用域**：位于 `~/.claude/skills`，对本机所有项目生效
> 进入到对应的 skills 文件夹下，在终端执行 `git clone <repo-url> esp-dev-skill` 即可

> **QwenCode**
> **项目作用域**：位于项目根目录下的 `.qwen/skills`
> **用户作用域**：位于 `~/.qwen/skills`，对本机所有项目生效

> **OpenCode**
> **项目作用域**：位于项目根目录下的 `.opencode/skills`
> **用户作用域**：位于 `~/.config/opencode/skills`，对本机所有项目生效

### 2. 使用指定 skill

在你的 AI Agent 中确认 Skill 已加载，可通过命令指定 skill。

例如：

> **Claude Code**
> 在终端中输入 `claude` 后回车，然后输入 `/esp-dev-skill` 并描述你的需求

> **QwenCode**
> 在终端中输入 `qwen` 后回车，输入 `/skills` 回车，选择 esp-dev-skill

> **OpenCode**
> 在终端中输入 `opencode` 后回车，输入 `/skills` 回车，选择 esp-dev-skill

## 工作原理

Skill 定义了一套工作流，AI Agent 在生成代码时会遵循：

| 步骤 | 名称 | 说明 |
|------|------|------|
| 1 | 定位 | 确定所需框架/SDK/功能，查阅路由表 |
| 2 | 进入子技能 | 打开 `repos/<repo>/SKILL.md`，读取核心原则与 recipe 索引 |
| 3 | 匹配 Recipe | 在 `repos/<repo>/recipes/` 中找到对应场景，获取完整调用链 |
| 4 | 查询 | 查阅 `repos/<repo>/resources/` 中的 API/配置/陷阱文档 |
| 5 | 验证 | 确认 include、初始化顺序、config 宏、引脚/芯片支持 |
| 6 | 确认 | 向用户展示实现方案（头文件、初始化序列、主循环、构建命令） |
| 7 | 执行 | 生成代码，遵循标准项目结构 |
| 8 | 构建烧录 | ESP-IDF: `idf.py build flash monitor` / Arduino: upload / AT: 固件打包 |

## 跨仓库通用原则

- **先选对层级**：ESP-IDF（底层控制）/ Arduino-ESP32（快速原型）/ ESP8266 RTOS SDK（ESP8266）
- **版本敏感**：ESP-IDF 跨大版本 API 会变，注意目标版本
- **组件可复用**：优先从 ESP Component Registry 获取组件
- **拒绝编造**：所有内容基于真实仓库文档，不含臆测
- **协议栈有依赖**：BLE/Zigbee/Thread 需对应芯片能力（经典蓝牙仅 ESP32，BLE Mesh/Thread/Zigbee 需 C/H 系列）
- **芯片能力差异**：不同芯片支持的外设、协议、内存不同，务必确认

## 覆盖仓库（41 个）

### 核心框架（3）
`esp-idf` · `arduino-esp32` · `ESP8266_RTOS_SDK`

### 领域框架（12）
`esp-adf` · `esp-gmf` · `esp-wdf` · `esp-wasmachine` · `esp-brookesia` · `esp-iot-solution` · `esp-vision` · `esp-who` · `esp-insights` · `esp-claw` · `esp-amp` · `esp-lowcode-matter` · `esp-agents-firmware`

### 协议与连接（13）
`esp-matter` · `esp-zigbee-sdk` · `esp-thread-br` · `connectedhomeip` · `esp-now` · `esp-hosted-mcu` · `esp-nimble` · `esp-mqtt` · `esp-modbus` · `esp-usb` · `tinyusb` · `esp-at` · `esp-protocols`

### AI/语音/视觉/DSP（7）
`esp-skainet` · `esp-sr` · `esp-dl` · `esp-dsp` · `esp-detection` · `esp-video-components` · `esp-rainmaker`

### 安全/加密（3）
`mbedtls` · `TF-PSA-Crypto` · `esp_secure_cert_mgr`

### 其他组件库（2）
`esp-bist` · `esp-serial-flasher`

## 参考资源

- 跨仓库 Recipe 总索引 → `resources/recipe_index.md`（554 个场景一表通查）
- 仓库索引 → `resources/repo_index.md`
- 术语表 → `resources/glossary.md`
- 各子技能文档 → `repos/<repo>/SKILL.md`
- Espressif 官网 → https://espressif.com

## 许可

内容遵循各上游仓库许可（多为 Apache-2.0）。本聚合技能元数据 Apache-2.0。
