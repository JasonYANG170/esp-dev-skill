# 无线调试：日志、Console 与命令

> **适用摘要**: 通过 ESP-NOW 远程抓取设备日志（按等级 UART/flash/ESPNOW/custom 分流）、下发调试命令、运行 console（参考 `examples/wireless_debug` 与 debug 模块头文件）。

## 触发意图

- "无线调试"
- "远程日志"
- "设备日志抓取"
- "espnow_log"
- "console 命令"
- "wireless debug"

## 前置条件

| 条件 | 要求 |
|---|---|
| 头文件 | `espnow_log.h`, `espnow_console.h`, `espnow_cmd.h` |
| 参考示例 | `examples/wireless_debug/main/app_main.c`（monitor / monitored 设备） |
| Kconfig | `CONFIG_ESPNOW_DEBUG_LOG_PARTITION_LABEL_DATA="log_info"` 等 |

## 分步说明

### 1. 日志初始化与等级配置（来自 espnow_log.h）

```c
#include "espnow_log.h"

espnow_log_config_t log_config = {
    .log_level_uart    = ESP_LOG_INFO,    // UART 打印等级
    .log_level_flash   = ESP_LOG_WARN,    // 存 flash 等级
    .log_level_espnow  = ESP_LOG_DEBUG,   // 经 ESP-NOW 发送等级
    .log_level_custom  = ESP_LOG_NONE,    // 自定义回调等级
    .log_custom_write  = NULL,            // 自定义写回调 espnow_log_custom_write_cb
};
espnow_log_init(&log_config);

// 运行期改配置
espnow_log_get_config(&log_config);
log_config.log_level_espnow = ESP_LOG_VERBOSE;
espnow_log_set_config(&log_config);

// 退出
espnow_log_deinit();
```

### 2. flash 日志读取与大小

```c
// 读取 flash(SPIFFS) 中存储的日志
size_t size = espnow_log_flash_size();   // 已存日志字节
char *buf = ESP_MALLOC(size);
espnow_log_flash_read(buf, &size);
// 导出或上报后释放
ESP_FREE(buf);
```

### 3. 自定义日志输出回调

```c
esp_err_t my_log_write(const char *data, size_t size, const char *tag, esp_log_level_t level)
{
    // 每次有日志满足 log_level_custom 时被调用
    return ESP_OK;
}
// 配置时 log_config.log_custom_write = my_log_write;
```

### 4. Console 初始化（接收 UART 或 ESP-NOW 命令）

```c
#include "espnow_console.h"

espnow_console_config_t console_cfg = {
    .monitor_command = { .uart = true, .espnow = true },  // 命令来源
    .store_history = {
        .base_path       = "/spiflash",
        .partition_label = "storage",   // 需 Kconfig ESPNOW_STORE_HISTORY=y
    },
};
espnow_console_init(&console_cfg);
espnow_console_commands_register();   // 注册公共命令

// 注册各类命令组（来自 espnow_cmd.h）
register_espnow();        // espnow 命令
register_system();        // 系统命令
register_wifi();          // WiFi 命令
register_peripherals();   // 外设命令
register_iperf();         // iperf
register_wifi_sniffer();  // sniffer
register_sdcard();        // sdcard

espnow_console_deinit();
```

### 5. wireless_debug 示例入口

```c
// examples/wireless_debug/main/app_main.c
#include "monitor.h"

void app_main(void)
{
#if CONFIG_APP_ESPNOW_DEBUG_MONITOR
    app_espnow_monitor_device_start();    // 调试主机：抓取被调试设备日志
#elif CONFIG_APP_ESPNOW_DEBUG_MONITORED
    app_espnow_monitored_device_start();  // 被调试设备：上报日志、接收命令
#endif
}
```

> `monitor.h` 与 `app_espnow_monitor_device_start` / `app_espnow_monitored_device_start` 来自 `examples/wireless_debug/components/espnow_device`，是示例内部封装，不是公开 API。日志/console/cmd 的公开 API 在 `espnow_log.h`/`espnow_console.h`/`espnow_cmd.h`。

### 关键 Kconfig（Debug Configuration 段）

| 选项 | 默认 | 说明 |
|---|---|---|
| `ESPNOW_STORE_HISTORY` | y | 命令历史存 flash（FAT） |
| `ESPNOW_DEBUG_LOG_PARTITION_LABEL_DATA` | "log_info" | 日志数据分区 |
| `ESPNOW_DEBUG_LOG_PARTITION_LABEL_NVS` | "log_status" | 日志状态分区 |
| `ESPNOW_DEBUG_LOG_FILE_MAX_SIZE` | 65536 | 单日志文件大小(8196~131072) |
| `ESPNOW_DEBUG_LOG_PRINTF_ENABLE` | n | 输出 espnow 模块 printf |
| `ESPNOW_DEBUG_CONSOLE_UART_NUM` | 0 | console UART(0 或 1) |

## 常见错误

| 错误 | 原因 | 解决方法 |
|---|---|---|
| 看不到 ESPNOW 日志 | `log_level_espnow` 设太高 | 设为 `ESP_LOG_DEBUG`/`VERBOSE` |
| flash 日志读不到 | 分区未配置 | 设 `ESPNOW_DEBUG_LOG_PARTITION_LABEL_DATA` 并在分区表加该分区 |
| console 历史丢失 | `ESPNOW_STORE_HISTORY=n` | 开启并配 `store_history.partition_label` |
| 命令无响应 | 未 `register_*` | 调用对应 `register_xxx()` 注册命令组 |
| 日志回调里阻塞 | 在回调做重活 | 回调内只转发，处理交应用任务 |

## 参考

- `examples/wireless_debug/main/app_main.c` + `examples/wireless_debug/components/espnow_device/`
- `src/debug/include/espnow_log.h` / `espnow_console.h` / `espnow_cmd.h`
- Kconfig：ESP-NOW Debug Configuration 段
