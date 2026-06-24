# NimBLE 仓库示例索引

> 路径相对于仓库根 `D:/esp-skill/espressif-repos/esp-nimble/apps/`。所有示例均为 Mynewt 风格（`sysinit()` + `os_eventq_run`）。移植到 ESP-IDF 时需替换为 `nimble_port_init` + `nimble_port_freertos_init`（见 `recipes/host_init.md`）。

| 示例路径 | 说明 |
|---|---|
| `apps/bleprph` | 基础外设（Peripheral）：完整 GAP 事件处理（CONNECT/DISCONNECT/ENC_CHANGE/CONN_UPDATE/SUBSCRIBE/MTU/REPEAT_PAIRING），含 `gatt_svr.c`、PHY 支持（`src/phy.c`：按键切换 1M/2M/Coded S2/S8 + LED 指示 + `ble_gap_read_le_phy`）；最常用的外设参考。 |
| `apps/peripheral` | 精简可连接外设：含 scan response、conn params 更新回调。 |
| `apps/blehr` | 心率传感器（HRS）：GATT 服务表 + NOTIFY 特征 + `ble_gatts_notify_custom` + 定时器推送。学习 notify 的首选。 |
| `apps/blecsc` | 骑行速度与步频传感器（CSC）：GATT 自定义服务。 |
| `apps/blecent` | 中心：扫描 → 取消扫描 → 连接 → `peer_disc_all` 服务发现 → `peer_chr_find_uuid` / `peer_dsc_find_uuid` → 读写订阅。学习 central 完整链路的首选。 |
| `apps/central` | 精简中心示例。 |
| `apps/scanner` | 观察者：NRPA 生成 + 被动扫描 + `ble_hs_adv_parse_fields` 全字段解析 + 自动重启。 |
| `apps/advertiser` | 广播者：NRPA + 不可连接广播（non-connectable）。Beacon 类应用参考。 |
| `apps/ext_advertiser` | 扩展广播综合示例：non-connectable / scannable / scannable-legacy / legacy-duration / max-events / periodic 六类实例配置。扩展广播首选。 |
| `apps/blehci` | Controller-only 应用：通过 HCI over UART 暴露控制器，供外部 Host（如 BlueZ）使用。 |
| `apps/dtm` | 直接测试模式（DTM）：射频测试，含 `parse.c` 命令解析。 |
| `apps/blemesh` | Mesh 节点基础：Health server fault 回调 + provisioning + 元素/模型定义。 |
| `apps/blemesh_light` | Mesh 灯：Generic OnOff server + WS2812 LED 驱动（`light_model.c`）。 |
| `apps/blemesh_models_example_1` | Mesh 自定义模型示例 1。 |
| `apps/blemesh_models_example_2` | Mesh 综合：含设备组合（`device_composition.c`）、状态绑定（`state_binding.c`）、发布者（`publisher.c`）、存储（`storage.c`）、过渡（`transition.c`）。 |
| `apps/blemesh_shell` | Mesh shell 调试应用。 |
| `apps/mesh_badge` | Mesh 徽章应用（含 `gatt_svr.c`、`mesh.c`、reel board）。 |
| `apps/btshell` | 命令行 BLE 工具：`cmd_gatt.c`、`cmd_l2cap.c` 等，覆盖大多数 NimBLE 功能的命令接口。功能探索参考。 |
| `apps/bttester` | 蓝牙测试工具（BTP 协议）：GAP/GATT/L2CAP/Mesh 测试，含 UART/RTT pipe。 |
| `apps/blestress` | 压力测试：`tx_stress.c`、`rx_stress.c`、`stress_gatt.c`。 |

## 移植平台示例

位于 `porting/examples/`，演示 NimBLE 在不同 OS 上的移植：

| 路径 | 说明 |
|---|---|
| `porting/examples/linux` | Linux 移植示例（`include/` 平台头）。 |
| `porting/examples/linux_blemesh` | Linux Mesh 移植示例。 |
| `porting/examples/nuttx` | NuttOS 移植示例。 |
| `porting/examples/dummy` | 占位/模板移植。 |

## NPL 移植端口

位于 `porting/npl/`，每种 RTOS 一个端口：

| 路径 | 说明 |
|---|---|
| `porting/npl/freertos` | FreeRTOS 端口（ESP32/ESP32-C3/ESP32-S3/ESP32-C2 使用）。 |
| `porting/npl/esp-idf` | ESP-IDF 专用端口（ESP32-H4 等新芯片使用，见其 `README.md`）。 |
| `porting/npl/mynewt` | Mynewt 原生端口。 |
| `porting/npl/nuttx` | NuttOS 端口。 |
| `porting/npl/riot` | RIOT 端口。 |
| `porting/npl/linux` | Linux 端口。 |
| `porting/npl/dummy` | 占位端口。 |

## Targets（构建目标）

| 路径 | 说明 |
|---|---|
| `targets/dialog_cmac` | Dialog/Renesas DA1469x (cmac) 控制器目标。 |
| `targets/nordic_pca10056-blehci-usb` | Nordic nRF52840 HCI-over-USB 目标。 |
| `targets/nordic_pca10095_net-blehci` | Nordic nRF5340 网络核 HCI 目标。 |
| `targets/unittest` | 单元测试目标。 |
