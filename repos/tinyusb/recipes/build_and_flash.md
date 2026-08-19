# 编译与烧录

> **适用摘要**: 用 CMake（首选）或 Make 构建 TinyUSB 示例/项目，包括拉取 MCU 依赖、选 `BOARD`、生成固件、jlink/openocd/uf2 烧录。

> Version: selected repo/component version; align with the user project and dependency manifest.
> Validation: draft metadata added from repository routing; verify APIs, Kconfig symbols, and component dependencies against the selected version.

## 触发意图

- "TinyUSB 怎么编译"
- "BOARD 怎么选"
- "make / cmake 构建示例"
- "get_deps.py 拉依赖"
- "jlink / openocd / uf2 烧录"

## 前置条件

| 条件 | 要求 |
|---|---|
| 参考文档 | `docs/getting_started.rst`、仓库根 `CLAUDE.md` |
| 目标板 | `hw/bsp/<FAMILY>/boards/<BOARD>/` 必须存在 |

## 分步说明

### 1. 拉取 MCU 依赖（首次或换家族时）

```bash
# 按板拉（自动解析家族）
python tools/get_deps.py -b stm32h743eval

# 按家族拉
python tools/get_deps.py rp2040
python tools/get_deps.py espressif     # ESP32-S2/S3 等

# 在示例目录里用 make 拉（Make 流程）
cd examples/device/cdc_msc && make BOARD=stm32h743eval get-deps
```

### 2. CMake + Ninja（首选）

```bash
cd examples/device/cdc_msc
cmake -G Ninja -DBOARD=raspberry_pi_pico -B build
ninja -C build

# 调试构建 / 日志
cmake -G Ninja -DBOARD=raspberry_pi_pico -DCMAKE_BUILD_TYPE=Debug -B build
cmake -G Ninja -DBOARD=raspberry_pi_pico -DLOG=2 -B build
cmake -G Ninja -DBOARD=raspberry_pi_pico -DLOG=2 -DLOGGER=rtt -B build
```

### 3. CMake 烧录 target

```bash
ninja -C build cdc_msc-jlink        # JLink
ninja -C build cdc_msc-openocd      # OpenOCD
ninja -C build cdc_msc-uf2          # 生成 UF2（RP2040 等）
ninja -C build -t targets           # 列出所有 target
```

### 4. Make（备选；部分家族如 espressif/rp2040 仅支持 CMake）

```bash
cd examples/device/cdc_msc
make BOARD=raspberry_pi_pico all
make BOARD=raspberry_pi_pico DEBUG=1 all
make BOARD=raspberry_pi_pico LOG=2 LOGGER=rtt all
make BOARD=raspberry_pi_pico flash-jlink      # 或 flash-stlink / flash-openocd
make BOARD=raspberry_pi_pico all uf2          # 生成 UF2
```

### 5. 选择 RootHub 端口 / 速度

```bash
# Make
make BOARD=<board> RHPORT_DEVICE=1 all
make BOARD=<board> RHPORT_DEVICE_SPEED=OPT_MODE_FULL_SPEED all

# CMake
cmake -DBOARD=<board> -DRHPORT_DEVICE=1 -B build
cmake -DBOARD=<board> -DRHPORT_DEVICE_SPEED=OPT_MODE_FULL_SPEED -B build
```

### 6. 验证枚举

- 主机端：`lsusb`（Linux）、设备管理器（Windows）、系统信息（macOS）
- 日志：`LOG=2` 构建后通过配置的 LOGGER（uart/rtt）查看栈日志

## 常见错误

| 错误 | 原因 | 解决方法 |
|---|---|---|
| 找不到 `hw/mcu/<vendor>` | 未拉依赖 | 运行 `python tools/get_deps.py <family>` |
| espressif/rp2040 用 make 报错 | 这两家族仅支持 CMake | 改用 `cmake -DBOARD=...` |
| BOARD 名拼错 | BSP 路径不存在 | 在 `hw/bsp/<family>/boards/` 查正确 BOARD 名 |
| Ninja 未安装 | 缺构建工具 | 装 Ninja，或去 `-G Ninja` 用默认生成器 |
| 烧录 target 名不对 | target 含项目名前缀 | 用 `ninja -t targets` / `make help` 查实际 target |

## 参考

- `docs/getting_started.rst` — 官方快速开始
- `CLAUDE.md`（仓库根） — 构建命令速查
- `hw/bsp/` — 各家族板级支持
- `docs/reference/boards.rst` — 支持板卡列表
