# SPI 全双工（Full-Duplex）通信链路搭建

> **适用摘要**: 使用 SPI 全双工作为 Host 与 Co-processor 之间的传输介质，完成协处理器（slave）固件烧录、主机（host）工程配置、引脚连接与链路验证。这是最易上手、可用跳线评估的传输方式。

## 触发意图

- "用 SPI 连接 host 和 slave"
- "ESP-Hosted SPI Full Duplex"
- "Standard SPI bring-up"
- "最快验证 ESP-Hosted 的方式"
- "iperf over SPI"

## 前置条件

| 条件 | 要求 |
|---|---|
| ESP-IDF | >= 5.3，已通过 installer 或 `docs/setup_esp_idf__latest_stable__linux_macos.sh` 安装 |
| 硬件 | 一片 ESP 作为 host，一片 ESP 作为 co-processor；6 根数据线 + Reset + GND |
| 跳线 | 长度 <= 10 cm，等长，尽量多接 GND |
| 参考文档 | `docs/spi_full_duplex.md` |

## 分步说明

### 1. 协处理器（slave）固件

```bash
# 在 ESP-IDF 工程目录之外��建 slave 工程
idf.py create-project-from-example "espressif/esp_hosted:slave"
cd <生成的 slave 工程目录>
idf.py set-target esp32c6        # 替换为你的协处理器型号

# 选择传输介质
idf.py menuconfig
# Example configuration -> Bus Config in between Host and Co-processor
#   -> Transport layer -> 选择 "SPI Full-duplex"
# 可在 "SPI Full-duplex" 菜单配置 MOSI/MISO/CLK/CS/Handshake/Data Ready/Reset 引脚、
#   SPI mode (1/2/3)、SPI 时钟频率、Checksum（建议启用）

idf.py build
idf.py -p <co-processor_serial_port> flash
```

> 若 host 拉低信号导致无法烧录 slave，先把 host 置于 bootloader：
> `esptool.py -p <host_serial_port> --before default_reset --after no_reset run`，再重试。

### 2. 主机（host）工程（以 ESP-IDF iperf 例程为例）

```bash
cd $IDF_PATH/examples/wifi/iperf

# 添加 ESP-Hosted 依赖
idf.py add-dependency "espressif/esp_wifi_remote"
idf.py add-dependency "espressif/esp_hosted"

# 编辑 main/idf_component.yml：删除/注释下面的冲突块（ESP32-P4 例程常见）
#   espressif/esp-extconn:
#     version: "~0.1.0"
#     rules:
#       - if: "target in [esp32p4]"

idf.py set-target esp32p4       # 替换为你的 host 型号
idf.py menuconfig
# Component config -> ESP-Hosted config
#   -> Transport layer = SPI Full-duplex
#   -> Slave chipset to be used = (与 slave 一致，例如 ESP32-C6)
#   -> SPI Configuration：时钟、引脚、Checksum(启用)

idf.py build
idf.py -p <host_serial_port> flash monitor
```

> 若 host 自带 Wi-Fi（如 ESP32-C3 作 host），需编辑
> `components/soc/<soc>/include/soc/Kconfig.soc_caps.in`，将所有 `WIFI` 相关项置为 `n`。

### 3. 硬件连接（host 侧默认 Kconfig 引脚）

| 信号 | ESP32 | ESP32-S2/S3 | ESP32-C2/C3/C5/C6 | ESP32-P4 |
|---|---|---|---|---|
| CLK | 14 | 12 | 6 | 9 |
| MOSI | 13 | 11 | 7 | 8 |
| MISO | 12 | 13 | 2 | 10 |
| CS | 15 | 10 | 10 | 7 |
| Handshake | 26 | 17 | 3 | 6 |
| Data Ready | 4 | 4 | 4 | 11 |
| Reset Out | 5 | 5 | 5 | 12 |

slave 侧默认引脚见 `docs/spi_full_duplex.md` 的 "Co-processor connections" 表。Host 的 Reset-Out 接到 slave 的 EN/RST。所有 GND 互联。

### 4. 链路验证

启动后应看到：

**Host 侧**
```
I (522) transport: Attempt connection with slave: retry[0]
I (1712) transport: Received INIT event from ESP32 peripheral
I (1715) transport: capabilities: 0xe8
I (1741) transport: Base transport is set-up
```

**Slave 侧**
```
I (492) fg_mcu_slave: ESP-Hosted-MCU Slave FW version :: X.Y.Z
I (511) fg_mcu_slave: Transport used :: SPI
```

### 5. iperf 性能测试（连接 AP 后）

```
sta_scan
sta_connect <SSID> <password>
sta_ip
# Host TX UDP
iperf -u -c <STA_IP> -t 60 -i 3
```

参考吞吐：Standard SPI FD（跳线/PCB）UDP 约 24 Mbits/s、TCP 约 22 Mbits/s。

## 常见错误

| 错误 | 原因 | 解决方法 |
|---|---|---|
| 一直 `Attempt connection with slave: retry[N]` | 双方传输介质不一致 | host 与 slave menuconfig 都选 "SPI Full-duplex" |
| 收不到 INIT 事件 | Handshake/Data Ready/Reset 接错或未接 | 核对 6 根数据线 + Reset + GND；Reset 接到 slave EN/RST |
| 偶发 RPC 错误 | SPI 无硬件检错且未启用 checksum | host/slave 都在 menuconfig 启用 Checksum |
| 通信不稳定 | 时钟过高或走线差 | 时钟先用 5 MHz，确认后再阶梯式提升；跳线 <= 10 cm |
| 无法烧录 slave | host 拉低信号 | 先 `esptool.py ... run` 把 host 置 bootloader |
| Quad SPI 不工作 | Quad 必须用 PCB | Quad/HD 信号完整性要求高，跳线不支持（且 ESP32 不支持 1-bit/Dual/Quad） |

## 参考

- `docs/spi_full_duplex.md`（理论与完整步骤）
- `examples/host_network_split__power_save/`（含 SPI 的网络分流与低功耗例程 sdkconfig）
- `examples/host_transport_config/`（运行期配置 SPI 的代码范式）
- README "Hosted Transports table"（吞吐参考值）
