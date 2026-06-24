# 常见陷阱汇总

> 所有 WRONG/CORRECT 源自真实 ESP-IDF 用法。详见 `SKILL.md` 的 "Critical Pitfalls"。

## 系统/入口

### app_main 返回后任务被删除
ESP-IDF 启动后由系统任务调用 `void app_main(void)`。它返回后该任务被删除，程序"卡住"。长驻逻辑必须自建 `while(1)` 或 `xTaskCreate`。

```c
// ❌
void app_main(void){ init(); }   // 返回后无主循环

// ✅
void app_main(void){ init(); while(1) vTaskDelay(pdMS_TO_TICKS(1000)); }
```

### esp_err_t 静默失败
返回 `esp_err_t` 的调用必须检查；不可恢复用 `ESP_ERROR_CHECK`（失败 abort 并打印 backtrace），可恢复则判断后用 `esp_err_to_name` 输出。

### 日志被裁剪
`ESP_LOGD/V` 在 `CONFIG_LOG_DEFAULT_LEVEL <= INFO` 时被编译裁剪；调时用 `esp_log_level_set(tag, ESP_LOG_DEBUG)` 运行时提级，或调高 `CONFIG_LOG_DEFAULT_LEVEL`。

## 构建/组件

### 顶层 CMakeLists 顺序错
必须严格 `cmake_minimum_required(3.22)` → `include($ENV{IDF_PATH}/tools/cmake/project.cmake)` → `project(name)`。

### 组件未注册
`main/CMakeLists.txt` 必须用 `idf_component_register(SRCS ... INCLUDE_DIRS ... REQUIRES ... PRIV_REQUIRES ...)`；漏依赖会出现 `undefined reference`。

### 未先 set-target
首次或换芯片必须 `idf.py set-target <chip>`；换目标前 `idf.py fullclean`。

### 路径含空格
ESP-IDF 构建不支持路径含空格；移到无空格目录。

## 外设

### GPIO ISR 未装服务
`gpio_isr_handler_add` 前必须全局调一次 `gpio_install_isr_service(0)`，否则返回 `ESP_ERR_INVALID_STATE`。

### ISR 内用阻塞 API
ISR 函数须 `IRAM_ATTR`，只调 `...FromISR` 系列；禁止 `vTaskDelay`、`xQueueSend`、`printf`、`malloc`（malloc 在某些路径安全但应避免）。

### UART 未 install 驱动
仅 `uart_param_config` 不够，必须 `uart_driver_install` 才能读到数据。

### I2C 用了旧 v4 API
v5.x 弃用 `i2c_param_config`/`i2c_master_write`；改用 `i2c_new_master_bus` + `i2c_master_bus_add_device` + `i2c_master_transmit_receive`。

### SPI length 单位是 bit
`spi_transaction_t.length` 单位为 bit，4 字节写 `.length = 4 * 8`。

### SPI1_HOST 不能占用
`SPI1_HOST` 接主 Flash，外设用 `SPI2_HOST`/`SPI3_HOST`。

### LEDC 改 duty 不生效
`ledc_set_duty` 后必须 `ledc_update_duty` 才生效。

### GPTimer 改配置无效
改回调/告警前必须 `gptimer_disable`，改完再 `gptimer_enable` + `gptimer_start`。

## 网络

### Wi-Fi 缺 netif / 事件循环
`esp_wifi_init/start` 前必须 `esp_netif_init()` + `esp_event_loop_create_default()` + `esp_netif_create_default_wifi_sta/ap()`。

### NVS 未初始化致 Wi-Fi 崩
`app_main` 开头应初始化 NVS，并处理 `ESP_ERR_NVS_NO_FREE_PAGES` / `ESP_ERR_NVS_NEW_VERSION_FOUND`（`nvs_flash_erase()` 后重试）。

### SoftAP 用错 netif
AP 用 `esp_netif_create_default_wifi_ap()`，STA 用 `create_default_wifi_sta()`；APSTA 模式两者都要。

## 蓝牙

### BT 启动顺序错
固定顺序不可颠倒：NVS → `esp_bt_controller_init` → `esp_bt_controller_enable(mode)` → `esp_bluedroid_init_with_cfg` → `esp_bluedroid_enable` → 注册回调 → 注册 app。颠倒会导致崩溃或返回错误。

```c
// ❌ bluedroid 在 controller enable 前就 init
esp_bluedroid_init_with_cfg(&cfg);   // 此时控制器没起，失败
esp_bt_controller_init(&bt_cfg);

// ✅
esp_bt_controller_init(&bt_cfg);
esp_bt_controller_enable(ESP_BT_MODE_BLE);
esp_bluedroid_init_with_cfg(&cfg);
esp_bluedroid_enable();
```

### BLE/Classic 模式不匹配 / 内存未释放
BLE 与 Classic BT 共用控制器，模式互斥。纯 BLE 应 `esp_bt_controller_mem_release(ESP_BT_MODE_CLASSIC_BT)` 腾内存；纯 Classic 应 release `ESP_BT_MODE_BLE`；双模用 `ESP_BT_MODE_BTDM` 且不能 release。

### 用错 Host API
Bluedroid（`CONFIG_BT_BLUEDROID_ENABLED`）用 `esp_ble_gap_*` / `esp_ble_gatts_*`；NimBLE（`CONFIG_BT_NIMBLE_ENABLED`）用 `ble_gap_*`，API 完全不同。两套互斥，不可混用。

### Classic BT 用在 esp32c3/s3
Classic BT（SPP/A2DP/HFP）仅 esp32 双模支持（`SOC_BT_CLASSIC_SUPPORTED`）。c3/s3/c6/h2 等无 Classic BT，需 `CONFIG_BT_CLASSIC_ENABLED` 选不上；改用 BLE 方案（BLE SPP / BLE Audio）。

### Classic GAP 头用错
经典蓝牙 GAP 用 `esp_gap_bt_api.h`（`esp_bt_gap_*`，如 `set_device_name`/`set_scan_mode`）；BLE GAP 用 `esp_gap_ble_api.h`（`esp_ble_gap_*`）。两者不通用。

### Bluedroid v5.x init API 变了
v5.x 用 `esp_bluedroid_init_with_cfg(BT_BLUEDROID_INIT_CONFIG_DEFAULT())`，旧 `esp_bluedroid_init()` 仍存在但推荐新版本。SPP 用 `esp_spp_enhanced_init(&esp_spp_cfg_t)` 取代 `esp_spp_init(mode)`。

### BT 回调里做重活
A2DP/SPP/mesh 的协议栈回调上下文不适合做 I2S 写、解码、长时间处理。用任务派发（队列 + 应用任务），回调仅转发数据。

### BLE Mesh 配网后收不到 SET
配网（Provisioning）完成 ≠ 能收应用消息。Provisioner 还要通过 Config Server 发 `AppKey Add` + `Model App Bind` 给模型绑 AppKey，之后模型才能收发消息。

## 组网 / 以太网

### Wi-Fi Mesh 用错 netif
mesh 用专用 `esp_netif_create_default_wifi_mesh_netifs(&sta, &ap)`，不是 `create_default_wifi_sta/ap`。

### esp_mesh_start 后调 esp_wifi API
mesh 自组织模式接管 Wi-Fi，`esp_mesh_start` 后、`esp_mesh_stop` 前禁止调 `esp_wifi_connect`/`esp_wifi_scan_*` 等，否则干扰 mesh。

### 以太网用错 netif 挂载
以太网用 `ESP_NETIF_DEFAULT_ETH()` + `esp_netif_new` + `esp_eth_new_netif_glue` + `esp_netif_attach`，不是 wifi 的接口。

### 以太网 PHY 地址/复位脚错
`phy_addr` 由硬件 PHY 决定（常见 0/1，PHY 自举电阻设定）；`reset_gpio_num` 必须对应板子 PHY 复位脚。错则 `Link Down` 一直不上。

## 存储

### NVS 写后未 commit
写操作（`nvs_set_*`）后调 `nvs_commit` 才保证落盘（虽然 close 时会自动 commit，但中途断电会丢）。

### Flash 覆盖写
SPI Flash 必须先擦后写，最小擦除单位 4KB（扇区）。`esp_partition_erase_range` 的 `offset/size` 必须 4KB 对齐。频繁/大块数据用 NVS / wear_levelling / FATFS。

### 改分区表不擦 flash
改分区表或 OTA 后，旧数据偏移错位，必须 `idf.py erase-flash` 再重烧。

## OTA

### 缺 OTA 分区
OTA 需 `factory` + `ota_0` + `ota_1` + `otadata`；menuconfig 选 `Factory app, two OTA` 或自定义 CSV。

### 升级后不切换
`esp_ota_end` 后必须 `esp_ota_set_boot_partition(next)` 再 `esp_restart()`；只 `esp_ota_end` 不切分区，下次仍启动旧版。

### HTTPS 证书
生产环境必须内嵌正确 PEM 证书；`skip_cert_common_name_check` 仅测试用。

## 低功耗

### 深睡变量全丢
`esp_deep_sleep_start` 后 RAM 掉电，醒来等价复位从 `app_main` 重新执行；重要状态写 RTC memory 或 NVS。

### ext0/ext1 必须用 RTC GPIO
非 RTC GPIO 不能做 ext0/ext1 唤醒源；查目标 SoC datasheet 的 RTC GPIO 列表。

### esp_deep_sleep_start 不返回
该函数之后写的代码不会执行；不要依赖其后逻辑。
