# 通过 Wi-Fi / Bypass 模式配网

> **适用摘要**: 当不使用 BLE 时，将设备配网模式切换为 Wi-Fi 或 Bypass：在 menuconfig 设置 SSID/密码与 Rendezvous 模式，使设备开机即尝试联网（Bypass 跳过安全配对，用于调试）。

## 触发意图

- "Bypass 配网"
- "Wi-Fi Rendezvous"
- "不用 BLE 配网"
- "调试模式直接联网"
- "RENDEZVOUS_MODE_BYPASS"

## 前置条件

| 条件 | 要求 |
|---|---|
| Rendezvous 模式 | `CONFIG_RENDEZVOUS_MODE` = 1 (Wi-Fi) 或 0 (Bypass) |
| Wi-Fi 凭据 | `Component config -> CHIP Device Layer -> WiFi Station Options` 中填 SSID/密码 |
| 控制器 | chip-tool `pairing bypass <ip> <port>` 或 python controller |
| 网络 | 2.4 GHz SSID（ESP32 不支持 5 GHz） |

## 分步说明

### 1. menuconfig 选择模式并填凭据

```bash
idf.py menuconfig
```

- `Demo -> Rendezvous Mode` → `Bypass`（=0）或 `Wi-Fi`（=1）
- `Component config -> CHIP Device Layer -> WiFi Station Options`：填 SSID 与 Password
- 重新构建烧录：

```bash
idf.py build
idf.py -p /dev/ttyUSB0 flash monitor
```

### 2. 确认设备已获取 IP

```
I (5524) chip[DL]: SYSTEM_EVENT_STA_GOT_IP
I (5524) chip[DL]: IPv4 address changed on WiFi station interface: 192.168.0.30...
```

记录设备 IP（如 `192.168.0.30`）。CHIP 默认端口 `11097`（见 `src/INET`/`CHIP_PORT`）。

### 3. 用 chip-tool Bypass 配对

```bash
./out/debug/chip-tool pairing bypass 192.168.0.30 11097
```

### 4. 下发集群命令

```bash
./out/debug/chip-tool onoff on 1
./out/debug/chip-tool onoff off 1
```

> 注意：Bypass 模式跳过标准 PASE 安全配对，仅适合受控调试网络，不可用于生产。

## 常见错误

| 错误 | 原因 | 解决方法 |
|---|---|---|
| 设备反复重启不联网 | Wi-Fi 凭据空或错 | menuconfig 重新填 SSID/密码后 build+flash |
| `pairing bypass` 连接被拒 | 端口/IP 错 | 确认日志中 IP 与 CHIP 端口（默认 11097） |
| 关联 5GHz 失败 | ESP32 仅 2.4GHz | 改 2.4GHz SSID |
| 改了模式但行为不变 | 未重新烧录 | menuconfig 后必须 `idf.py build` + `flash` |
| `CONFIG_RENDEZVOUS_MODE` 数值错 | Kconfig 映射记错 | Bypass=0, Wi-Fi=1, BLE=2, Thread=4, Ethernet=8 |

## 参考

- `examples/lock-app/esp32/README.md` — "Commissioning and cluster control"（Bypass/Wi-Fi 说明）
- `examples/lock-app/esp32/main/Kconfig.projbuild` — `RENDEZVOUS_MODE` choice 与数值映射
- `examples/chip-tool/README.md` — `pairing bypass` 用法
