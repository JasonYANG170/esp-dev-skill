# ESP-AT Kconfig 配置选项速查

> 全部来自 `main/Kconfig`。在 `./build.py menuconfig` → `AT` 菜单中配置，或写入 `module_config/module_<name>/sdkconfig.defaults` / `sdkconfig_silence.defaults`，或用 `at_override_module_config/sdkconfig.defaults` 覆盖。

## 总开关与通信方式

| Kconfig | 默认 | 说明 |
|---|---|---|
| `AT_ENABLE` | y | AT 指令总开关 |
| `AT_COMMUNICATION_METHOD` | UART | choice：`AT_BASE_ON_UART` / `AT_BASE_ON_SPI` / `AT_BASE_ON_SOCKET` / `AT_BASE_ON_SDIO` |
| `AT_PROCESS_TASK_STACK_SIZE` | 2048 | AT 处理任务栈；HTTPS 需 ≥4096 |
| `AT_SOCKET_TASK_STACK_SIZE` | 6144（范围 2048–8196） | socket 任务栈 |
| `AT_COMMAND_TERMINATOR_SUPPORT` | n | 自定义指令终结符支持 |
| `AT_COMMAND_TERMINATOR` | 0x0A | 终结符（单字符，0x01–0xFF） |
| `AT_SELF_COMMAND_SUPPORT` | n | 固件内部执行 AT（`esp_at_exe_cmd`） |

## 指令集开关（基础）

| Kconfig | 默认 | 说明 |
|---|---|---|
| `AT_BASE_COMMAND_SUPPORT` | y | 基础指令集 |
| `AT_USER_COMMAND_SUPPORT` | y | 用户指令集 |
| `AT_USERWKMCU_COMMAND_SUPPORT` | y | `AT+USERWKMCU` 唤醒 MCU |
| `AT_SIGNALING_COMMAND_SUPPORT` | y | 信令测试指令 |
| `AT_OTA_SUPPORT` | y | OTA 指令 |

## Wi-Fi 相关

| Kconfig | 默认 | 说明 |
|---|---|---|
| `AT_WIFI_COMMAND_SUPPORT` | y | Wi-Fi 指令 |
| `AT_WPS_COMMAND_SUPPORT` | y | WPS（依赖 WIFI） |
| `AT_SMARTCONFIG_COMMAND_SUPPORT` | y | SmartConfig（依赖 WIFI） |
| `AT_EAP_COMMAND_SUPPORT` | n | WPA2/WPA3-Enterprise（依赖 WIFI，select 多项） |
| `AT_EAP_LEGACY_NAMESPACE_SUPPORT` | n | 旧 wpa2_ 命名空间兼容 |

## 网络 / TCP-IP

| Kconfig | 默认 | 说明 |
|---|---|---|
| `AT_NET_COMMAND_SUPPORT` | y | TCP/IP 指令 |
| `AT_SOCKET_MAX_CONN_NUM` | 5 | 最大连接数 |
| `AT_PING_COMMAND_SUPPORT` | y | PING 指令 |
| `AT_MDNS_COMMAND_SUPPORT` | y | mDNS 指令 |

### SSL 鉴权（依赖 NET）

| Kconfig | 默认 | 说明 |
|---|---|---|
| `AT_SSL_CLIENT_SERVER_AUTH_CLIENT` | y | 服务器认证客户端（需 client_cert/key 分区） |
| `AT_SSL_CLIENT_CLIENT_AUTH_SERVER` | y | 客户端认证服务器（需 client_ca 分区） |
| `AT_SSL_SERVER_SERVER_AUTH_CLIENT` | y | 服务端认证客户端（需 server_ca 分区） |
| `AT_SSL_SERVER_CLIENT_AUTH_SERVER` | y | 客户端认证服务端（需 server_cert/key 分区） |

## MQTT

| Kconfig | 默认 | 说明 |
|---|---|---|
| `AT_MQTT_COMMAND_SUPPORT` | n | MQTT 指令 |
| `AT_MQTT_BROKER_AUTH_CLIENT` | y（开 MQTT 时） | Broker 认证客户端（需 mqtt_ca 分区） |
| `AT_MQTT_CLIENT_AUTH_BROKER` | y | 客户端认证 Broker（需 mqtt_cert/key 分区） |

## HTTP

| Kconfig | 默认 | 说明 |
|---|---|---|
| `AT_HTTP_COMMAND_SUPPORT` | n | HTTP 指令 |
| `AT_HTTPS_SERVER_AUTH_CLIENT` | n | HTTPS 服务端认证客户端 |
| `AT_HTTPS_CLIENT_AUTH_SERVER` | n | HTTPS 客户端认证服务端 |
| `AT_HTTP_TX_BUFFER_SIZE` | 2048 | HTTP 发送缓冲 |
| `AT_HTTP_RX_BUFFER_SIZE` | 2048 | HTTP 接收缓冲 |

> 开 HTTPS 时务必把 `AT_PROCESS_TASK_STACK_SIZE` 调到 4096 以上。

## WebSocket

| Kconfig | 默认 | 说明 |
|---|---|---|
| `AT_WS_COMMAND_SUPPORT` | n | WebSocket 指令 |
| `AT_WSS_SERVER_AUTH_CLIENT` | y（开 WS 时） | 服务端认证客户端 |
| `AT_WSS_CLIENT_AUTH_SERVER` | y | 客户端认证服务端 |

## BLE / Classic BT

| Kconfig | 默认 | 说明 |
|---|---|---|
| `AT_BLE_COMMAND_SUPPORT` | y（C2 为 n） | BLE 指令（依赖 BT_ENABLED） |
| `AT_BLE_HID_COMMAND_SUPPORT` | y | BLE HID（依赖 BT_BLUEDROID） |
| `AT_BLUFI_COMMAND_SUPPORT` | y | BluFi |
| `AT_BLE_OTA_COMMAND_SUPPORT` | n | BLE OTA |
| `AT_BT_COMMAND_SUPPORT` | n | Classic BT（仅 ESP32，select BT_CLASSIC） |
| `AT_BT_SPP_COMMAND_SUPPORT` | y（开 BT 时） | BT SPP |
| `AT_BT_A2DP_COMMAND_SUPPORT` | n | BT A2DP（select BT_A2DP_ENABLE） |
| `AT_I2S_LRCK_PIN` | 18 | I2S LRCK GPIO（A2DP） |
| `AT_I2S_BCK_PIN` | 26 | I2S BCK GPIO |
| `AT_I2S_DATA_PIN` | 25 | I2S DATA GPIO |

## Ethernet（仅 ESP32）

| Kconfig | 默认 | 说明 |
|---|---|---|
| `AT_ETHERNET_SUPPORT` | n | Ethernet（仅 ESP32） |
| `PHY_MODEL`（choice） | IP101 | `PHY_IP101`/`PHY_RTL8201`/`PHY_DP83848`/`PHY_LAN8720` |

## 文件系统

| Kconfig | 默认 | 说明 |
|---|---|---|
| `AT_FS_COMMAND_SUPPORT` | n | FS 指令 |
| `AT_FS_TYPE`（choice） | LITTLEFS | `AT_FS_LITTLEFS`（推荐，分区标签 `fs_storage`）/ `AT_FS_FATFS`（`fs_storage`）/ `AT_FS_FATFS_LEGACY`（`fatfs`，仅升级用） |

## 其它

| Kconfig | 默认 | 说明 |
|---|---|---|
| `AT_DRIVER_COMMAND_SUPPORT` | n | 驱动指令 |
| `AT_WEB_SERVER_SUPPORT` | n | Web Server（`AT+WEBSERVER`） |
| `AT_WEB_USE_FS` | n | 用 FS 存 HTML（依赖 FS） |
| `AT_WEB_CAPTIVE_PORTAL_ENABLE` | n | Captive portal（建议 LWIP_MAX_SOCKETS ≥14） |
| `AT_WEB_ROOT_DIR` | "/" | Web 根目录 |

## 文件系统相关宏（esp_at_types.h）

| Kconfig | 挂载点宏 | 分区标签宏 |
|---|---|---|
| `CONFIG_AT_FS_LITTLEFS` | `AT_FS_MOUNT_POINT` = `"/littlefs"` | `AT_FS_PARTITION_LABEL` = `"fs_storage"` |
| `CONFIG_AT_FS_FATFS` | `"/fatfs"` | `"fs_storage"` |
| `CONFIG_AT_FS_FATFS_LEGACY` | `"/fatfs"` | `"fatfs"` |

## 常用 sdkconfig.defaults 片段示例

```
# 启用 HTTP 并放大任务栈
CONFIG_AT_HTTP_COMMAND_SUPPORT=y
CONFIG_AT_PROCESS_TASK_STACK_SIZE=4096

# 启用 WebSocket，关闭 mDNS
CONFIG_AT_WS_COMMAND_SUPPORT=y
CONFIG_AT_MDNS_COMMAND_SUPPORT=n

# 启用 MQTT
CONFIG_AT_MQTT_COMMAND_SUPPORT=y

# 文件系统用 LittleFS
CONFIG_AT_FS_COMMAND_SUPPORT=y
CONFIG_AT_FS_LITTLEFS=y
```
