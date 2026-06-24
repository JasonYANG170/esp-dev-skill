---
name: esp-idf-skill
description: >-
  AI Skill for Espressif ESP-IDF firmware development. Used when users need to
  create, modify, build, or debug ESP-IDF (Espressif IoT Development Framework)
  projects on ESP32 / ESP32-S / ESP32-C / ESP32-H / ESP32-P SoCs, covering Wi-Fi,
  BLE, peripherals, NVS storage, OTA updates, partition tables, FreeRTOS tasks,
  and the idf.py / CMake build system.
  Trigger words: "ESP-IDF", "esp-idf", "Espressif", "ESP32", "ESP32-S3", "ESP32-C3", "ESP32-C6", "ESP32-H2", "idf.py", "app_main", "FreeRTOS", "乐鑫", "安信可", "Wi-Fi 配网", "OTA", "NVS", "外设", "分区表"
tags:
  - embedded
  - esp-idf
  - espressif
  - esp32
  - esp32s3
  - esp32c3
  - esp32c6
  - wifi
  - bluetooth
  - freertos
  - firmware
  - IoT
license: Apache-2.0
compatibility: >-
  Targets ESP32, ESP32-S2, ESP32-S3, ESP32-C2/C3/C5/C6/C61, ESP32-H2/H4/H21,
  ESP32-P4; build via idf.py (CMake + Ninja) with the ESP-IDF toolchain on
  Windows/Linux/macOS.
metadata:
  author: Community
  version: "1.1.0"
---

# esp-idf-skill

面向 Espressif 官方 IoT 开发框架 ESP-IDF 的 AI 技能。提供场景化 recipe、真实 API 参考、Kconfig 配置速查与常见陷阱，覆盖 ESP32 全系列 SoC 的 Wi-Fi/BLE、外设驱动、NVS/Flash 存储、OTA 升级、分区表、FreeRTOS 任务以及 idf.py/CMake 构建系统。所有函数签名、结构体、宏、配置项与代码片段均取自 ESP-IDF 仓库的真实文档与源码。

## Core Principles

1. **绝不臆造 API** — 写代码前先查 `resources/`；在仓库 `components/*/include` 中找不到的符号，即视为不存在。
2. **入口函数恒为 `app_main`** — ESP-IDF 启动后由系统任务调用 `void app_main(void)`，用户无需手写 `main()`；`app_main` 返回后该任务被自动删除，故长驻逻辑需自建 `while(1)` 或 `xTaskCreate`。
3. **构建基于 idf.py + CMake + Ninja** — 每个组件用 `idf_component_register(...)` 注册；项目根 `CMakeLists.txt` 必须按固定顺序 `cmake_minimum_required` → `include($ENV{IDF_PATH}/tools/cmake/project.cmake)` → `project(name)`。
4. **目标芯片用 `idf.py set-target <chip>` 指定** — 不同 SoC (esp32/esp32s3/esp32c3/...) 生成不同的 `sdkconfig` 与链接脚本；切换目标会清除既有构建。
5. **NVS 是默认持久化层** — Wi-Fi 凭据、校准数据、应用配置均存于 NVS 分区；`app_main` 开头应初始化 NVS，并在 `ESP_ERR_NVS_NO_FREE_PAGES` / `ESP_ERR_NVS_NEW_VERSION_FOUND` 时 `nvs_flash_erase()` 后重试。
6. **事件循环统一驱动 Wi-Fi/IP/蓝牙** — 先 `esp_netif_init()`、`esp_event_loop_create_default()`，再用 `esp_event_handler_instance_register(WIFI_EVENT/IP_EVENT, ...)` 注册回调；连接成功由 `IP_EVENT_STA_GOT_IP` 标志。
7. **每个返回 `esp_err_t` 的调用都应检查** — 优先用 `ESP_ERROR_CHECK()`（失败时 abort 并打印 backtrace）或显式判断后用 `esp_err_to_name()` 输出。
8. **分区表决定 Flash 布局** — `CONFIG_PARTITION_TABLE_*` 选择内置或自定义 CSV；OTA 应用需至少 factory + ota_0/ota_1 + otadata 分区。
9. **日志统一用 ESP_LOG** — `ESP_LOGI/E/W(tag, fmt, ...)`，等级受 `CONFIG_LOG_DEFAULT_LEVEL` 控制；崩溃由 `idf.py monitor` 自动解码 panic backtrace。
10. **外设驱动走 v5.x 新 HAL（`esp_driver_*` 组件）** — GPIO/UART/I2C/SPI/LEDC/GPTimer 等均为句柄式 API（如 `gptimer_new_timer`、`i2c_new_master_bus`），旧版 `gpio_matrix_out` 等底层函数非首选。
11. **FreeRTOS 是唯一 RTOS** — 任务、队列、信号量、事件组来自 `freertos/FreeRTOS.h`；延时用 `vTaskDelay(pdMS_TO_TICKS(ms))`，禁止在 ISR 中调用阻塞 API。
12. **Flash 写入需先擦除** — SPI Flash 按 4KB 扇区擦除；流式存储优先 NVS/FATFS/LittleFS/wear_levelling，而非裸 `spi_flash_*`。
13. **蓝牙启动有固定顺序** — Bluedroid/NimBLE 均需：NVS → `esp_bt_controller_init` → `esp_bt_controller_enable(ESP_BT_MODE_BLE/CLASSIC_BT/BTDM)` → `esp_bluedroid_init_with_cfg` → `esp_bluedroid_enable` → 注���回调 → 注册 app。BLE 与 Classic BT 共用控制器，模式互斥；纯 BLE 应 `esp_bt_controller_mem_release(ESP_BT_MODE_CLASSIC_BT)` 腾内存。Classic BT（SPP/A2DP/HFP）仅 esp32 双模支持，需 `CONFIG_BT_CLASSIC_ENABLED=y`。
14. **Mesh/以太网先建 netif 再启协议栈** — Wi-Fi Mesh 需 `esp_netif_create_default_wifi_mesh_netifs` 后 `esp_mesh_start`；以太网需 `esp_eth_new_netif_glue` + `esp_netif_attach` 把驱动挂到 TCP/IP 栈，再 `esp_eth_start`。`esp_mesh_start` 后禁止调任何 `esp_wifi_*` API（mesh 自组织接管）。

## When to Use

**适用：**
- 新建 ESP-IDF 工程或基于 `examples/` 修改（hello_world/blink/wifi/getting_started 等）
- 配置外设驱动：GPIO、UART、I2C（主/从）、SPI 主机、LEDC PWM、GPTimer、ADC
- Wi-Fi STA/SoftAP/扫描/配网、获取 IP 事件
- Wi-Fi Mesh（ESP-WIFI-MESH）组网、内部以太网（RMII/RGMII + PHY）
- BLE（Bluedroid）：GATT Server/Client、广播、扫描、notify
- 经典蓝牙（仅 esp32）：SPP 串口透传、A2DP 音频、AVRCP/HFP
- ESP-BLE-MESH 蓝牙网格：节点、配网、Generic OnOff 等模型
- NVS 键值存储、分区表查询、SPI Flash 读写
- OTA 升级（simple/advanced/native_https_ota）、双分区切换
- FreeRTOS 任务、队列、事件组、定时器回调
- 深度睡眠与唤醒源、低功耗
- idf.py 构建配置、menuconfig、自定义 component

**不适用：**
- ESP8266 / ESP8285（使用独立的 ESP8266_RTOS_SDK，非本框架）
- Arduino-ESP32 框架（API 与组件名不同）
- 纯 PCB 原理图/硬件设计（应查 Espressif 硬件手册）
- ESP-ADF / ESP-ROUTERS / ESP-MATTER 等上层框架（有各自仓库）

---

## Scenario Quick Reference (Recipes)

当用户意图匹配下列场景时，**先读对应 recipe**，其中包含完整调用链、分步说明、可复制代码与常见错误表。

### 工程与构建

| recipe | 场景 |
|---|---|
| `recipes/new_project.md` | 新建 ESP-IDF 工程：目录结构、CMakeLists、set-target、build/flash/monitor |
| `recipes/freertos_task.md` | FreeRTOS 任务与队列：创建任务、任务通知、队列通信 |

### 外设驱动

| recipe | 场景 |
|---|---|
| `recipes/gpio_control.md` | GPIO 输入/输出、中断（`gpio_install_isr_service` + `gpio_isr_handler_add`） |
| `recipes/uart_comm.md` | UART 收发：`uart_driver_install`、`uart_read_bytes`、事件驱动 |
| `recipes/i2c_master.md` | I2C 主机：新总线 API `i2c_new_master_bus` + 设备读写 |
| `recipes/spi_master.md` | SPI 主机：`spi_bus_initialize`/`spi_bus_add_device`/`spi_device_transmit` |
| `recipes/ledc_pwm.md` | LEDC PWM 输出与调光 |
| `recipes/gptimer.md` | GPTimer 通用定时器与告警回调 |

### 网络 / 无线

| recipe | 场景 |
|---|---|
| `recipes/wifi_sta.md` | Wi-Fi STA 连接：事件循环、`esp_wifi_start`、获取 IP |
| `recipes/softap.md` | Wi-Fi SoftAP 热点 |
| `recipes/wifi_mesh.md` | ESP-WIFI-MESH：`esp_mesh_init`/`set_config`/`start`，P2P 收发，根节点选举与事件 |
| `recipes/ethernet.md` | 内部以太网 MAC+PHY：`esp_eth_mac_new_esp32`/`phy_new_generic`/`driver_install`/`start`，netif 挂载 |

### 蓝牙（Bluedroid）

> 经典蓝牙（SPP/A2DP）仅 **esp32** 双模支持；BLE 与 BLE Mesh 在支持 BT 的 SoC 上通用。Bluedroid Host 用 `esp_ble_*` 系列 API（NimBLE Host 用 `ble_*`，API 不同，本组仅适用 `CONFIG_BT_BLUEDROID_ENABLED=y`）。

| recipe | 场景 |
|---|---|
| `recipes/ble_peripheral.md` | BLE GATT Server：属性表建服务（`esp_ble_gatts_create_attr_tab`）、广播、notify/indicate |
| `recipes/ble_central.md` | BLE GATT Client：扫描（`esp_ble_gap_start_scanning`）、连接、服务/特征值发现、读写、订阅 notify |
| `recipes/classic_bt_spp.md` | 经典蓝牙 SPP 串口透传：`esp_spp_enhanced_init`/`start_srv`/`write`，CB 与 VFS 模式 |
| `recipes/classic_bt_a2dp.md` | 经典蓝牙 A2DP Sink 音频接收 + AVRCP CT 音量控制：`esp_a2d_sink_init`、PCM 数据回调 |
| `recipes/ble_mesh.md` | ESP-BLE-MESH 节点：Composition Data（Config Server + Generic OnOff）、配网承载、SET/STATUS 处理与 publish |

### 存储 / 系统

| recipe | 场景 |
|---|---|
| `recipes/nvs_storage.md` | NVS 读写：`nvs_open`/`nvs_set_i32`/`nvs_get_*`/blob |
| `recipes/partition_table.md` | 自定义分区表、`esp_partition_find_first` 查询 |
| `recipes/ota_update.md` | OTA 升级：`esp_https_ota` / `esp_ota_begin/write/end` 双分区 |
| `recipes/deep_sleep.md` | 深度睡眠与 GPIO/定时器唤醒 |

---

## 目标芯片支持矩阵（基于 ESP-IDF 仓库 `docs/en/get-started/index.rst`）

| 目标 | 内核 | Wi-Fi | BT/BLE | 802.15.4 | 备注 |
|---|---|---|---|---|---|
| esp32 | 双核 Xtensa LX6 | 2.4G b/g/n | BT + BLE | — | 经典双核 |
| esp32s2 | 单核 Xtensa LX7 | 2.4G | — | — | 含 USB OTG |
| esp32s3 | 双核 Xtensa LX7 | 2.4G | BLE | — | USB OTG + USB Serial/JTAG |
| esp32c3 | 单核 RISC-V | 2.4G | BLE | — | 高性价比 |
| esp32c2 (esp8684) | 单核 RISC-V | 2.4G | BLE | — | 简单 IoT |
| esp32c5 | 单核 RISC-V | Wi-Fi 6 双频 | BLE | Thread/Zigbee | 2.4G+5G |
| esp32c6 | 单核 RISC-V | Wi-Fi 6 2.4G | BLE | Thread/Zigbee | |
| esp32c61 | 单核 RISC-V | Wi-Fi 6 2.4G | BLE | — | |
| esp32h2 | 单核 RISC-V | — | BLE | Thread/Zigbee | 无 Wi-Fi |
| esp32p4 | 双核 RISC-V | — (外挂) | — | — | MIPI/USB/SDIO/Ethernet |

> 运行 `idf.py set-target`（无参数）可列出当前 ESP-IDF 版本支持的全部目标。

## 关键 idf.py 命令

| 命令 | 作用 |
|---|---|
| `idf.py set-target <chip>` | 指定目标芯片，初始化 sdkconfig |
| `idf.py menuconfig` | 文本菜单配置（生成/编辑 sdkconfig） |
| `idf.py build` | 编译 app + bootloader + 生成分区表 |
| `idf.py -p PORT flash` | 烧录 app+bootloader+分区表 |
| `idf.py -p PORT monitor` | 串口监控（Ctrl+] 退出，自动解码 panic） |
| `idf.py -p PORT flash monitor` | 编译+烧录+监控一条命令 |
| `idf.py app` / `idf.py app-flash` | 仅编译/仅烧录 app |
| `idf.py erase-flash` | 整片擦除（改分区表或 OTA 后常用） |
| `idf.py fullclean` | 删除全部构建产物 |
| `idf.py --list-targets` | 列出支持的芯片目标 |

> 路径中含空格不被支持；Windows 串口形如 `COM3`，Linux `/dev/ttyUSB0`，macOS `/dev/cu.usbserial-*`。

---

## Critical Pitfalls (Must Read)

以下是最常见错误，违反任何一条都会导致固件不工作。

### 1. app_main 返回后任务被删除

```c
// ❌ WRONG — app_main 直接返回，长驻逻辑丢失
void app_main(void) {
    init_hardware();
    // 返回后系统删除 main 任务，程序"卡住"
}

// ✅ CORRECT — 自持循环或创建任务
void app_main(void) {
    init_hardware();
    xTaskCreate(my_task, "my_task", 4096, NULL, 5, NULL);
    while (1) {
        vTaskDelay(pdMS_TO_TICKS(1000));
    }
}
```

### 2. 组件未在 CMakeLists 中注册

```cmake
# ❌ WRONG — main 没注册源文件，编译报“undefined reference”
# main/CMakeLists.txt 为空

# ✅ CORRECT
# main/CMakeLists.txt
idf_component_register(SRCS "main.c"
                       INCLUDE_DIRS "."
                       PRIV_REQUIRES nvs_flash driver)
```

### 3. NVS 初始化未处理升级错误

```c
// ❌ WRONG — 不处理 NVS 版本/页错误，后续 nvs_open 失败
esp_err_t err = nvs_flash_init();
// 直接继续...

// ✅ CORRECT
esp_err_t err = nvs_flash_init();
if (err == ESP_ERR_NVS_NO_FREE_PAGES || err == ESP_ERR_NVS_NEW_VERSION_FOUND) {
    ESP_ERROR_CHECK(nvs_flash_erase());
    err = nvs_flash_init();
}
ESP_ERROR_CHECK(err);
```

### 4. Wi-Fi 缺少 netif / 事件循环

```c
// ❌ WRONG — 未建 netif 与默认事件循环，esp_wifi_start 行为异常
esp_wifi_init(&cfg);
esp_wifi_start();

// ✅ CORRECT — 先建网络接口与事件循环
ESP_ERROR_CHECK(esp_netif_init());
ESP_ERROR_CHECK(esp_event_loop_create_default());
esp_netif_create_default_wifi_sta();
wifi_init_config_t cfg = WIFI_INIT_CONFIG_DEFAULT();
ESP_ERROR_CHECK(esp_wifi_init(&cfg));
// 再注册 WIFI_EVENT / IP_EVENT 处理器，最后 start
```

### 5. GPIO ISR 服务未安装

```c
// ❌ WRONG — 直接加 ISR handler，返回 ESP_ERR_INVALID_STATE
gpio_isr_handler_add(BUTTON_GPIO, my_isr, NULL);

// ✅ CORRECT — 先安装 ISR 服务
gpio_config_t io = {/* ...interrupt...*/};
gpio_config(&io);
gpio_install_isr_service(0);                 // 一次性
gpio_isr_handler_add(BUTTON_GPIO, my_isr, NULL);
```

### 6. UART 未安装驱动就拿不到数据

```c
// ❌ WRONG — 仅 uart_param_config 就读
uart_param_config(UART_NUM_1, &cfg);
int n = uart_read_bytes(UART_NUM_1, buf, 1, 0); // 拿不到

// ✅ CORRECT — 必须 install 驱动
uart_param_config(UART_NUM_1, &cfg);
uart_set_pin(UART_NUM_1, TX, RX, RTS, CTS);
uart_driver_install(UART_NUM_1, 1024, 1024, 0, NULL, 0);
int n = uart_read_bytes(UART_NUM_1, buf, sizeof(buf), pdMS_TO_TICKS(100));
```

### 7. I2C 用了已废弃的 v4 驱动 API

```c
// ❌ WRONG — v5.x 下 i2c_param_config/i2c_master_write 已弃用
i2c_param_config(I2C_NUM_0, &conf);
i2c_driver_install(I2C_NUM_0, I2C_MODE_MASTER, 0, 0, 0);

// ✅ CORRECT — 新总线式句柄 API
i2c_master_bus_config_t bus = {
    .i2c_port = I2C_NUM_0,
    .sda_io_num = 8, .scl_io_num = 9,
    .clk_source = I2C_CLK_SRC_DEFAULT,
    .glitch_ignore_cnt = 7,
    .flags.enable_internal_pullup = true,
};
i2c_master_bus_handle_t bus_handle;
i2c_new_master_bus(&bus, &bus_handle);
i2c_device_config_t dev = { .dev_addr_length = I2C_ADDR_BIT_LEN_7,
                            .device_address = 0x68,
                            .scl_speed_hz = 100000 };
i2c_master_dev_handle_t dev_handle;
i2c_master_bus_add_device(bus_handle, &dev, &dev_handle);
i2c_master_transmit_receive(dev_handle, &reg, 1, buf, len, -1);
```

### 8. esp_err_t 返回值被忽略

```c
// ❌ WRONG — 静默失败，排查困难
nvs_set_i32(handle, "count", 1);

// ✅ CORRECT — 检查并打印可读错误名
esp_err_t err = nvs_set_i32(handle, "count", 1);
if (err != ESP_OK) {
    ESP_LOGE(TAG, "nvs_set_i32 failed: %s", esp_err_to_name(err));
}
// 或对不可恢复错误：ESP_ERROR_CHECK(nvs_set_i32(...));
```

### 9. OTA 缺少 otadata / 双分区

```csv
# ❌ WRONG — 分区表只有 factory，OTA 报“no next partition”
# Name,   Type, SubType, Offset, Size
nvs,      data, nvs,     ,        0x4000
factory,  app,  factory, ,        1M

# ✅ CORRECT — factory + ota_0 + ota_1 + otadata
# Name,   Type, SubType, Offset, Size
nvs,      data, nvs,     ,        0x4000
otadata,  data, ota,     ,        0x2000
factory,  app,  factory, ,        1M
ota_0,    app,  ota_0,   ,        1M
ota_1,    app,  ota_1,   ,        1M
```

### 10. Flash 写入前未擦除 / 直接覆盖写

```c
// ❌ WRONG — 直接覆盖写，旧数据残留、写失败
esp_partition_write(part, offset, buf, 64); // 写前必须先擦

// ✅ CORRECT — 按扇区（4KB）擦除后再写
esp_partition_erase_range(part, offset_aligned, 4096);  // offset/size 须 4KB 对齐
esp_partition_write(part, offset, buf, 64);
// 应用层优先用 NVS / wear_levelling / FATFS，避免裸分区操作
```

### 11. ISR 中调用阻塞 / FreeRTOS 非 ISR 版 API

```c
// ❌ WRONG — ISR 内调用 xQueueSend、vTaskDelay
void IRAM_ATTR gpio_isr(void *arg) {
    vTaskDelay(1);            // 禁止
    xQueueSend(q, &v, 0);     // 应使用 FromISR 版
}

// ✅ CORRECT
void IRAM_ATTR gpio_isr(void *arg) {
    BaseType_t hpw = pdFALSE;
    xQueueSendFromISR(q, &v, &hpw);
    portYIELD_FROM_ISR(hpw);
}
```

### 12. set-target 与 menuconfig 顺序 / 旧构建残留

```bash
# ❌ WRONG — 未先 set-target 直接 build，目标不匹配
idf.py build

# ✅ CORRECT
idf.py set-target esp32s3
idf.py menuconfig
idf.py build
idf.py -p COM3 flash monitor
# 切换目标后用 idf.py fullclean 再 set-target
```

### 13. 日志等级被 CONFIG 截断

```c
// ❌ WRONG — 看不到 ESP_LOGD，以为代码没执行
ESP_LOGD(TAG, "debug here"); // 默认 CONFIG_LOG_DEFAULT_LEVEL=INFO 时被编译裁剪

// ✅ CORRECT — 调临时提级或改 Kconfig
esp_log_level_set(TAG, ESP_LOG_DEBUG);   // 运行时提级该 tag
// 或 menuconfig → Component config → Log output → Default log verbosity = Debug
```

---

## Execution Workflow

| Step | Name | Description |
|------|------|-------------|
| 1 | Plan | 明确目标芯片、所需外设/网络/存储功能，确认是否需要 OTA/NVS/分区 |
| 2 | Recipe | 在 `recipes/` 中匹配最接近场景，沿其调用链实现 |
| 3 | Query | recipe 未覆盖的 API，查 `resources/api_reference.md` 与 `resources/config_reference.md` |
| 4 | Validate | 核对函数签名、头文件 include、组件依赖 (PRIV_REQUIRES)、Kconfig 选项 |
| 5 | Confirm | 向用户呈现：组件依赖、引脚分配、初始化顺序、`app_main` 结构、分区表 |
| 6 | Execute | **新工程**：复制最接近的 `examples/` 子目录到工作区再改；**已有工程**：原地编辑 |
| 7 | Build | `idf.py set-target <chip>` → `idf.py build`，留意 warning 与未定义符号 |
| 8 | Flash | `idf.py -p PORT flash monitor`；首次或改分区表后先 `idf.py erase-flash` |
| 9 | Debug | 用 `idf.py monitor` 查看 ESP_LOG 输出与 panic backtrace；必要时 JTAG |

### Step 6 Detail — 工程创建策略

**目标目录尚无工程（首次创建）：**

1. 按 user 需求在 ESP-IDF `examples/` 中选最接近的样例：
   - 起步模板 → `examples/get-started/hello_world`、`examples/get-started/blink`
   - GPIO/UART/I2C/SPI/LEDC/GPTimer → `examples/peripherals/<periph>/...`
   - Wi-Fi STA/SoftAP/扫描 → `examples/wifi/getting_started/station`、`examples/wifi/getting_started/softAP`
   - NVS → `examples/storage/nvs/nvs_rw_value`
   - 分区查询 → `examples/storage/partition_api/partition_find`
   - OTA → `examples/system/ota/simple_ota_example`、`native_ota_example`、`advanced_https_ota`
   - 深度睡眠 → `examples/system/deep_sleep`
2. 将整个样例目录复制到用户工程目录，保留 `CMakeLists.txt`、`main/`、`idf_component.yml`、`partitions/` 等结构。
3. 在副本上修改：改组件名、调整 `idf_component_register`、改引脚/SSID、增删 Kconfig.projbuild。
4. 向用户说明复制来源与修改点。

**目标目录已存在工程：** 原地编辑，未经允许不覆盖既有文件。

---

## Failure Strategies

| Situation | Action |
|---|---|
| API 在 `resources/` 与组件头文件中均找不到 | 立即停止，告知用户该符号可能不存在或版本不符 |
| `undefined reference` 链接错误 | 检查 `main/CMakeLists.txt` 的 `PRIV_REQUIRES`/`REQUIRES` 是否含对应组件 |
| `idf.py build` 报 target 未设 | 先 `idf.py set-target <chip>`；切目标前 `idf.py fullclean` |
| Wi-Fi 连不上 AP | 查 `WIFI_EVENT_STA_DISCONNECTED` 的 reason code；确认 NVS 已初始化、SSID/密码正确、信号强度 |
| OTA 校验失败 | 确认镜像同目标、签名/哈希；分区表含 ota_0/ota_1/otadata；`esp_ota_set_boot_partition` 后再重启 |
| `nvs_flash_init` 返回 `ESP_ERR_NVS_NO_FREE_PAGES` | `nvs_flash_erase()` 后重新 `nvs_flash_init()` |
| Flash 写入异常 | 确认 4KB 扇区对齐、写前已擦；改用 NVS/wear_levelling |
| GPIO ISR 报 `INVALID_STATE` | 先 `gpio_install_isr_service(0)` 再 `gpio_isr_handler_add` |
| panic / Guru Meditation | 用 `idf.py monitor` 解码 backtrace；或 `idf.py mon -p PORT` 配合 addr2line |

## References

- 场景 recipe → `recipes/` 目录
- API 速查（按模块） → `resources/api_reference.md`
- Kconfig / sdkconfig 配置 → `resources/config_reference.md`
- 常见陷阱汇总 → `resources/pitfalls.md`
- 示例工程索引（真实 examples/ 路径） → `resources/example_list.md`
- ESP-IDF 仓库本体 → `D:/esp-skill/espressif-repos/esp-idf`
