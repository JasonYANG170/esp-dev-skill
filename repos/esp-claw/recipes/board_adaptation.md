# 适配新开发板

> **适用摘要**: 为 ESP-Claw `edge_agent` 新增一块开发板：编写 board YAML、`setup_device.c`、板级 sdkconfig 默认与可选 FATFS overlay，并用 `idf.py bmgr` 选中构建。

> Evidence: `repos/esp-claw/resources/`, source/examples in `repos/esp-claw/`, and this recipe path `repos/esp-claw/recipes/board_adaptation.md`.
> Validation: draft metadata added from repository routing; verify APIs, Kconfig symbols, and component dependencies against the selected version.

## 触发意图

- "适配新板子"
- "add a new board to esp-claw"
- "Board Manager 加开发板"
- "board_info.yaml / board_devices.yaml"

## 前置条件

| 条件 | 要求 |
|---|---|
| 已能构建 | 至少跑通过 `recipes/build_and_flash.md` |
| 参考 | `application/edge_agent/boards/espressif/esp32_S3_DevKitC_1/`、`boards/espressif/esp_sparkbot/` |
| 文档 | ESP Board Manager「create-board」指南（见 build-from-source.mdx 提示） |

## 分步说明

### 1. 建板子目录

```
application/edge_agent/boards/<vendor>/<board>/
├── board_info.yaml
├── board_devices.yaml
├── board_peripherals.yaml
├── sdkconfig.defaults.board
├── setup_device.c
├── components/            # 可选：板级本地组件
└── fatfs_image/           # 可选：构建期 overlay 到 SYSTEM 镜像（不进 DATA）
```

vendor 目录取现有之一（`espressif` / `m5stack` / `lilygo` / `waveshare` / `dfrobot` / `movecall` / `rockbase-iot` / `Nologo.Tech` / `community`）或新建。

### 2. board_info.yaml：板子元信息

声明芯片、名称、显示/触控/摄像头等板级能力标志（这些会驱动 `ESP_BOARD_DEV_*` Kconfig，进而决定 `APP_CLAW_LUA_MODULE_*` 默认值）。

### 3. board_devices.yaml：设备清单（占用 IO）

列出板上设备（`led_strip` / `gpio_led` / `flashlight` / `display` / `touch` / `camera` …）及其占用的 IO。`board_hardware_info` Skill 会读它，`light_switch` 等 Skill 据此选 `io` / `led_count` / `active_level`。

### 4. board_peripherals.yaml：外设总线

I2C / SPI / I2S 等总线号与引脚，供 `board_manager` 装配设备句柄。

### 5. setup_device.c：板级初始化

实现 Board Manager 调用的 setup 入口，做 GPIO/背光/电源等板级初始化。

### 6. sdkconfig.defaults.board：板级默认

```ini
# 例：覆盖分区表、PSRAM、控制台通道等
CONFIG_PARTITION_TABLE_CUSTOM=y
CONFIG_PARTITION_TABLE_CUSTOM_FILENAME="partitions_16MB.csv"
```
> 应用层 `sdkconfig.defaults` 已含通用默认；板级文件只覆盖差异。

### 7. （可选）fatfs_image/ overlay

板子自带字体/图片/Skill 可放 `fatfs_image/`，构建期 overlay 进 **SYSTEM** 镜像（只读）。注意：板子 overlay **不**写 DATA；隐藏目录不被采纳。

### 8. 选中并构建

```bash
cd application/edge_agent
idf.py bmgr -c ./boards -b <board>
idf.py build
idf.py flash monitor
```

## 常见错误

| 错误 | 原因 | 解决方法 |
|---|---|---|
| `bmgr -l` 看不到新板 | 目录/YAML 命名不规范 | 确认在 `boards/<vendor>/<board>/` 且 `board_info.yaml` 齐全 |
| Lua 模块（如 `lcd_touch`）没启用 | 板子未声明对应 `ESP_BOARD_DEV_*` 能力 | 在 board_info 声明显示/触控支持，对应 `APP_CLAW_LUA_MODULE_*` 才默认 y |
| 板子 overlay 没进 DATA | overlay 只进 SYSTEM | 用户可写文件走 DATA（`/fatfs`），别把用户文件放板子 overlay |
| PSRAM/分区报错 | 板级 sdkconfig 未覆盖 | 在 `sdkconfig.defaults.board` 设对 PSRAM 与分区表 |

## 参考

- `application/edge_agent/boards/espressif/esp32_S3_DevKitC_1/`
- `application/edge_agent/boards/espressif/esp_sparkbot/`（含 README、board_devices.yaml、setup_device.c）
- `application/edge_agent/boards/lilygo/lilygo_t_display_p4_v1/README.md`
- `docs/src/content/docs/en/reference-project/build-from-source.mdx`
- `.agents/design.md`（文件系统分层 / 板子 overlay 规则）
