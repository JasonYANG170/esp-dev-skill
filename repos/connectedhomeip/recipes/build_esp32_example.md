# 构建 ESP32/ESP32-C3 CHIP 示例

> **适用摘要**: 使用 ESP-IDF v4.3 构建、烧录并监视一个 CHIP（Matter）设备示例（以 lock-app / all-clusters-app 为例），覆盖环境准备、目标设置、menuconfig、烧录与日志确认。

## 触发意图

- "编译 ESP32 Matter 示例"
- "烧录 lock-app / all-clusters-app"
- "idf.py build chip"
- "ESP32-C3 构建 connectedhomeip"
- "如何跑起来一个 CHIP 设备"

## 前置条件

| 条件 | 要求 |
|---|---|
| ESP-IDF | v4.3（`git checkout v4.3`） |
| 工具链 | `xtensa-esp32-elf`（ESP32）或 `riscv-esp32-elf`（ESP32-C3） |
| 子模块 | `git submodule update --init --recursive` |
| 参考示例 | `examples/lock-app/esp32/`、`examples/all-clusters-app/esp32/` |
| 硬件 | ESP32-DevKitC / ESP32-WROVER-KIT / M5Stack / ESP32C3-DevKitM |

## 分步说明

### 1. 准备 ESP-IDF 环境

```bash
mkdir -p ${HOME}/tools && cd ${HOME}/tools
git clone https://github.com/espressif/esp-idf.git
cd esp-idf
git checkout v4.3
git submodule update --init
./install.sh
. ./export.sh          # 导出 IDF_PATH 与工具链到 PATH
```

### 2. 激活 CHIP 工具环境

```bash
cd {path-to-connectedhomeip}
source ./scripts/activate.sh     # 已安装则直接激活；首次需先 bootstrap
```

### 3. 进入示例目录并设置目标芯片

```bash
cd examples/lock-app/esp32

# ESP32（Xtensa）
idf.py set-target esp32

# 或 ESP32-C3（RISC-V，需 riscv-esp32-elf）
# idf.py set-target esp32c3
```

### 4. menuconfig 选择 Rendezvous 模式 / 设备类型

```bash
idf.py menuconfig
```

- `Demo -> Rendezvous Mode`：BLE（默认）/ Wi-Fi / Bypass / Thread / Ethernet
- all-clusters-app 还有 `Demo -> Device Type`：`ESP32-DevKitC` / `ESP32-WROVER-KIT_V4.1` / `M5Stack` / `ESP32C3-DevKitM`
- Bypass/Wi-Fi 模式下需在 `Component config -> CHIP Device Layer -> WiFi Station Options` 填 SSID/密码

### 5. 构建并烧录监视

```bash
idf.py build
idf.py -p /dev/ttyUSB0 flash monitor
```

> 提示：连接 USB 后若无法识别，按住开发板 `boot` 键再烧录。退出 monitor：`Ctrl+]`。

### 6. 确认设备已联网（日志）

成功关联 Wi-Fi 后会看到：

```
I (5524) chip[DL]: SYSTEM_EVENT_STA_GOT_IP
I (5524) chip[DL]: IPv4 address changed on WiFi station interface: <IP_ADDRESS>...
```

### 7.（可选）清除已存配网信息

```bash
idf.py -p /dev/ttyUSB0 erase_flash
```

## 常见错误

| 错误 | 原因 | 解决方法 |
|---|---|---|
| `third_party` 找不到文件 | 子模块未初始化 | `git submodule update --init --recursive` |
| `factory` 分区空间不足 | 默认分区表太小 | 使用示例自带 `partitions.csv`（factory 1945K） |
| 链接报 BT/NimBLE 未定义 | sdkconfig 未启用 BLE | 在 `sdkconfig.defaults` 加 `CONFIG_BT_ENABLED=y` + `CONFIG_BT_NIMBLE_ENABLED=y` |
| `set-target` 报工具链缺失 | 未 `source export.sh` 或目标不匹配 | 重新 `. ./export.sh`；ESP32-C3 用 `riscv-esp32-elf` |
| 无法识别串口 | 缺 VCP 驱动 / 端口错 | 安装 SiLabs VCP 驱动；Linux 用 `/dev/ttyUSB0`，macOS 用 `/dev/tty.SLAB_USBtoUART` |
| 关联 5GHz AP 失败 | ESP32 仅支持 2.4GHz | 改用 2.4GHz SSID |

## 参考

- `examples/lock-app/esp32/README.md` — Lock 示例构建说明
- `examples/all-clusters-app/esp32/README.md` — All Clusters 示例与设备类型
- `docs/BUILDING.md` — 通用 CHIP 构建文档
- `recipes/commissioning_ble.md` — 烧录后通过 BLE 配网
