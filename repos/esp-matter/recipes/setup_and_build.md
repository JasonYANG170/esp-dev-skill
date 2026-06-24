# 环境搭建与首次构建

> **适用摘要**: 克隆 esp-matter 及 connectedhomeip 子模块，配置 ESP-IDF，设置目标芯片，构建并烧录 `light` 示例，确认设备能启动并打印 commissioning 广播日志。

## 触发意图

- "搭建 esp-matter 环境"
- "怎么编译 esp-matter 示例"
- "light 例程怎么烧录"
- "esp-matter clone"
- "首次烧录 Matter 设备"

## 前置条件

| 条件 | 要求 |
|---|---|
| ESP-IDF | v5.5.4（esp32c5/esp32c61 也用 v5.5.4） |
| 主机 | Ubuntu 20.04/22.04/24.04 或 macOS 10.15+；Windows 使用 WSL2 + usbipd-win |
| 仓库 | `examples/light/` |
| 工具 | `install.sh` 会构建 host 工具（chip-tool / chip-cert） |

## 分步说明

### 1. 安装 ESP-IDF（参考 `developing.rst`）

```bash
git clone --recursive https://github.com/espressif/esp-idf.git
cd esp-idf
git checkout v5.5.4
git submodule update --init --recursive
./install.sh
cd ..
```

### 2. 浅克隆 esp-matter（节省时间）

```bash
cd esp-idf && source ./export.sh && cd ..

git clone --depth 1 https://github.com/espressif/esp-matter.git
cd esp-matter
git submodule update --init --depth 1
cd ./connectedhomeip/connectedhomeip
./scripts/checkout_submodules.py --platform esp32 linux --shallow   # macOS 用 darwin
cd ../..
./install.sh                  # 不需要 host 工具时加 --no-host-tool
cd ..
```

### 3. 每次开终端都要 source

```bash
cd esp-idf && source ./export.sh && cd ..
cd esp-matter && source ./export.sh && cd ..
export IDF_CCACHE_ENABLE=1     # 强烈建议，Matter 构建慢
```

### 4. 设置目标芯片并构建 light 示例

```bash
cd esp-matter/examples/light
idf.py set-target esp32c3      # 可选: esp32 / esp32s3 / esp32c2 / esp32c6 / esp32h2 / esp32c5 / esp32c61 / esp32p4
idf.py build
```

> ESP32-P4 没有原生 Wi-Fi/BLE，需要 `esp_hosted` 从机（参见 `developing.rst` 的 ESP32-P4 段落）。

### 5. 首次烧录要擦除 flash

```bash
idf.py erase_flash flash monitor
```

设备启动后会打印 commissioning 二维码 / manual code，例如 `MT:Y.K9042C00KA0648G00`，对应默认 `passcode=20202021`、`discriminator=3840`（来自 `CONFIG_ENABLE_TEST_SETUP_PARAMS`）。

### 6. （可选）切换 esp32c6 的 Wi-Fi / Thread

在 `menuconfig` 里三选一同步切换：

| 模式 | `CONFIG_OPENTHREAD_ENABLED` | `CONFIG_ENABLE_WIFI_STATION` | `CONFIG_USE_MINIMAL_MDNS` |
|---|---|---|---|
| Thread | y | n | n |
| Wi-Fi | n | y | y |

## 常见错误

| 错误 | 原因 | 解决方法 |
|---|---|---|
| `dial tcp ... connect: connection refused` | 克隆子模块时网络受限 | 使用 VPN，或 `checkout_submodules.py --shallow` 重试 |
| `This script was called from a virtual environment` | 在 venv 里再次创建 venv | `pip install -r $IDF_PATH/requirements.txt` |
| 启动后无 commissioning 日志 | NVS 残留旧 fabric | `idf.py erase_flash flash monitor` |
| 构建极慢 | 未启用 ccache | `export IDF_CCACHE_ENABLE=1` |
| Windows 下 COM 口不可见 | 未用 WSL + usbipd-win | 安装 usbipd-win 并 `usbipd bind`/`attach` |

## 参考

- `D:/esp-skill/espressif-repos/esp-matter/docs/en/developing.rst` — Getting the Repository / Building Applications / Flashing the Firmware
- `D:/esp-skill/espressif-repos/esp-matter/examples/light/README.md`
- `D:/esp-skill/espressif-repos/esp-matter/install.sh`、`export.sh`
- `D:/esp-skill/espressif-repos/esp-matter/idf_component.yml`（声明依赖的 IDF 版本）
