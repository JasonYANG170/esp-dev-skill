# ESP-Hosted-MCU 常见陷阱汇总

> 汇总自仓库 `docs/`（troubleshooting、各传输文档、migration、design_consideration）与头文件说明。每条给出症状、根因、解决。

## A. 链路建立 / INIT 事件

1. **host 一直 `Attempt connection with slave: retry[N]`，收不到 INIT 事件**
   - 根因：host 与 slave 选择了不同传输介质，或引脚/Reset/GND 未正确连接。
   - 解决：双方 menuconfig 传输一致；核对全部数据线 + Reset + GND；SDIO 确认上拉；时钟先降到 5 MHz 验证。

2. **SDIO slave 进入 SPI 模式**
   - 根因：DAT2/DAT3 没有上拉（即便 1-bit 模式也必须上拉）。
   - 解决：CMD、DAT0–DAT3 全部加 51 kΩ 外部上拉。

3. **经典 ESP32 SDIO 不稳**
   - 根因：某数据引脚电压需 eFuse 修正（一次性、不可逆）。
   - 解决：按 ESP-IDF `sd_pullup_requirements` 核对，确认需要后再烧 eFuse。

4. **选了 ESP32-C3 / ESP32-S3 作 SDIO slave**
   - 根因：这些芯片不支持 SDIO slave 模式。
   - 解决：SDIO slave 仅 ESP32/C5/C6/C61；C3/S3 改用 SPI 或 UART。

5. **host Wi-Fi 例程在 ESP32-P4 上冲突**
   - 根因：`main/idf_component.yml` 残留 `espressif/esp-extconn`，与 `esp_hosted` 互斥。
   - 解决：删除/注释 esp-extconn 块，确保已加 `esp_wifi_remote` + `esp_hosted`。

6. **host 自带 Wi-Fi 时仍报冲突**
   - 根因：host 原生 Wi-Fi caps 未关。
   - 解决：编辑 `components/soc/<soc>/include/soc/Kconfig.soc_caps.in`，所有 WIFI 项置 `n`。

## B. 初始化顺序 / 运行期

7. **`esp_wifi_*` 调用失败或行为异常**
   - 根因：未在 `esp_hosted_init()` + `connect_to_slave()` 且收到 `ESP_HOSTED_EVENT_TRANSPORT_UP` 之前调用 Wi-Fi。
   - 解决：严格顺序 netif → event loop → hosted init → connect → 等 TRANSPORT_UP → Wi-Fi。

8. **BT host 栈初始化失败（v2.5.2+）**
   - 根因：协处理器 BT 控制器默认关闭。
   - 解决：在 BT host 栈之前调用 `esp_hosted_bt_controller_init()` + `enable()`。

9. **设置 BT MAC 不生效**
   - 根因：在 controller enable 之后才 set；或设置为永久但实际是临时（重启恢复）。
   - 解决：在 enable 之前 set；如需永久请在协处理器侧配置。

10. **传输偶发 RPC 错误（SPI）**
    - 根因：SPI 无硬件检错且 checksum 关闭。
    - 解决：host 与 slave 都启用 `*_CHECKSUM`。

11. **`esp_hosted_*_set_config()` 编译告警**
    - 根因：函数标了 `warn_unused_result`。
    - 解决：检查返回的 `esp_hosted_transport_err_t`。

12. **自定义数据回调签名不匹配（v2.12.4）**
    - 根因：回调增加了 `local_context` 参数。
    - 解决：host 与 slave 都升到 >= 2.12.4，使用新签名（4 参回调 + context 指针）。

## C. 构建 / 内存

13. **UART slave 在 IDF v5.5 IRAM 不足**
    - 根因：ringbuf 占 IRAM。
    - 解决：启用 `CONFIG_RINGBUF_PLACE_FUNCTIONS_INTO_FLASH=y`（或 menuconfig ESP Ringbuf 对应项）。

14. **SDIO Streaming 模式 host 内存吃紧**
    - 根因：双缓冲 `2 * Tx队列 * 1536`。
    - 解决：切 Packet 模式，或调小 slave `SDIO Tx queue size`（默认 20，25 以上吞吐基本饱和）。

## D. 蓝牙 / OpenThread / Zigbee

15. **同时启用 Hosted HCI 和标准 HCI over UART**
    - 根因：两者互斥。
    - 解决：只保留其一；Hosted HCI 复用主传输无需额外 GPIO，标准 HCI 需专用 UART（2/4 GPIO）。

16. **ESP32 协处理器 BT 版本不匹配**
    - 根因：ESP32 仅支持 BT v4.2。
    - 解决：host BT 栈配为 v4.2；其他协处理器仅支持 BLE。

17. **OpenThread / Zigbee 数据不通**
    - 根因：误用 ESP-Hosted 主传输；当前 OpenThread/Zigbee 需专用 UART 通道。
    - 解决：协处理器配为 RCP，OpenThread/Zigbee 走专用 UART。

18. **Border Router 单射频丢包**
    - 根因：Wi-Fi 与 802.15.4 共用一个射频，软件共存冲突。
    - 解决：用两个协处理器（Wi-Fi + RCP 分离）。

## E. OTA / 升级

19. **slave OTA 中途失败**
    - 根因：链路不稳 / 校验失败 / 镜像不完整。
    - 解决：稳定链路、启用 checksum、按顺序写完整字节流（`begin → write(连续) → end → activate`）。

20. **升级后版本未变**
    - 根因：未 `activate`（不会重启）。
    - 解决：调用 `esp_hosted_slave_ota_activate()` 触发激活重启。

21. **使用了 deprecated 的 `esp_hosted_slave_ota(url)`**
    - 解决：改用 `esp_hosted_slave_ota_begin/write/end/activate` 三段式，自行取镜像。

## F. 供电 / 硬件通用

22. **无法烧录 slave**
    - 根因：host 拉低信号。
    - 解决：先 `esptool.py -p <host_port> --before default_reset --after no_reset run` 把 host 置 bootloader，再烧 slave。

23. **传输不稳随走线恶化**
    - 解决：SPI 跳线 <= 10 cm；SDIO PCB < 5 cm、等长；多接 GND；先低时钟再阶梯升；优先 IO_MUX 引脚。

24. **Quad SPI 用跳线**
    - 根因：信号完整性要求 PCB。
    - 解决：Quad/HD 必须用 PCB；经典 ESP32 不支持 1-bit/Dual/Quad SPI。

## 参考

- `docs/troubleshooting.md`、`docs/design_consideration.md`
- `docs/spi_full_duplex.md`、`docs/spi_half_duplex.md`、`docs/sdio.md`、`docs/uart.md`
- `docs/bluetooth_design.md`、`docs/openthread_zigbee.md`、`docs/migration_guide.md`
- SKILL.md "Critical Pitfalls (Must Read)" 一节有 WRONG/CORRECT 代码对照
