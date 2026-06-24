# ESP-Hosted-MCU 状态机与传输序列

> 仓库未提供形式化状态机；本文根据 `docs/`（bring-up、events、recovery）与 `examples/host_hosted_events` 例程归纳出 host 与 co-processor 之间的实际行为状态与转换。所有事件 ID/枚举来自 `host/esp_hosted_event.h`。

## 1. Host 侧应用 / 传输状态

```
                    esp_hosted_init()
                          │
                          ▼
              esp_hosted_connect_to_slave()
                  │            ▲
   Reset slave   │            │ re-init (recovery)
   (GPIO)        ▼            │
        ┌──────────────────┐  │
        │  CONNECTING      │  │
        │  "Attempt        │  │
        │   connection     │  │
        │   retry[N]"      │  │
        └────────┬─────────┘  │
                 │ 收到协处理器 INIT
                 ▼
        ┌──────────────────┐
        │  TRANSPORT_UP    │◄──── ESP_HOSTED_EVENT_TRANSPORT_UP
        │  (sem give)      │      (应用放行，开始 Wi-Fi/BT)
        └────────┬─────────┘
                 │
        ┌────────▼─────────┐
        │  RUNNING         │
        │  Wi-Fi/BT 正常   │
        │  HEARTBEAT 周期  │
        └────────┬─────────┘
                 │ TRANSPORT_FAILURE / TRANSPORT_DOWN
                 │ / 意外 CP_INIT / 心跳超时
                 ▼
        ┌──────────────────┐
        │  RECOVERY        │
        │  deinit Wi-Fi    │
        │  esp_hosted_     │
        │    deinit()      │
        │  (回到 init)     │──┐
        └──────────────────┘  │
                              │
                  re-init ────┘
```

### 关键事件（host 监听 `ESP_HOSTED_EVENT`）
| 事件 ID | 含义 | 典型处理 |
|---|---|---|
| `ESP_HOSTED_EVENT_CP_INIT` | 协处理器已启动（含 reset reason） | 首次=正常；运行期再次出现=slave 重启→触发恢复 |
| `ESP_HOSTED_EVENT_TRANSPORT_UP` | 传输就绪 | 放行主流程（give 信号量），开始 Wi-Fi/BT |
| `ESP_HOSTED_EVENT_TRANSPORT_DOWN` | 传输掉线 | 告知数据线程中止 |
| `ESP_HOSTED_EVENT_TRANSPORT_FAILURE` | 传输故障 | 置恢复标志，deinit→重新 init/connect |
| `ESP_HOSTED_EVENT_CP_HEARTBEAT` | 心跳（递增计数） | 重置心跳超时定时器；检查计数连续性 |
| `ESP_HOSTED_EVENT_MEM_MONITOR` | 内存监控报告 | 按需记录/告警 |

## 2. Co-processor（slave）启动序列（来自 slave 日志）

```
上电 / 被 host Reset
   │
   ▼
fg_mcu_slave: ESP-Hosted-MCU Slave FW version :: X.Y.Z
fg_mcu_slave: Transport used :: SPI | SDIO | UART only
   │
   ▼
打印 capabilities（如 0xe8 / 0x88）与支持特性
   │   - WLAN over <transport>
   │   - BT/BLE（HCI Over <transport>，BLE only / Classic+BLE for ESP32）
   ▼
等待 host 的 RPC（控制 ESP_SERIAL_IF）与数据（ESP_STA_IF/ESP_AP_IF/ESP_HCI_IF/...）
```

## 3. SPI 全双工事务握手（来自 `docs/spi_full_duplex.md`）

每笔 SPI 事务固定 1600 字节 TX/RX 缓冲，规则：

1. 协处理器准备好收发 → 拉高 **Handshake**（若 TX 有数据同时拉高 **Data Ready**）。
2. host 收到 Handshake 中断：仅在 Data Ready 高、或 host 有数据要发时才发起事务；否则忽略中断。
3. 事务期间在 MOSI/MISO 同时交换 TX/RX 缓冲（全双工）。
4. 协处理器在事务后拉低 Handshake（若有 TX 数据同时拉低 Data Ready）。
5. 双方按 payload header（`if_type`/`len`/`offset`/`checksum`）处理收到的缓冲。

> host 必须在 Handshake 高时才发起事务；协处理器总是准备好接收并立即排队下一笔。

## 4. 帧头与接口复用（来自 README §7）

每帧起始为 ESP-Hosted header，`if_type` 决定载荷类别：

| if_type | 值 | 载荷 |
|---|---|---|
| `ESP_STA_IF` | 1 | Station（Wi-Fi）数据 |
| `ESP_AP_IF` | 2 | SoftAP（Wi-Fi）数据 |
| `ESP_SERIAL_IF` | 3 | 控制 / RPC |
| `ESP_HCI_IF` | 4 | 蓝牙 Hosted HCI |
| `ESP_PRIV_IF` | 5 | 私有 host↔slave |
| `ESP_TEST_IF` | 6 | 传输吞吐测试 |

> 仅 RPC（控制，`ESP_SERIAL_IF`）走 protobuf 序列化；数据帧不序列化。

## 5. 协处理器 OTA 状态流（来自 `examples/host_performs_slave_ota`）

```
host 取镜像 ──> esp_hosted_slave_ota_begin()
                        │
                        ▼
              循环 esp_hosted_slave_ota_write(chunk)
                        │
                        ▼
              esp_hosted_slave_ota_end()
                        │
                        ▼
              esp_hosted_slave_ota_activate()
                        │  (激活并重启协处理器)
                        ▼
              host 收到 ESP_HOSTED_EVENT_CP_INIT（升级后版本）
```

OTA 状态枚举（`host/api/include/esp_hosted_ota.h`）：
`ESP_HOSTED_SLAVE_OTA_NOT_STARTED` → `IN_PROGRESS` → `COMPLETED` → `ACTIVATED`（或 `NOT_REQUIRED` / `FAILED`）。

## 6. host 省电唤醒流（来自 `docs/feature_host_power_save.md`）

```
host 运行 ──> esp_hosted_power_save_start(DEEP_SLEEP)  (不返回)
                        │
                        ▼
              host 进入 deep sleep，网络由 slave 维持
                        │
                        ▼
              slave 需要时通过 Host Wakeup GPIO 唤醒 host
                        │
                        ▼
              host 重启 ──> app_main 中 esp_hosted_woke_from_power_save() == 1
                        │
                        ▼
              恢复网络/会话状态，继续运行
```

## 参考

- `host/esp_hosted_event.h`（事件 ID 与参数结构）
- `host/api/include/esp_hosted_ota.h`、`host/api/include/esp_hosted_power_save.h`
- `examples/host_hosted_events/main/main.c`（恢复循环实现）
- `docs/spi_full_duplex.md`（SPI 握手细节）、`docs/sdio.md`（Stream/Packet 模式）
- README §7（帧头与接口类型）
