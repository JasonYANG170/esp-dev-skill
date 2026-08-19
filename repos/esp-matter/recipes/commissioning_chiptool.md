# 用 chip-tool 入网与控制

> **适用摘要**: 用 host 端 chip-tool 作为 commissioner，通过 BLE-Wi-Fi 或 BLE-Thread 把 Matter 设备加入 fabric，并用 cluster 命令读写属性。

> Version: selected repo/component version; align with the user project and dependency manifest.
> Evidence: `repos/esp-matter/resources/`, source/examples in `repos/esp-matter/`, and this recipe path `repos/esp-matter/recipes/commissioning_chiptool.md`.
> Validation: draft metadata added from repository routing; verify APIs, Kconfig symbols, and component dependencies against the selected version.

## 触发意图

- "怎么 commission Matter 设备"
- "chip-tool 怎么用"
- "pairing ble-wifi"
- "onoff toggle"
- "入网 / 配网"

## 前置条件

| 条件 | 要求 |
|---|---|
| 工具 | `connectedhomeip/connectedhomeip/out/host/chip-tool`（`install.sh` 构建） |
| 设备 | 已烧录示例（默认 passcode `20202021`、discriminator `3840`） |
| 网络 | Wi-Fi AP（Wi-Fi 设备）或 Thread dataset（Thread 设备） |

## 分步说明

### 1. 启动 chip-tool 交互模式（推荐，复用 CASE 会话）

```bash
chip-tool interactive start
```

> macOS 用 BLE 入网前要安装 Bluetooth Central Matter Client Developer Mode Profile（见 `developing.rst` 的 macOS 说明）。

### 2. BLE-Wi-Fi 入网（Wi-Fi 类设备：esp32/esp32s3/esp32c3/esp32c2/esp32c6/esp32p4/esp32c5/esp32c61）

```text
pairing ble-wifi 0x7283 <ssid> <passphrase> 20202021 3840
```

- `0x7283`：自选的 node_id
- `20202021`：setup_passcode（测试值，来自 `CONFIG_ENABLE_TEST_SETUP_PARAMS`）
- `3840`：discriminator

### 3. BLE-Thread 入网（Thread 类设备：esp32h2/esp32c6/esp32c5）

```text
pairing ble-thread 0x7283 hex:<operationalDataset> 20202021 3840
```

`operationalDataset` 是 Thread 网络 Active Operational Dataset 的十六进制串。

### 4. 用 manual code / QR code 入网

```text
pairing code-wifi 0x7283 <ssid> <passphrase> 34970112332
pairing code-wifi 0x7283 <ssid> <passphrase> MT:Y.K9042C00KA0648G00
```

默认 manual code `34970112332`、QR `MT:Y.K9042C00KA0648G00` 对应 Version 0 / VID 0xFFF1 / PID 0x8000 / Discriminator 3840 / Passcode 20202021。

### 5. cluster 控制

```text
onoff toggle 0x7283 1
onoff on 0x7283 1
levelcontrol move-to-level 10 0 0 0 0x7283 1
levelcontrol move-to-level 100 0 0 0 0x7283 1
colorcontrol move-to-color-temperature 0 10 0 0 0x7283 1
```

格式：`<cluster> <command> <args...> <node-id> <endpoint-id>`。

### 6. 用自定义 PAA 入网（自定义 DAC 时）

```bash
./chip-tool pairing ble-wifi 1234 my_SSID my_PASSPHRASE my_PASSCODE my_DISCRIMINATOR \
    --paa-trust-store-path /path/to/PAA-Certificates/
```

PAA 证书须为 DER 格式。

### 7. 修改入网凭据（生产）

用 `esp-matter-mfg-tool` 生成新的 factory partition（含新的 passcode/discriminator/VID/PID/CD），烧录后用其产出的 QR/manual code 入网。见 `recipes/factory_data_attestation.md`。

## 常见错误

| 错误 | 原因 | 解决方法 |
|---|---|---|
| pairing 超时无响应 | 设备没擦 flash / 没广播 | `idf.py erase_flash flash monitor`，确认串口有 BLE 广播 |
| `SRC_ERR_INVALID_SETUP_CSR` 等 | 测试凭据被关 | 保留 `CONFIG_ENABLE_TEST_SETUP_PARAMS=y`，或刷 factory 分区 |
| Thread 入网失败 | dataset 不匹配 | 确认 `operationalDataset` 是当前 Thread 网络的 Active Dataset |
| 重复 commission 报错 | node_id 已在 fabric | 用新 node_id 或先 `chip-tool pairing unpair <node>` |
| macOS BLE 不工作 | 未装开发者 profile | 装 Bluetooth Central Matter Client Developer Mode Profile |

## 参考

- `D:/esp-skill/espressif-repos/esp-matter/docs/en/developing.rst` — Commissioning and Control / Test Setup (CHIP Tool)
- `D:/esp-skill/espressif-repos/esp-matter/docs/en/using_chip_tool.rst`（WSL 下用 chip-tool）
- `D:/esp-skill/espressif-repos/esp-matter/docs/en/controller.rst`（设备端 controller）
- `D:/esp-skill/espressif-repos/esp-matter/examples/light/README.md`
