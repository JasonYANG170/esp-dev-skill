# 构建与烧录 Thread Border Router

> **适用摘要**: 从零拉取 esp-thread-br、构建 RCP 镜像、配置并烧录 `basic_thread_border_router`，使设备连上 Wi-Fi 并形成 Thread 网络。

## 触发意图
- "怎么构建 esp-thread-br"
- "如何烧录 Thread Border Router"
- "idf.py set-target esp32s3"
- "构建 ot_rcp"

## 前置条件

| 条件 | 要求 |
|---|---|
| ESP-IDF | >= 5.1.0，推荐 v5.5.4（仓库 README 指定） |
| 硬件 | ESP Thread Border Router 板（ESP32-S3 + ESP32-H2）或 Standalone 模组 |
| 参考 | `docs/en/dev-guide/build_and_run.rst`、`examples/basic_thread_border_router/README.md` |

## 分步说明

### 1. 准备 ESP-IDF

```bash
git clone --recursive https://github.com/espressif/esp-idf.git
cd esp-idf
git checkout v5.5.4
git submodule update --init --depth 1
./install.sh
. ./export.sh
```

### 2. 克隆 esp-thread-br

```bash
cd ..
git clone --recursive https://github.com/espressif/esp-thread-br.git
```

### 3. 构建 RCP 镜像（ot_rcp）

RCP 固件无需手动烧写，BR 构建期会自动打包并首发时烧到 ESP32-H2。

```bash
cd $IDF_PATH/examples/openthread/ot_rcp
idf.py set-target esp32h2
idf.py build
```

> 默认 BR 板用 UART0 @ 460800。Standalone 模组需修改 `esp_ot_config.h` 的 `radio_uart_config`。

### 4. 配置并构建 BR

```bash
cd esp-thread-br/examples/basic_thread_border_router
idf.py set-target esp32s3          # 官方 BR 板默认
idf.py menuconfig                   # 按需调板型/引脚/特性
idf.py build
```

`LWIP_IPV6_NUM_ADDRESSES` 必须与 IDF 版本匹配：

| IDF 版本 | LWIP_IPV6_NUM_ADDRESSES |
|---|---|
| v5.1.4 及更早 | 8 |
| v5.2.2 及更早 | 8 |
| v5.3.0 | 8 |
| v5.1.5 / v5.2.3 / v5.3.1 / v5.4 及之�� | 12 |

### 5. 烧录并监控

```bash
idf.py -p PORT flash monitor
```

成功标志（Wi-Fi backbone）：

```
I (5729) example_connect: - IPv4 address: 192.168.1.102
I(5779) OPENTHREAD:[I] Platform------: RCP reset: RESET_POWER_ON
I(5809) OPENTHREAD:[N] Platform------: RCP API Version: 6
I (5929) OPENTHREAD: OpenThread attached to netif
I(5939) OPENTHREAD:[I] SrpServer-----: Selected port 53535
```

### 6. （未启用 AUTO_START 时）手动起网

```
> wifi connect -s <ssid> -p <psk>
> dataset init new
> dataset commit active
> ifconfig up
> thread start
```

### 7. 验证角色

```
> state
leader
Done
```

## 常见错误

| 错误 | 原因 | 解决方法 |
|---|---|---|
| `Wait for response timeout` (Spinel) | RCP UART/SPI 引脚或波特率不匹配 | 核对 `esp_ot_config.h` 引脚与波特率 460800；ESP32-S3 避开 GPIO17/18 |
| 一直 `detached` | `thread start` 在 `ifconfig up` 之前 | 严格按 dataset→ifconfig up→thread start 顺序 |
| 构建报 LWIP_IPV6_NUM_ADDRESSES 错 | 与 IDF 版本不匹配 | 按 IDF 版本表设 8 或 12 |
| 首发后 RCP 不工作 | ot_rcp 未构建 / 未打包 | 先构建 ot_rcp 再构建 BR；确认 `CONFIG_AUTO_UPDATE_RCP=y` |

## 参考
- `docs/en/dev-guide/build_and_run.rst`
- `examples/basic_thread_border_router/README.md`
- `examples/basic_thread_border_router/sdkconfig.defaults`
