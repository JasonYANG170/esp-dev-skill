# ESP-IoT-Solution 配置项速查（Kconfig）

> 所有符号取自仓库 `components/<name>/Kconfig`。通过 `idf.py menuconfig` 在「Component config」下访问。只收录真实存在的 Kconfig 符号。

## 分支 / IDF 版本

| ESP-IoT-Solution | 依赖 ESP-IDF | 状态 |
|:---:|:---:|:---:|
| master | >= v5.3 | 新特性开发 |
| release/v2.0 | >= v4.4, <= v5.3 | Bugfix 维护 |
| release/v1.1 | v4.0.1 | 归档 |
| release/v1.0 | v3.2.2 | 归档 |

> 各组件的实际 IDF 最低版本见其 `idf_component.yml` 的 `dependencies.idf`（如 `button>=4.0`、`led_indicator>=5.0`）。

## i2c_bus（Component config → Bus Options → I2C Bus Options）

| 符号 | 类型 | 默认 | 说明 |
|---|---|---|---|
| `I2C_BUS_DYNAMIC_CONFIG` | bool | y | 每次传输前动态重装驱动，支持同总线多不同速率设备 |
| `I2C_MS_TO_WAIT` | int | 200 | 取总线互斥锁的阻塞时间 ms（范围 50~5000） |
| `I2C_BUS_BACKWARD_CONFIG` | bool | n | IDF>=5.3 强制使用旧 `driver/i2c` 驱动（向后兼容） |
| `I2C_BUS_SUPPORT_SOFTWARE` | bool | n | 启用软件 I2C（用于硬件 I2C 不足或调试） |
| `I2C_BUS_SOFTWARE_MAX_PORT` | int | 2 | 软件 I2C 端口上限（1~5，依赖上一项） |
| `I2C_BUS_REMOVE_NULL_MEM_ADDR` | bool | n | 禁用 NULL_MEM_ADDR 限制，任意寄存器地址都会发送 |

## button（Component config → IoT Button）

| 符号 | 类型 | 默认/范围 | 说明 |
|---|---|---|---|
| `BUTTON_PERIOD_TIME_MS` | int | 5（2~500） | 按键扫描周期 ms |
| `BUTTON_DEBOUNCE_TICKS` | int | 2（1~7） | 消抖节拍数（1 节拍 = 1 个 PERIOD） |
| `BUTTON_SHORT_PRESS_TIME_MS` | int | 180（50~800） | 短按判定阈值 |
| `BUTTON_LONG_PRESS_TIME_MS` | int | 1500（500~5000） | 长按判定阈值 |
| `BUTTON_LONG_PRESS_HOLD_SERIAL_TIME_MS` | int | 20（2~1000） | 长按保持事件连续触发间隔 |
| `ADC_BUTTON_MAX_CHANNEL` | int | 3（1~5） | ADC 按键最大通道数 |
| `ADC_BUTTON_MAX_BUTTON_PER_CHANNEL` | int | 8（1~10） | 每通道最大按键数 |
| `ADC_BUTTON_SAMPLE_TIMES` | int | 1（1~4） | 每次扫描采样次数 |

## knob（Component config → IOT Knob）

| 符号 | 类型 | 默认/范围 | 说明 |
|---|---|---|---|
| `KNOB_PERIOD_TIME_MS` | int | 3（2~10） | 旋钮扫描周期 ms |
| `KNOB_DEBOUNCE_TICKS` | int | 2（1~8） | 消抖节拍数 |
| `KNOB_HIGH_LIMIT` | int | 1000（1~10000） | 计数上限 |
| `KNOB_LOW_LIMIT` | int | -1000（-10000~-1） | 计数下限 |

## led_indicator（Component config → LED Indicator）

| 符号 | 类型 | 默认/范围 | 说明 |
|---|---|---|---|
| `BRIGHTNESS_TICKS` | int | 10（1~50） | 单次亮度变化所需节拍 |
| `USE_GAMMA_CORRECTION` | bool | y | 启用伽马校正 |
| `USE_MI_RGB_BLINK_DEFAULT` | bool | n | 使用小米 RGB 默认闪烁表 |
| `LEDC_SPEED_MODE` | choice | 低速 | LEDC 速度模式（依赖 `SOC_LEDC_SUPPORT_HS_MODE`） |
| `LEDC_TIMER_BIT_NUM` | int | 13（1~14/20） | LEDC 定时器位宽 |
| `LEDC_TIMER_FREQ_HZ` | int | 5000 | LEDC 定时器频率（1~40000000） |

## usb_stream（Component config → USB Stream；仅 ESP32-S2/S3）

常用项：

| 符号 | 类型 | 默认 | 说明 |
|---|---|---|---|
| `CTRL_TRANSFER_DATA_MAX_BYTES` | int | 1024（64~2048） | 控制传输最大数据字节 |
| `USB_STREAM_QUICK_START` | bool | n | 快速启动（跳过描述符获取/解析，需手动指定全部可选参数） |
| `UVC_GET_DEVICE_DESC` / `UVC_GET_CONFIG_DESC` / `UVC_PRINT_DESC` | bool | y | 枚举期间获取/解析/打印描述符（QUICK_START 时禁用） |
| `USB_PROC_TASK_PRIORITY` | int | 5（1~25） | USB 处理任务优先级 |
| `USB_PROC_TASK_CORE` | int | S3 默认 1、S2 默认 0 | USB 处理任务核 |
| `USB_PROC_TASK_STACK_SIZE` | int | 3072 | USB 处理任务栈 |
| `USB_ENUM_FAILED_RETRY` | bool | y | 枚举失败重试 |
| `USB_ENUM_FAILED_RETRY_COUNT` | int | 10 | 重试次数上限 |
| `SAMPLE_PROC_TASK_PRIORITY` | int | 2（1~25） | UVC 采样处理任务优先级 |
| `UVC_DROP_OVERFLOW_FRAME` | bool | y | 丢弃溢出图像帧 |
| `NUM_BULK_STREAM_URBS` / `NUM_BULK_BYTES_PER_URB` | int | 2 / 2048 | bulk 模式 urb 数与每段字节数 |
| `NUM_ISOC_UVC_URBS` / `NUM_PACKETS_PER_URB` | int | 3 / 4 | 等时模式 urb 数与每 urb 包数 |
| `UAC_MIC_CB_MIN_MS_DEFAULT` | int | 16（1~32） | mic 回调默认间隔 ms |
| `UAC_SPK_ST_MAX_MS_DEFAULT` | int | 16（1~32） | speaker 默认最大发送时长 ms |
| `UAC_SPK_PACKET_COMPENSATION` | bool | y | speaker 包补偿 |

## 工程级常用 sdkconfig（非组件，来自 ESP-IDF，但工程常调）

| 符号 | 用途 |
|---|---|
| `CONFIG_IDF_TARGET` | 目标芯片（esp32 / esp32s3 / esp32c3 …） |
| `CONFIG_FREERTOS_USE_TICKLESS_IDLE` | 启用 tickless，配合 `esp_pm_configure` 的 `light_sleep_enable` |
| `CONFIG_PM_ENABLE` | 启用电源管理 |

> 组件版本与依赖以 `idf_component.yml` 为准；示例拷贝后必须删除其中的 `override_path`。
