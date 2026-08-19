# 通过 BLE 完成 Rendezvous 配网

> **适用摘要**: 设备烧录后，在默认 BLE Rendezvous 模式下，使用 chip-tool 或 python controller 完成 PASE 配对、下发 Wi-Fi 凭据、关闭 BLE 并解析 mDNS，使设备加入网络。这是示例的默认配网路径。

> Version: selected repo/component version; align with the user project and dependency manifest.
> Validation: draft metadata added from repository routing; verify APIs, Kconfig symbols, and component dependencies against the selected version.

## 触发意图

- "BLE 配网"
- "Rendezvous BLE"
- "commissioning over BLE"
- "chip-device-ctrl ble-scan"
- "如何让 ESP32 Matter 设备入网"

## 前置条件

| 条件 | 要求 |
|---|---|
| 设备已烧录 | Rendezvous 模式 = BLE（默认 `CONFIG_RENDEZVOUS_MODE=2`） |
| BLE 栈 | `sdkconfig`：`CONFIG_BT_ENABLED=y` + `CONFIG_BT_NIMBLE_ENABLED=y` |
| 默认凭据 | discriminator `3840`，setup-pin-code `20202021`（可经 menuconfig 改） |
| 控制器 | chip-tool（`examples/chip-tool`）或 python `chip-device-ctrl` |
| Wi-Fi | 一个 2.4 GHz SSID（ESP32 不支持 5 GHz） |

## 分步说明（Python Controller）

### 1. 启动 python controller

```bash
cd {path-to-connectedhomeip}
./scripts/build_python.sh -m platform
source ./out/python_env/bin/activate
chip-device-ctrl
```

### 2. BLE 扫描并建立安全会话

```
chip-device-ctrl > ble-scan
chip-device-ctrl > connect -ble 3840 20202021 135246
```

参数：discriminator `3840`、pin `20202021`、node ID `135246`（自选，后续命令复用）。

### 3. 下发 Wi-Fi 凭据并使能网络

```
chip-device-ctrl > zcl NetworkCommissioning AddWiFiNetwork 135246 0 0 ssid=str:TESTSSID credentials=str:TESTPASSWD breadcrumb=0 timeoutMs=1000
chip-device-ctrl > zcl NetworkCommissioning EnableNetwork 135246 0 0 networkID=str:TESTSSID breadcrumb=0 timeoutMs=1000
```

### 4. 关闭 BLE 并解析 mDNS 地址

```
chip-device-ctrl > close-ble
chip-device-ctrl > resolve 0 135246
```

设备成功关联后日志会出现：

```
I (5524) chip[DL]: SYSTEM_EVENT_STA_GOT_IP
```

### 5. 下发集群命令验证

```
chip-device-ctrl > zcl OnOff Off 135246 1 0
```

## 分步说明（chip-tool CLI）

```bash
# 配对（discriminator/pin 为示例默认值）
./out/debug/chip-tool pairing ble 20202021 3840

# 配对成功后下发 OnOff 命令（endpoint 必须在 1..240）
./out/debug/chip-tool onoff on 1
./out/debug/chip-tool onoff off 1
```

> chip-tool 在 BLE 配网流程中由其内置 network commissioning 完成 Wi-Fi 下发；具体参数取决于构建版本。详细命令见 `examples/chip-tool/README.md`。

## 常见错误

| 错误 | 原因 | 解决方法 |
|---|---|---|
| `ble-scan` 找不到设备 | Rendezvous 非 BLE / 未广播 | menuconfig 选 BLE 模式重新烧录；短按 function 按钮触发 fast advertising |
| `connect -ble` 失败 | discriminator/pin 不匹配 | 确认示例默认值 3840/20202021 或自定义值 |
| `EnableNetwork` 后无 IP | SSID 为 5GHz 或密码错 | 用 2.4GHz SSID；`erase_flash` 重置后重试 |
| `resolve` 超时 | mDNS 未启动 | 确认设备在 `kInterfaceIpAddressChanged` 调了 `chip::app::Mdns::StartServer()` |
| 关 BLE 太早 | 凭据未提交 | 先 `EnableNetwork` 成功再 `close-ble` |

## 参考

- `examples/all-clusters-app/esp32/README.md` — "Commissioning over BLE" 完整流程
- `examples/chip-tool/README.md` — chip-tool `pairing ble` 用法
- `recipes/python_controller.md` — python controller 进阶
- `recipes/device_event_handling.md` — 设备端 IP/会话事件处理
