# UART 传输链路搭建（Wi-Fi + 蓝牙）

> **适用摘要**: 使用 UART 作为 Host 与 Co-processor 之间的传输介质。UART 只需 2 根数据线（TX/RX）+ Reset + GND，所有 ESP 芯片都支持，适合低吞吐场景。该模式下 Wi-Fi 与蓝牙以 Hosted HCI（复用模式）在同一根 UART 上传输。不要与"仅蓝牙的专用 HCI over UART"混淆。

> Version: selected repo/component version; align with the user project and dependency manifest.
> Evidence: `repos/esp-hosted-mcu/resources/`, source/examples in `repos/esp-hosted-mcu/`, and this recipe path `repos/esp-hosted-mcu/recipes/bringup_uart.md`.
> Validation: draft metadata added from repository routing; verify APIs, Kconfig symbols, and component dependencies against the selected version.

## 触发意图

- "用 UART 连接 host 和 slave"
- "ESP-Hosted UART"
- "低引脚数传输"
- "Wi-Fi + BT 共用一根 UART"
- "Hosted HCI over UART"

## 前置条件

| 条件 | 要求 |
|---|---|
| 硬件 | host 与 slave 各一片 ESP；2 根数据线 + Reset + GND |
| 吞吐预期 | 低（921600 波特参考下 iperf 约 0.6 Mbits/s；P4+C6 EV 板 4 Mbit/s 可达 ~3.3 Mbit/s） |
| ESP-IDF | >= 5.3；注意 v5.5 下 UART slave 可能 IRAM 不足 |
| 参考文档 | `docs/uart.md` |

## 分步说明

### 1. 协处理器（slave）固件

```bash
idf.py create-project-from-example "espressif/esp_hosted:slave"
cd <生成的 slave 工程目录>
idf.py set-target esp32c3        # 任意支持 ESP 作 slave

idf.py menuconfig
# Example configuration -> Bus Config -> Transport layer = UART
# UART Configuration：配置 Tx/Rx GPIO、波特率、Checksum（建议启用）
idf.py build
idf.py -p <co-processor_serial_port> flash
```

> ESP-IDF v5.5 构建 UART slave 若报 IRAM 不足：
> - menuconfig 启用 `Component config -> ESP Ringbuf -> Place non-ISR ringbuf functions into flash`，或
> - 在 `slave/sdkconfig.defaults.esp32` 解注释 `CONFIG_RINGBUF_PLACE_FUNCTIONS_INTO_FLASH=y` 后重新生成 sdkconfig。

### 2. 主机（host）工程

```bash
cd $IDF_PATH/examples/wifi/iperf
idf.py add-dependency "espressif/esp_wifi_remote"
idf.py add-dependency "espressif/esp_hosted"
# 删除 main/idf_component.yml 中的 esp-extconn 块

idf.py set-target esp32p4
idf.py menuconfig
# Component config -> ESP-Hosted config -> Transport layer = UART
#   -> Slave chipset to be used = (与 slave 一致)
# UART Configuration：UART Tx/Rx GPIO、波特率、Checksum
#   避开 ESP 的 UART0（用于调试输出），ESP-Hosted 使用另一个 UART 控制器
idf.py build
idf.py -p <host_serial_port> flash monitor
```

### 3. 硬件连接

UART 可用任意 GPIO，但**避开 UART0 的 TX0/RX0**（调试用）。连接：

```
host TX  <---> slave RX
host RX  <---> slave TX
host Reset-Out <---> slave EN/RST
GND <---> GND（多接几根）
```

> 实际波特率由硬件决定。若实际波特率漂移超过几个百分点，接收端将解码失败——必要时用示波器/逻辑分析仪核对。

### 4. 链路验证

**Host**
```
I (1650) transport: Received INIT event from ESP32 peripheral
I (1662) transport: capabilities: 0x88
I (1666) transport:        - BLE only
I (1680) transport:      * WLAN over UART
I (1693) transport: Base transport is set-up
```

**Slave**
```
I (503) fg_mcu_slave: Transport used :: UART only
I (526) h_bt: - BT/BLE
I (529) h_bt:    - BLE only
I (543) fg_mcu_slave: - WLAN over UART
I (551) h_bt:    - HCI Over UART (VHCI)
```

### 5. "仅蓝牙专用 UART"（独立 HCI）

若只需蓝牙、且希望标准透明 HCI（不复用、可移植到任意带 BT 控制器的协处理器），可用专用 HCI over UART：需要额外 2 或 4 根 GPIO（视硬件流控）。在该配置下 Wi-Fi 走另一传输或不用。详见 `docs/bluetooth_design.md` 第 6 节。

## 常见错误

| 错误 | 原因 | 解决方法 |
|---|---|---|
| 误用 UART0 引脚 | 与调试输出冲突 | 改用其他 UART 控制器的引脚 |
| IRAM 不足（IDF v5.5） | ringbuf 占 IRAM | 启用 `CONFIG_RINGBUF_PLACE_FUNCTIONS_INTO_FLASH=y` |
| 吞吐远低于预期 | 波特率太低或漂移 | 提高波特率；P4+C6 EV 板可到 4 Mbit/s；核对实际波特 |
| 数据乱码/丢包 | 波特率漂移、Checksum 关闭 | 启用 Checksum；核对波特；缩短走线 |
| 期望高吞吐 | UART 本身低吞吐 | 改用 SPI/SDIO；UART 仅推荐低吞吐环境 |

## 参考

- `docs/uart.md`
- `docs/bluetooth_design.md`（标准 HCI over UART 与 Hosted HCI 区别）
- `examples/host_nimble_bleprph_host_only_uart_hci/`（标准 HCI over UART 的 NimBLE 例程）
- README "Hosted Transports table"
