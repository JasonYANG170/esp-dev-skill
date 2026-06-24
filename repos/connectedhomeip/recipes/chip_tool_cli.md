# 使用 chip-tool CLI 配对并下发 ZCL 命令

> **适用摘要**: 用 CHIP 自带的命令行客户端 `chip-tool` 配对设备（BLE/Bypass）并列出/调用支持的集群命令。适合快速功能验证与命令速查。

## 触发意图

- "chip-tool"
- "chip 命令行客户端"
- "pairing ble / bypass"
- "onoff 命令"
- "如何控制 Matter 设备"

## 前置条件

| 条件 | 要求 |
|---|---|
| 构建 | GN + ninja（主机侧，非 ESP-IDF） |
| 产物 | `examples/chip-tool/out/debug/chip-tool` |
| 设备 | 已烧录示例（默认 BLE + disc 3840 / pin 20202021），或 Bypass 已联网 |

## 分步说明

### 1. 构建 chip-tool

```bash
cd examples/chip-tool
git submodule update --init
source third_party/connectedhomeip/scripts/activate.sh
gn gen out/debug
ninja -C out/debug
```

产物：`out/debug/chip-tool`。

### 2. 查看支持的集群

```bash
./out/debug/chip-tool
```

输出示例集群列表：`onoff`、`levelcontrol`、`colorcontrol`、`doorlock`、`temperaturemeasurement`、`groups`、`scenes`、`identify`、`basic`、`barriercontrol`、`iaszone`、`pairing`、`payload`。

### 3. 配对设备

```bash
# BLE（默认凭据）
./out/debug/chip-tool pairing ble 20202021 3840

# Bypass（已联网，IP + 端口）
./out/debug/chip-tool pairing bypass 192.168.0.30 11097

# 解除配对
./out/debug/chip-tool pairing unpair
```

### 4. 下发集群命令（endpoint 1..240）

```bash
./out/debug/chip-tool onoff on 1
./out/debug/chip-tool onoff off 1
```

> endpoint 必须在 1 到 240 之间。

## 常见错误

| 错误 | 原因 | 解决方法 |
|---|---|---|
| `gn gen` 失败 | 未激活环境 / 缺 ninja | `source .../activate.sh`；`apt-get install ninja-build` |
| `pairing ble` 超时 | 设备未广播 / 凭据错 | 确认 BLE 模式与默认 3840/20202021 |
| `pairing bypass` 连接被拒 | IP/端口错 | 用设备日志中的 IP；默认端口 11097 |
| 命令报 "No cluster" | 集群名拼错 / 设备未实现 | 先 `./chip-tool` 查列表；用 all-clusters-app |
| endpoint 报越界 | 用了 0 或 >240 | endpoint 取 1..240 |

## 参考

- `examples/chip-tool/README.md` — 构建与 pairing/集群命令用法
- `recipes/commissioning_ble.md` — BLE 配网设备端配合
- `recipes/commissioning_wifi_bypass.md` — Bypass 配对
