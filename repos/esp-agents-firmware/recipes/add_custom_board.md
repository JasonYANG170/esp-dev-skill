# 添加自定义板级配置

> **适用摘要**: 在 `examples/common/boards/` 下新增一块自定义板的配置，使其能被 `idf.py select-board` 识别并参与构建。

## 触发意图

- "添加自定义板"
- "适配新硬件板"
- "select-board 我的板子"
- "板级配置 board_defs.h"
- "ESP Board Manager 复用"

## 前置条件

| 条件 | 要求 |
|---|---|
| 参考目录 | `examples/common/boards/esp_box_3/`、`esp_vocat_board_v1_2/` |
| 板的硬件 | ESP32-S3 系，音频 codec / LCD / 触摸（可选） |

## 分步说明

### 1. 新建板目录

按 `docs/board_customisation.md` 的规定，自定义板必须是 `examples/common/boards/` 的子目录：

```bash
mkdir -p examples/common/boards/my_custom_board
```

### 2. 添加板配置文件

`board_defs.h` — 定义板级宏（参考现有板）：

```c
/* examples/common/boards/my_custom_board/board_defs.h */
#pragma once

/* LCD 是否水平/垂直镜像（参考 esp_box_3） */
#define LCD_MIRROR_X_Y 0

/* 是否支持 LEDC 背光 */
#define LEDC_BACKLIGHT_SUPPORTED 1

/* 是否支持电容触摸及其 GPIO（参考 esp_vocat_board_v1_2） */
#define CAPACITIVE_TOUCH_SUPPORTED 1
#define CAPACITIVE_TOUCH_CHANNEL_GPIO 7

/* 指示灯设备名（可选，参考 esp_vocat_board_v1_2） */
#define INDICATOR_DEVICE_NAME "led_green"

/* 设备手册 URL，会显示在 RainMaker Home App */
#define BOARD_DEVICE_MANUAL_URL "https://example.com/my_board_manual.md"
```

`sdkconfig.defaults` — 覆盖 ESP-IDF menuconfig 默认值（按板硬件填，如 PSRAM 大小、LCD 驱动等）。

### 3a. 若板已在 ESP Board Manager 中

> ESP Board Manager 位于 <https://github.com/espressif/esp-gmf/tree/main/packages/esp_board_manager/boards>

只需在板目录加一个空文件 `.use_from_esp_board_manager`，跳过下一步：

```bash
touch examples/common/boards/my_custom_board/.use_from_esp_board_manager
```

### 3b. 若板不在 ESP Board Manager 中

额外添加：
- `board_devices.yaml` — 设备配置
- `board_peripherals.yaml` — 外设配置

> 注意：此种情况下**不要**添加 `.use_from_esp_board_manager` 文件。

### 4. 选择并构建

```bash
cd examples/voice_chat          # 或 matter_controller
idf.py select-board --board my_custom_board
idf.py build flash monitor
```

## board_defs.h 现有板参考

| 宏 | esp_box_3 | esp_vocat_board_v1_2 | 含义 |
|---|---|---|---|
| `LCD_MIRROR_X_Y` | 1 | 未定义 | LCD 双向镜像 |
| `LEDC_BACKLIGHT_SUPPORTED` | 1 | 1 | 支持 LEDC 背光 |
| `CAPACITIVE_TOUCH_SUPPORTED` | 未定义 | 1 | 支持电容触摸 |
| `CAPACITIVE_TOUCH_CHANNEL_GPIO` | 未定义 | 7 | 电容触摸通道 GPIO |
| `INDICATOR_DEVICE_NAME` | 未定义 | `"led_green"` | 监听状态指示 LED 设备名 |
| `BOARD_DEVICE_MANUAL_URL` | 已定义 | 已定义 | RainMaker App 内显示的手册 URL |

## 常见错误

| 错误 | 原因 | 解决方法 |
|---|---|---|
| `select-board` 看不到新板 | 目录不在 `examples/common/boards/` 下 | 必须是该目录的直接子目录 |
| 外设行为异常 | `board_defs.h` 缺所需宏 | 参考 `recipes/` 与 app 源码确认依赖哪些宏，补齐 |
| 同时加了 yaml 与 `.use_from_esp_board_manager` | 两者互斥 | 二选一：在 Board Manager 就用标记文件，否则用 yaml |
| `BOARD_DEVICE_MANUAL_URL` 不可访问 | RainMaker App 打不开手册 | 填可公开访问的 URL |

## 参考

- `docs/board_customisation.md` — 官方自定义板步骤
- `examples/common/boards/esp_box_3/board_defs.h` — ESP-BOX-3 板级宏
- `examples/common/boards/esp_vocat_board_v1_2/board_defs.h` — ESP-VoCat 板级宏
- `examples/common/boards/README.md` — 板目录说明
