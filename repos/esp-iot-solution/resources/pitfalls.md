# ESP-IoT-Solution 常见坑点汇总

> 本文汇总 `SKILL.md` 与各 recipe 的避坑要点。每条都来自仓库真实组件行为。若不确定，回到对应 `recipes/<scenario>.md` 看完整调用链。

## 通用 / 工程级

1. **入口必须是 `void app_main(void)`**，不是 `int main`。`app_main` 在 IDLE 优先级任务中运行，返回后该任务被删除；常驻逻辑要 `while(1) vTaskDelay(...)` 或自建任务。
2. **组件经 Component Manager 引入**：`idf.py add-dependency "espressif/<component>"`；依赖写在 `main/idf_component.yml`。
3. **拷贝 example 到别处后必须删除 `main/idf_component.yml` 中的 `override_path`**，否则 CMake 找不到原仓库相对路径。
4. **master 分支需 IDF >= v5.3**；release/v2.0 适配 v4.4–v5.3。组件级最低 IDF 见各 `idf_component.yml`。
5. **不要改 `managed_components/`**，会被 Component Registry 覆盖；要改版本就改 `idf_component.yml`。

## i2c_bus

6. **先建总线、再建设备；删除顺序相反**。带未删设备时 `i2c_bus_delete` 无效。
7. **无内部寄存器的设备用 `NULL_I2C_MEM_ADDR`**，不要把首字节当寄存器地址。
8. **IDF >= 5.3 默认走新 `i2c_master`**；要继续用旧 `driver/i2c` 须 menuconfig 启用 `CONFIG_I2C_BUS_BACKWARD_CONFIG`，且不要同时 `#include "driver/i2c.h"` 与 `"i2c_bus.h"`（`i2c_config_t` 定义冲突）。
9. **16 位寄存器地址设备用 `i2c_bus_read_reg16` / `i2c_bus_write_reg16`**，用 8 位接口会错位。
10. **多不同速率设备须开 `CONFIG_I2C_BUS_DYNAMIC_CONFIG`**（默认开启）。

## spi_bus

11. **总线只建一次**，同 host 多设备共享，每个设备各自 `spi_bus_device_create` 与独立 `cs_io_num`。
12. **`clock_speed_hz` 选 `SPI_MASTER_FREQ_*` 系列**（80MHz 整除），`mode` 与从机一致。
13. **`max_transfer_sz` < 4096 自动取 4096**；大屏刷帧要显式设大。
14. **不要解引用 opaque handle**（`spi_bus_handle_t` 是 `void *`），用 `spi_bus_transmit_begin` + `spi_transaction_t` 走底层。

## button

15. **用工厂函数 `iot_button_new_gpio_device`**（分离 `button_config_t` + `button_gpio_config_t`），旧 `iot_button_create(&cfg)` 单结构体 API 已废弃，新版 `button_config_t` 不含 `gpio_num`。
16. **`iot_button_register_cb(handle, event, event_args*, cb, usr_data)`**：第 3 参数对单击/双击/长按可传 `NULL`；多击用 `button_event_args_t.multiple_clicks.clicks`。
17. **回调内禁止阻塞**（`vTaskDelay`、长 `printf`），否则丢事件并卡住扫描定时器；耗时工作转交任务/队列。
18. **低功耗**：`button_gpio_config_t.enable_power_save=true`，配合 `iot_button_register_power_save_cb` + `esp_pm_configure(light_sleep_enable=true)`；GPIO 唤醒由组件配置。

## knob

19. **回调里用 `usr_data` 区分 `KNOB_LEFT`/`KNOB_RIGHT`**：注册时把事件枚举作为 `usr_data` 传入，回调里强转读取。

## led_indicator

20. **`blink_lists` 数组下标即优先级（0 最高），且要连续**；末尾 `BLINK_MAX` 位置填 `NULL`，`blink_list_num` 与枚举数一致。
21. **闪烁序列必须以 `LED_BLINK_STOP` 或 `LED_BLINK_LOOP` 收尾**，否则状态机行为未定义。
22. **GPIO 单色灯用 `LED_DUTY_1_BIT`**；调亮度/颜色用 LEDC/RGB/Strips 后端（`led_indicator_set_brightness` 对 GPIO 无效）。

## sensor_hub

23. **传感器名要与驱动注册名一致**（如 `"sht3x"`、`"lis2dh12"`、`"veml6040"`），用 `iot_sensor_scan()` 看精确名字。
24. **创建后必须 `iot_sensor_start`** 才会发 `SENSOR_*_DATA_READY` 事件。
25. **`min_delay` 设合理**（100~1000ms），过小会过频。
26. **`bus` 必须是已建好的 `i2c_bus_handle_t`**（或 board 组件的 `iot_board_get_handle`）。

## usb_stream（仅 ESP32-S2/S3）

27. **仅支持 ESP32-S2/S3**（USB OTG），ESP32 无 USB OTG。
28. **UVC 摄像头须 USB1.1 全速 + MJPEG**；等时带宽 < 4Mbps（500KB/s）、bulk < 8.8Mbps（1100KB/s）、等时接口 MPS ≤ 512 字节。
29. **mic 回调禁止阻塞**，否则丢帧；要阻塞处理改用 `uac_mic_streaming_read` 轮询。
30. **先 config 再 start**；已运行须先 `usb_streaming_stop()`。
31. **`spk_buf_size` 须为 `spk_ep_mps` 整数倍**。
32. **`uvc_frame_size_reset` / `uac_frame_size_reset` 需在 SUSPEND 状态调用**，RESUME 后生效。

## iot_usbh_cdc

33. **驱动全局单例**，`usbh_cdc_driver_install` 只调一次。
34. **无环形缓冲模式**（`in_ringbuf_size=0`）必须靠 `recv_data` 回调读，且读长度必须等于 `in_transfer_buffer_size`。
35. **`usbh_cdc_driver_uninstall` 前须关闭所有端口**。
36. **设置波特率**用 `usbh_cdc_send_custom_request` 发 SET_LINE_CODING（`bmRequestType=0x21`，`bRequest=0x03`）。

## iot_servo

37. **`channel_number` 必须等于实际填入的通道项数**，否则错位/不动。
38. **`min/max_width_us` 按舵机规格**（常见 500~2500µs，部分 SG90 用 500~2400µs）。
39. **`iot_servo_write_angle` 非线程安全**，多任务调用加锁。
40. **舵机用独立 5V 供电**，勿从 3V3/开发板取大电流。

## touch_button / touch_button_sensor

41. **需 IDF >= v5.3**，芯片须带 Touch（ESP32/S2/S3/P4）。
42. **`touch_button_sensor` 须周期调 `touch_button_sensor_handle_events`** 才会出回调（建任务 10~30ms 调一次）。
43. **触摸 config 结构体是 `button_touch_config_t`**（`touch_button` 组件）或 `touch_button_config_t`（`touch_button_sensor` 组件），二者不同，别混用。
44. **ESP32/S2/S3 触摸抗干扰有限，仅供测试/演示**，量产建议改专用触摸 IC 或其他方案。

## SPI LCD（基于 esp_lcd）

45. **SPI 不直接 DMA PSRAM**，大帧缓冲放 PSRAM 会先拷到 SRAM 可能耗尽内部 SRAM；推荐内部 SRAM 做 LVGL 渲染缓冲。
46. **ESP 硬件不支持 LCD 的 3 线（9 位）模式**，须用 `esp_lcd_panel_io_additions` 组件软件模拟（常见于 RGB LCD 初始化命令）。
47. **接口 I 模式只设 `mosi_io_num`，`miso_io_num` 设 -1**；`max_transfer_sz` 通常设为屏幕大小。
