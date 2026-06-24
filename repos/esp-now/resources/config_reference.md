# ESP-NOW 组件配置参考（Kconfig）

> 全部选项取自仓库根 `Kconfig`，通过 `idf.py menuconfig` → `ESP-NOW Configuration` 修改。类型与默认值以源码为准。

## Security Configuration

| Kconfig 选项 | 类型 | 默认 | 说明 |
|---|---|---|---|
| `CONFIG_ESPNOW_APP_SECURITY` | bool | y | 加密应用层数据（总开关） |
| `CONFIG_ESPNOW_ALL_SECURITY` | bool | n | 加密所有数据（依赖 `_APP_SECURITY`） |
| `CONFIG_ESPNOW_CONTROL_SECURITY` | bool | n | 加密控制数据（依赖 `_APP_SECURITY` 且非 `_ALL_SECURITY`） |
| `CONFIG_ESPNOW_DEBUG_SECURITY` | bool | n | 加密调试数据 |
| `CONFIG_ESPNOW_OTA_SECURITY` | bool | n | 加密 OTA 数据 |
| `CONFIG_ESPNOW_PROV_SECURITY` | bool | n | 加密配网数据 |

> 当 `CONFIG_ESPNOW_ALL_SECURITY=y` 时，会自动令 `CONFIG_ESPNOW_*_SECURITY=1`（见各头文件中的 `#ifdef CONFIG_ESPNOW_ALL_SECURITY` 派生）。

## Light Sleep Configuration

| Kconfig 选项 | 类型 | 范围/默认 | 说明 |
|---|---|---|---|
| `CONFIG_ESPNOW_LIGHT_SLEEP` | bool | n | 发包前进入 light sleep 给电容充电（硬币电池方案） |
| `CONFIG_ESPNOW_LIGHT_SLEEP_DURATION` | int | 15~120，默认 30 | light sleep 持续 ms |

## Control Configuration

| Kconfig 选项 | 类型 | 范围/默认 | 说明 |
|---|---|---|---|
| `CONFIG_ESPNOW_CONTROL_AUTO_CHANNEL_SENDING` | bool | n | 自动按 [1,6,11,1,6,11,2,3,4,5,7,8,9,10,12,13] 序列发包 |
| `CONFIG_ESPNOW_CONTROL_WAIT_ACK_DURATION` | int | 3~100，默认 40 | 等 ACK 时长（依赖 auto channel） |
| `CONFIG_ESPNOW_CONTROL_RETRANSMISSION_TIMES` | int | 1~15，默认 5 | 在保存信道上的重传次数 |
| `CONFIG_ESPNOW_CONTROL_AUTO_CHANNEL_FORWARD` | bool | n | 不同信道间转发 |
| `CONFIG_ESPNOW_CONTROL_FORWARD_TTL` | int | 1~31，默认 10 | 转发最大跳数 |
| `CONFIG_ESPNOW_CONTROL_FORWARD_RSSI` | int | -70~-30，默认 -55 | 转发最低信号阈值 |

## OTA Configuration

| Kconfig 选项 | 类型 | 范围/默认 | 说明 |
|---|---|---|---|
| `CONFIG_ESPNOW_OTA_RETRANSMISSION_TIMES` | int | 1~15，默认 2 | OTA 包在保存信道上的重传次数 |
| `CONFIG_ESPNOW_OTA_RETRY_COUNT` | int | 默认 50 | initiator 更新设备重试次数 |
| `CONFIG_ESPNOW_OTA_SEND_FORWARD_TTL` | int | 0~31，默认 0 | OTA 转发最大跳数 |
| `CONFIG_ESPNOW_OTA_SEND_FORWARD_RSSI` | int | -70~-30，默认 -65 | OTA 转发最低信号 |
| `CONFIG_ESPNOW_OTA_WAIT_RESPONSE_TIMEOUT` | int | 100~10000，默认 10000 | 等 responder 应答 ms |

## 通用行为

| Kconfig 选项 | 类型 | 默认 | 说明 |
|---|---|---|---|
| `CONFIG_ESPNOW_AUTO_RESTORE_CHANNEL` | bool | n | 发包结束若主信道变了则恢复 |
| `CONFIG_ESPNOW_DATA_FAST_ACK` | bool | n | 在接收回调里立即处理 ACK（否则排队空闲时处理） |

## Task Configuration

| Kconfig 选项 | 类型 | 范围/默认 | 说明 |
|---|---|---|---|
| `CONFIG_ESPNOW_TASK_STACK_SIZE` | int | 2048~8192，默认 4096 | espnow 任务栈大小 |
| `CONFIG_ESPNOW_TASK_PRIORITY` | int | 1~25，默认 1 | espnow 任务优先级 |

> 官方提示：回调里勿做重活（JSON/加密/深层调用），应通过队列移交应用任务，否则可能栈溢出，必要时调大 `TASK_STACK_SIZE`。

## Utils Configuration

| Kconfig 选项 | 类型 | 默认 | 说明 |
|---|---|---|---|
| `CONFIG_ESPNOW_MEM_ALLOCATION_DEFAULT` | choice | 选 | MALLOC/CALLOC/REALLOC 用默认策略 |
| `CONFIG_ESPNOW_MEM_ALLOCATION_SPIRAM` | choice | — | 分配到 SPIRAM（依赖芯片 SPIRAM 支持） |
| `CONFIG_ESPNOW_MEM_DEBUG` | bool | y | 内存调试记录 |
| `CONFIG_ESPNOW_MEM_DBG_INFO_MAX` | int | 默认 128 | 内存调试最大记录数 |
| `CONFIG_ESPNOW_NVS_NAMESPACE` | string | "espnow" | NVS 命名空间 |
| `CONFIG_ESPNOW_REBOOT_UNBROKEN_INTERVAL_TIMEOUT` | int | 默认 5000 | 连续重启判定间隔(ms) |
| `CONFIG_ESPNOW_REBOOT_UNBROKEN_FALLBACK_COUNT` | int | 默认 30 | 连续重启触发版本回滚次数 |

## Debug Configuration

### Console
| Kconfig 选项 | 类型 | 默认 | 说明 |
|---|---|---|---|
| `CONFIG_ESPNOW_STORE_HISTORY` | bool | y | 命令历史存 flash（FAT） |
| `CONFIG_ESPNOW_DEBUG_CONSOLE_UART_NUM_0` / `_1` | choice | 0 | console 输入 UART |
| `CONFIG_ESPNOW_DEBUG_CONSOLE_UART_NUM` | int | 0 | 实际值(0/1) |

### Debug Log
| Kconfig 选项 | 类型 | 默认 | 说明 |
|---|---|---|---|
| `CONFIG_ESPNOW_DEBUG_LOG_PARTITION_LABEL_DATA` | string | "log_info" | 日志数据分区 label |
| `CONFIG_ESPNOW_DEBUG_LOG_PARTITION_LABEL_NVS` | string | "log_status" | 日志状态分区 label |
| `CONFIG_ESPNOW_DEBUG_LOG_FILE_MAX_SIZE` | int | 8196~131072，默认 65536 | 单日志文件大小 |
| `CONFIG_ESPNOW_DEBUG_LOG_PARTITION_OFFSET` | int | 0~524288，默认 0 | 日志分区偏移 |
| `CONFIG_ESPNOW_DEBUG_LOG_PRINTF_ENABLE` | bool | n | 输出 espnow 模块 printf |

## 组件依赖与版本

- `idf_component.yml`: `version: "2.5.3"`, `dependencies: idf >= 4.4`, `cmake_utilities >= 0.5.3`
- 添加到工程: `idf.py add-dependency "espressif/esp-now=*"`

## IDF 版本兼容（来自 User_Guide.md）

| ESP-NOW 分支 | IDF v4.4 | v5.0 | v5.1 | v5.2 | master |
|---|---|---|---|---|---|
| master | 支持 | 支持 | 支持 | 支持 | 支持 |
| v2.x.x | 支持(内置从 v4.4) | 支持 | 支持 | 支持 | 支持 |
| v1.0 | 支持(内置 v4.4) | 不支持 | 不支持 | 不支持 | 不支持 |
