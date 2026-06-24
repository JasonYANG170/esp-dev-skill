# ESP8266_RTOS_SDK 常见陷阱汇总

> 与 `SKILL.md` 的 "Critical Pitfalls" 互为补充，按模块归类的扩展陷阱。所有结论来自仓库头文件注释与示例。

## 1. 框架 / 版本

- **不要混用 NonOS SDK / RTOS_SDK v2 的 API。** 本仓��自 v3.0 起是 esp-idf style，入口是 `app_main`，不是 `user_init`，也没有 `system_init_done_cb`。
- **确认芯片是 ESP8266EX。** ESP32 / ESP32-S/C/H 用各自 esp-idf，GPIO 数量、双核、蓝牙都不同。本 SDK 函数与 ESP32 esp-idf 高度相似但外设集合不同。
- **工具链版本对不上会链接失败。** 本 SDK（v3.0+）用 xtensa-lx106-elf gcc8.4.0（esp-2020r3）；若误装 v4.8.5（旧 SDK 用），编译报错。

## 2. app_main / 任务

- `app_main` 在系统主任务里运行，返回后该任务被删除；**不要在 app_main 里写裸 `while(1)` 阻塞**，会饿死系统事件任务。长逻辑用 `xTaskCreate`。
- ESP8266 FreeRTOS 任务栈单位是 **byte**（不是 word），`xTaskCreate(fn, "n", 4096, ...)` 是 4096 字节。
- `vTaskDelay(1000 / portTICK_PERIOD_MS)` 延时 1 秒；不要忙等 `for(;;)` 不让 CPU。

## 3. NVS / WiFi

- **WiFi 前必须 `nvs_flash_init()`**，否则 `esp_wifi_init` 默认需要 NVS 会 abort。
- `nvs_flash_init()` 返回 `ESP_ERR_NVS_NO_FREE_PAGES` 或 `ESP_ERR_NVS_NEW_VERSION_FOUND` 时，应先 `nvs_flash_erase()` 再 `nvs_flash_init()`。
- 网络栈二选一成套：`tcpip_adapter_init()`（旧，station/softap/espnow 示例）或 `esp_netif_init()`（新，http_request/simple_ota 示例）；不要两套混用。
- Station 必须在 `WIFI_EVENT_STA_START` 事件里调 `esp_wifi_connect()`，**不要**在 `esp_wifi_start()` 返回后立即调。
- 断线必须在 `WIFI_EVENT_STA_DISCONNECTED` 里带计数重连，否则掉线后永不自动恢复。
- 拿 IP 是 `IP_EVENT_STA_GOT_IP`（base 是 `IP_EVENT`，不是 `WIFI_EVENT`）；发网络请求前要等该事件。
- SoftAP 不支持 WEP；密码 < 8 字符不能用 WPA2，空密码应设 `WIFI_AUTH_OPEN`。

## 4. GPIO

- **GPIO6~GPIO11 接 SPI flash，禁用**；用作普通 IO 会崩溃。
- GPIO1/3 默认是 UART0 TX/RX；要复用需 `uart_disable_swap()` 并注意日志会丢。
- 普通 GPIO 只能上拉不能下拉；**只有 GPIO16 反过来**（能下拉、不能上拉），见 `driver/gpio.h` 注释。
- GPIO16 是唯一 RTC GPIO，可参与 deep sleep 唤醒。
- 中断用 per-pin 模型：`gpio_install_isr_service(0)` → `gpio_isr_handler_add(n, fn, arg)`；ISR 里只能 `xQueueSendFromISR` 等非阻塞 API，不能 printf/阻塞。
- `gpio_install_isr_service` 与 `gpio_isr_register`（全局 ISR）互斥，二选一。

## 5. UART

- 读/写前必须 `uart_param_config` + `uart_driver_install`；不装驱动直接 `uart_read_bytes` 行为未定义。
- `rx_buffer_size` 必须 > `UART_FIFO_LEN`(128)；`tx_buffer_size=0` 时 `uart_write_bytes` 阻塞直到 FIFO 推完。
- `uart_flush` / `uart_flush_input` 只清 RX ring buffer，不影响 TX FIFO；要等 TX 发完用 `uart_wait_tx_done`。
- 推荐事件队列模型：`uart_driver_install(..., &uart_queue, 0)`，在独立 task 里 `xQueueReceive(uart_queue, &event, ...)` 处理 `UART_DATA` / `UART_FIFO_OVF` / `UART_BUFFER_FULL`，见 `examples/peripherals/uart_events`。

## 6. PWM

- PWM 是软件实现（`driver/pwm.h` 的 `pwm_*`），最多 8 通道。
- **任何配置修改（duty / period / phase / invert）后必须 `pwm_start()` 才生效**。
- `period` 不要低于 20us（否则波形异常）；占空比是绝对值，`real_duty = duty / period`。
- `pwm_stop(mask)` 的 mask 决定各通道停止后输出电平。

## 7. I2C

- ESP8266 I2C 只有 `I2C_NUM_0` 且**仅主模式**（软件实现）。
- 用命令链：`i2c_cmd_link_create` → `i2c_master_start` → `write_byte` → ... → `i2c_master_stop` → `i2c_master_cmd_begin` → `i2c_cmd_link_delete`。
- `clk_stretch_tick` 影响 SCL 拉低时间；SCL/SDA 引脚驱动会自动开内部上拉，外部可不再加上拉。

## 8. ADC

- `adc_read_fast` 批量测量期间需**关闭 WiFi 与中断**，否则结果抖动（见 `driver/adc.h` 注释）。
- 测量模式由 menuconfig `Component config → PHY → vdd33_const` 决定：255=测系统电压（TOUT 必须悬空）；其它=测 TOUT 外部电压。
- `clk_div` 范围 [8,32]，采样时钟 = 80MHz / clk_div。

## 9. OTA / 分区表

- OTA 必须用 "Two OTA app" 分区表（ota_0/ota_1 + otadata）；用 "Single factory app" 时 `esp_https_ota`/`esp_ota_write` 失败。
- `app 分区必须完整落在单个 1MB 集成分区内`，跨边界运行时崩溃。
- 底层 OTA：`esp_ota_begin` → `esp_ota_write`（循环）→ `esp_ota_end` → `esp_ota_set_boot_partition` → `esp_restart`。**漏 `set_boot_partition` 会继续启动旧分区。**
- HTTPS OTA：证书（`cert_pem`）要匹配服务器；证书以二进制嵌入（`EMBED_TXTFILES` 或 `asm("_binary_ca_cert_pem_start")`）。

## 10. SPIFFS / 存储

- SPIFFS 需自定义分区表含 `data, spiffs` 子类型分区；默认表没有。
- 挂载设 `format_if_mount_failed = true` 避免首次烧录失败。
- SPIFFS **无掉电保护**，写后必须 `fclose`；重要配置优先用 NVS。
- `spi_flash_write` 写前必须先 `spi_flash_erase_range`（按扇区 4KB），不能直接改字节。

## 11. Sleep / 功耗

- `esp_deep_sleep` 唤醒等同复位，从 `app_main` 重新执行；进 deep sleep 前应 `esp_wifi_stop`/`esp_wifi_deinit` 关闭协议栈。
- light sleep 需要 `esp_wifi_set_ps(WIFI_PS_MIN_MODEM)` 等，且 GPIO 唤醒需 `esp_sleep_enable_gpio_wakeup`。
- 已废弃但仍可用的 `esp_wifi_fpm_*` 系列（force sleep）；新代码优先用 `esp_light_sleep_start` + `esp_sleep_enable_*`。

## 12. 日志

- 用 `ESP_LOGx(TAG, ...)` 而非 printf；TAG 是模块名。
- 等级由 menuconfig `CONFIG_LOG_DEFAULT_LEVEL` 控制，编译期会把高级别以下的日志去掉以省 flash。
- 调试时可 `esp_log_level_set("WIFI", ESP_LOG_DEBUG)` 运行期动态打开某模块。

## 13. 构建 / 烧录

- **`IDF_PATH` 必须设置且路径不含空格**（build-system.rst 明确）；工程目录路径同样不能含空格。
- 首次烧录用 `make flash`（含 bootloader + 分区表 + app + init data bin）；之后可 `make app-flash` 只烧 app。
- `make erase_flash` 全擦；改分区表后建议 `make erase_flash flash` 避免旧分区数据残留。
- `make monitor` Ctrl-] 退出；boot ROM 日志默认 74880 波特，应用日志常 115200，监视器会自动跟随。
- 并行构建：`make -jN`，N ≈ CPU 核数 + 1。
