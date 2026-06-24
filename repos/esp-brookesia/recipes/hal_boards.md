# HAL Boards 选板与设备初始化

> **适用摘要**: 通过 `brookesia_hal_boards` 选择开发板、用 `brookesia_hal_adaptor` 初始化 HAL 设备，并按接口名查询硬件能力（音频/显示/触摸/存储/电源/背光）。依赖外设的服务（Audio/Device/Agent）必须先完成 HAL 初始化。

## 触发意图

- "怎么选开发板 / gen-bmgr-config"
- "初始化 HAL 设备"
- "获取显示/音频/存储接口"
- "新增自定义板"
- "board_devices.yaml / board_peripherals.yaml"

## 前置条件

| 条件 | 要求 |
|---|---|
| 依赖 | `espressif/brookesia_hal_boards`、`espressif/brookesia_hal_adaptor`、`espressif/brookesia_hal_interface`（通常由板级工程 `idf_ext.py` 自动拉取）|
| 参考文档 | `docs/en/hal/boards/index.rst`、`docs/en/hal/adaptor.rst`、`docs/en/hal/interface/index.rst` |
| 参考示例 | `examples/service/device`、`examples/agent/chatbot` |

## 分步说明

### 1. 选择板（板级工程）

```bash
idf.py gen-bmgr-config -b esp_vocat_board_v1_2
```

板名见 SKILL.md 板支持表（Espressif 8 块 + Waveshare 3 块 + rymcu 1 块）。`gen-bmgr-config` 会从工程本地 `boards/` 优先查找，再查组件内 `boards/`。

### 2. 板配置目录结构（来自 `docs/en/hal/boards/index.rst`）

```
boards/<vendor>/<board>/
├── board_info.yaml          # 板元数据（名、芯片、版本、厂商）
├── board_devices.yaml       # 逻辑设备（audio_codec/display_lcd/lcd_touch/ledc_ctrl/fs_*/camera/power_ctrl/gpio_ctrl/gpio_expander/custom）
├── board_peripherals.yaml   # 底层外设（I2C/I2S/SPI/LEDC/GPIO 引脚与参数）
├── sdkconfig.defaults.board # 板相关 Kconfig 默认（Flash/PSRAM/CPU 频率/录音格式等）
├── setup_device.c           # 可选：自定义驱动初始化工厂回调
└── components/...           # 可选：非标准外设的特殊组件依赖
```

设备类型（`board_devices.yaml`）：`audio_codec`、`display_lcd`、`lcd_touch`、`ledc_ctrl`、`fs_fat`/`fs_spiffs`、`camera`、`power_ctrl`、`gpio_ctrl`、`gpio_expander`、`custom`。

### 3. 初始化 HAL 设备

```cpp
#include "brookesia/hal_interface.hpp"
#include "brookesia/hal_adaptor.hpp"
using namespace esp_brookesia;

// 按需初始化（推荐：用到的才初始化）
hal::init_device(hal::StorageDevice::DEVICE_NAME);
hal::init_device(hal::AudioDevice::DEVICE_NAME);
hal::init_device(hal::DisplayDevice::DEVICE_NAME);

// 或一次性初始化全部（device/chatbot 示例做法）
hal::init_all_devices();

// 退出时
hal::deinit_all_devices();
```

> 多核 SPI LCD：Display 初始化须用 `BROOKESIA_THREAD_CONFIG_GUARD({.core_id=1})` 锁核（见 SKILL.md 踩坑 #10）。

### 4. 按接口名查询 HAL 能力

```cpp
// 取第一个某类型接口及其名字
auto [name, panel_iface] = hal::get_first_interface<hal::DisplayPanelIface>();
auto [sname, fs_iface]   = hal::get_first_interface<hal::StorageFsIface>();
auto [pname, player]     = hal::get_first_interface<hal::AudioCodecPlayerIface>();

// 接口静态名常量（用于 capabilities 比对）
hal::BoardInfoIface::NAME
hal::DisplayPanelIface::NAME
hal::DisplayBacklightIface::NAME
hal::DisplayTouchIface::NAME
hal::AudioCodecPlayerIface::NAME
hal::AudioCodecRecorderIface::NAME
hal::StorageFsIface::NAME
hal::PowerBatteryIface::NAME
```

### 5. 直接操作 HAL 接口（device 示例：绘制色条 / 写 PCM）

```cpp
// 显示面板绘制
const auto &info = panel_iface->get_info();   // h_res/v_res/pixel_format/pixel_bits
panel_iface->draw_bitmap(0, y, info.h_res, y + lines, buf);

// 音频播放器
hal::AudioCodecPlayerIface::Config cfg{ .bits=16, .channels=1, .sample_rate=16000 };
player->open(cfg);
player->write_data(pcm_data, pcm_size);
player->close();

// 存储文件系统信息
auto infos = fs_iface->get_all_info();        // vector<Info>，含 fs_type/mount_point
for (const auto &i : infos)
    if (i.fs_type == hal::StorageFsIface::FileSystemType::LittleFS) { /* ... */ }
```

### 6. 新增自定义板

按 `docs/en/hal/boards/index.rst` “Add a Custom Board”：

1. 在 `boards/<vendor>/` 下建子目录
2. 填 `board_info.yaml`（名、芯片、版本、描述）
3. 写 `board_peripherals.yaml`（按实际引脚/总线）
4. 写 `board_devices.yaml`（设备类型与配置）
5. 加 `sdkconfig.defaults.board`（板相关 Kconfig）
6. 可选 `setup_device.c`（自定义驱动初始化回调）
7. 可选 `components/`（非标准外设特殊依赖）

完成后 `idf.py gen-bmgr-config -b <new_board>`。

## 常见错误

| 错误 | 原因 | 解决方法 |
|---|---|---|
| 板级工程缺组件 | 用了 `set-target` | 改用 `gen-bmgr-config -b <board>` |
| HAL 接口取不到 | 设备未初始化 | 先 `hal::init_device(...)` 或 `init_all_devices()` |
| LCD 崩溃 | init/draw 跨核 | `BROOKESIA_THREAD_CONFIG_GUARD` 锁同核 |
| 某外设不识别 | YAML 设备类型/外设引脚错 | 核对 `board_devices.yaml` 与 `board_peripherals.yaml` |
| 自定义驱动需特殊序列 | 标准配置覆盖不了 | 在 `setup_device.c` 写工厂回调 |

## 参考

- `docs/en/hal/boards/index.rst` — 板支持、目录结构、新增板流程
- `docs/en/hal/boards/espressif.rst` / `waveshare.rst` — 各板参数与支持接口
- `docs/en/hal/adaptor.rst` — HAL Adaptor（`hal::init_*`）
- `docs/en/hal/interface/index.rst` — HAL Interface（各 Iface 类）
- `examples/service/device/main/main.cpp` — `init_all_devices` + 接口查询范例
