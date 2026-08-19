# SDIO（1-Bit / 4-Bit）通信链路搭建

> **适用摘要**: 使用 SDIO 作为 Host 与 Co-processor 之间的高性能传输介质。SDIO 4-Bit 是 ESP-Hosted 吞吐最高的传输方式（shield-box 实测 UDP ~79.5 / TCP ~53.4 Mbits/s），但信号完整性要求严格，必须用 PCB 并加外部上拉。

> Version: selected repo/component version; align with the user project and dependency manifest.
> Evidence: `repos/esp-hosted-mcu/resources/`, source/examples in `repos/esp-hosted-mcu/`, and this recipe path `repos/esp-hosted-mcu/recipes/bringup_sdio.md`.
> Validation: draft metadata added from repository routing; verify APIs, Kconfig symbols, and component dependencies against the selected version.

## 触发意图

- "用 SDIO 连接 host 和 slave"
- "ESP32-P4 + C6 SDIO"
- "SDIO 4-bit 吞吐优化"
- "SDIO stream vs packet mode"
- "ESP-Hosted 最高性能"

## 前置条件

| 条件 | 要求 |
|---|---|
| 硬件 | SDIO **slave** 只能选 ESP32 / ESP32-C6 / ESP32-C5 / ESP32-C61；SDIO **master** 选 ESP32 / ESP32-S3 / ESP32-P4 |
| PCB | 4-bit 必须用 PCB（跳线仅可用于 1-bit 评估，且仍需上拉） |
| 上拉电阻 | CMD、DAT0、DAT1、DAT2、DAT3 各加 51 kΩ 外部上拉 |
| 电平 | 3.3 V |
| 参考文档 | `docs/sdio.md` |

## 分步说明

### 1. 协处理器（slave）固件

```bash
idf.py create-project-from-example "espressif/esp_hosted:slave"
cd <生成的 slave 工程目录>
idf.py set-target esp32c6        # SDIO slave 仅支持 ESP32/C5/C6/C61

idf.py menuconfig
# Example configuration -> Bus Config -> Transport layer = SDIO
# SDIO Configuration：可设 slave GPIO、SDIO Mode、时序等

idf.py build
idf.py -p <co-processor_serial_port> flash
```

### 2. 主机（host）工程

```bash
cd $IDF_PATH/examples/wifi/iperf
idf.py add-dependency "espressif/esp_wifi_remote"
idf.py add-dependency "espressif/esp_hosted"
# 删除 main/idf_component.yml 中的 esp-extconn 块

idf.py set-target esp32p4        # SDIO master: ESP32 / S3 / P4
idf.py menuconfig
# Component config -> ESP-Hosted config -> Transport layer = SDIO
#   -> Slave chipset to be used = (与 slave 一致)
# Hosted SDIO Configuration:
#   -> SDIO Bus Width = 4 Bit (或 1 Bit 用于先期评估)
#   -> SDIO Clock Freq (in kHz) = 先 400~20000，验证后再升（最高 50 MHz）
#   -> SDIO slot（P4 Slot1 默认，引脚可重映射；ESP32 仅固定 IO_MUX）
idf.py build
idf.py -p <host_serial_port> flash monitor
```

### 3. 硬件连接

**Host 引脚**（CMD/D0–D3 需外部上拉）

| 信号 | ESP32 | ESP32-S3 | ESP32-P4 + C6(EV board) |
|---|---|---|---|
| CLK | 14 | 19 | 18 |
| CMD | 15 | 47 | 19 |
| D0 | 2 | 13 | 14 |
| D1 | 4 | 35 | 15 |
| D2 | 12 | 20 | 16 |
| D3 | 13 | 9 | 17 |
| Reset Out | 5 | 42 | 54 |

**Slave 引脚**（SDIO slave 提供方：ESP32/C5/C6）

| 信号 | ESP32 | ESP32-C6 | ESP32-C5 |
|---|---|---|---|
| CLK | 14 | 19 | 9 |
| CMD | 15 | 18 | 10 |
| D0 | 2 | 20 | 8 |
| D1 | 4 | 21 | 7 |
| D2 | 12 | 22 | 14 |
| D3 | 13 | 23 | 13 |
| Reset In | EN | EN/RST | RST |

> 经典 ESP32 作 master 或 slave 时，可能需要一次性、不可逆的 **eFuse 烧写**（修正某数据引脚电压）。烧写前务必核对 ESP-IDF `sd_pullup_requirements` 文档，确认你的型号确实需要。

### 4. 性能与内存：Stream vs Packet 模式

SDIO slave 有两种模式（host 与 slave 必须一致）：

| 模式 | 行为 | 内存 | 吞吐 |
|---|---|---|---|
| Streaming（默认） | slave 把多个 Tx 包合并成一个大包，host 一次取回再拆分 | 较高（双缓冲 `2 * Tx队列 * 1536`） | 最高 |
| Packet | slave 逐包排队，host 逐包取 | 较低（host 约 `2*1536`） | 较低 |

切换到 Packet 模式（节省 host 内存）：
- slave menuconfig：`Example Configuration -> Bus Config -> SDIO Configuration -> Enable SDIO Streaming Mode` **取消勾选**
- host menuconfig：`ESP-Hosted config -> Hosted SDIO Configuration -> SDIO Receive Optimization` 改为 `Always Rx Max Packet size`（或 `No optimization`）

调队列大小改变内存/吞吐平衡（slave 侧）：
`Example Configuration -> Bus Config -> SDIO Configuration -> SDIO Tx queue size`（默认 20）。Tx 队列 25 以上吞吐基本饱和。

### 5. 链路验证

**Slave**：`Transport used :: SDIO`；**Host**：`capabilities: 0xe8`、`HCI over SDIO`、`Base transport is set-up`。

## 常见错误

| 错误 | 原因 | 解决方法 |
|---|---|---|
| 选了 ESP32-C3/S3 作 SDIO slave | 这些芯片不支持 SDIO slave | 改用 SPI/UART，或换 ESP32/C5/C6/C61 作 slave |
| 1-bit 模式下 slave 进入 SPI 模式 | DAT2/DAT3 无上拉 | 即便 1-bit，CMD/DAT0–D3 都必须加上拉 |
| 经典 ESP32 SDIO 不稳 | 未烧 eFuse / 电平不对 | 核对 `sd_pullup_requirements`，必要时烧 eFuse（不可逆，谨慎） |
| 链路偶发断开 | 走线不等长、时钟过高 | 等长走线 < 5 cm；时钟先 5 MHz，再阶梯式升 |
| host 内存吃紧（Streaming） | 双缓冲占用大 | 切 Packet 模式或调小 slave `SDIO Tx queue size` |
| 同时上拉缺失导致初始化失败 | 漏接上拉 | 所有 SDIO 线（含 DAT2/DAT3）必须 51 kΩ 上拉 |

## 参考

- `docs/sdio.md`（含 Stream/Packet 内存表、完整步骤）
- `docs/esp32_p4_function_ev_board.md`（P4+C6 EV 板快速验证）
- `examples/host_transport_config/`（运行期 SDIO 配置代码）
- README "Hosted Transports table"
