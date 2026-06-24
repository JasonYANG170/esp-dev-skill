# 使用 python controller 配网与集群控制

> **适用摘要**: 用 CHIP 自带的 python `chip-device-ctrl` 完成 BLE 扫描、配对、网络下发、DNS-SD 解析及 ZCL 集群命令下发。适合脚本化测试与自动化配网验证。

## 触发意图

- "python 控制器"
- "chip-device-ctrl"
- "脚本化配网"
- "zcl 命令测试"
- "自动化 Matter 测试"

## 前置条件

| 条件 | 要求 |
|---|---|
| 构建脚本 | `./scripts/build_python.sh -m platform` |
| 控制器 | `chip-device-ctrl`（位于 `out/python_env/bin/`） |
| 设备 | 已烧录示例（默认 BLE + disc 3840 / pin 20202021） |
| 主机蓝牙 | 支持 BLE 的主机适配器（Linux 需 bluez ≥ 5.55） |

## 分步说明

### 1. 构建并激活 python 环境

```bash
cd {path-to-connectedhomeip}
./scripts/build_python.sh -m platform
source ./out/python_env/bin/activate
chip-device-ctrl
```

### 2. BLE 扫描与配对

```
chip-device-ctrl > ble-scan
chip-device-ctrl > connect -ble 3840 20202021 135246
```

参数：discriminator `3840`、setup-pin-code `20202021`、node ID `135246`（后续命令复用该 ID）。

### 3. 网络下发（Wi-Fi）

```
chip-device-ctrl > zcl NetworkCommissioning AddWiFiNetwork 135246 0 0 ssid=str:TESTSSID credentials=str:TESTPASSWD breadcrumb=0 timeoutMs=1000
chip-device-ctrl > zcl NetworkCommissioning EnableNetwork 135246 0 0 networkID=str:TESTSSID breadcrumb=0 timeoutMs=1000
chip-device-ctrl > close-ble
chip-device-ctrl > resolve 0 135246
```

### 4. 下发 ZCL 集群命令

通用 `zcl` 语法：`zcl <Cluster> <Command> <nodeId> <endpoint> <groupId> [key=value ...]`

```
chip-device-ctrl > zcl OnOff Off 135246 1 0
chip-device-ctrl > zcl OnOff On  135246 1 0
```

### 5. 设备端日志确认

设备收到写入后调用 `PostAttributeChangeCallback`：

```
PostAttributeChangeCallback - Cluster ID: '0x0006', EndPoint ID: '0x01', Attribute ID: '0x0000'
```

## 常见错误

| 错误 | 原因 | 解决方法 |
|---|---|---|
| `build_python.sh` 失败 | 缺系统依赖 | 安装 `libdbus-1-dev libglib2.0-dev libavahi-client-dev bluez` |
| `ble-scan` 无结果 | 主机蓝牙不可用 / 设备非 BLE 模式 | 确认 bluez ≥ 5.55；设备 menuconfig 选 BLE |
| `connect` 失败 | pin/disc 不对 | 用默认 20202021 / 3840 |
| `resolve` 超时 | mDNS 未起 | 设备端确认 IP 变化时调用 `Mdns::StartServer()` |
| `zcl` 命令报 cluster 不支持 | endpoint 无该 server | 确认 endpoint（1..240）实现了该集群（见 all-clusters-app） |

## 参考

- `examples/all-clusters-app/esp32/README.md` — "Setting up Python Controller" + "Cluster control"
- `src/controller/python/` — python controller 源码
- `recipes/commissioning_ble.md` — BLE 配网完整流程
