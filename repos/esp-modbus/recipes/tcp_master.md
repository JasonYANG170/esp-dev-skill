# Modbus TCP 主站

> **适用摘要**: 在 ESP32 系列芯片（Wi-Fi 或以太网）上构建 Modbus TCP 主站，覆盖 netif 初始化、从站 IP 地址表（含 MDNS / 静态 IP / IPv6）、`mbc_master_create_tcp`、按 CID 轮询、销毁。

## 触发意图

- "Modbus TCP master"
- "Modbus TCP 主站"
- "Modbus TCP 主机轮询"
- "MDNS 解析 Modbus 从站"
- "Modbus IPv6 主机"

## 前置条件

| 条件 | 要求 |
|---|---|
| 参考示例 | `espressif-repos/esp-modbus/examples/tcp/mb_tcp_master/` |
| 共享参数组件 | `espressif-repos/esp-modbus/examples/mb_example_common/` |
| Kconfig | `CONFIG_FMB_COMM_MODE_TCP_EN=y`；`CONFIG_FMB_MDNS_INTEGRATION_ENABLE=y`（用 MDNS 名字时）；`CONFIG_FMB_MASTER_TIMEOUT_MS_RESPOND` |
| 网络 | `example_connect()` 已建立连接；`CONFIG_EXAMPLE_CONNECT_IPV6=y` 时用 `MB_IPV6` |
| 从站表 | `slave_ip_address_table[]` 中条目数 = 数据字典里不同 UID 的数量，且以 `NULL` 结尾 |

## 分步说明

### 1. 启动网络服务

来自 `examples/tcp/mb_tcp_master/main/tcp_master.c::init_services`：

```c
nvs_flash_init();                 // 失败则 erase 后重试
esp_netif_init();
esp_event_loop_create_default();
example_connect();                // Wi-Fi 或以太网
#if CONFIG_EXAMPLE_CONNECT_WIFI
esp_wifi_set_ps(WIFI_PS_NONE);
#endif
```

### 2. 定义从站 IP 地址表

每行格式 `UID;host_or_ip;port`，支持 MDNS 主机名、IPv4、IPv6 字面量。**最后一项必须为 `NULL`**。

```c
char *slave_ip_address_table[] = {
    "01;mb_slave_tcp_01;502",                                   // MDNS 名
    "200;mb_slave_tcp_c8;1502",
    "35;192.168.32.54;1502",                                    // IPv4 字面量
    "12:2001:0db8:85a3:0000:0000:8a2e:0370:7334:502",           // IPv6:UID:addr:port（见 docs）
    NULL
};
```

### 3. 创建 TCP 主站对象

```c
#define MB_TCP_PORT  (CONFIG_FMB_TCP_PORT_DEFAULT)
static void *master_handle = NULL;

mb_communication_info_t tcp_master_config = {
    .tcp_opts.port = MB_TCP_PORT,
    .tcp_opts.mode = MB_TCP,
    .tcp_opts.addr_type = MB_IPV4,                                  // 或 MB_IPV6
    .tcp_opts.ip_addr_table = (void *)slave_ip_address_table,
    .tcp_opts.uid = 0,
    .tcp_opts.start_disconnected = false,                           // false = 启动前连上所有从站
    .tcp_opts.response_tout_ms = CONFIG_FMB_MASTER_TIMEOUT_MS_RESPOND,
    .tcp_opts.ip_netif_ptr = (void *)get_example_netif(),
};
ESP_ERROR_CHECK(mbc_master_create_tcp(&tcp_master_config, &master_handle));
```

### 4. 注册数据字典并启动

```c
ESP_ERROR_CHECK(mbc_master_set_descriptor(master_handle,
                                          &device_parameters[0],
                                          num_device_parameters));
ESP_ERROR_CHECK(mbc_master_start(master_handle));
```

> 当 `start_disconnected = false` 时，`mbc_master_start` 会尝试连接表中所有从站；连不上会记日志但不会阻断启动。

### 5. 按 CID 轮询

```c
const mb_parameter_descriptor_t *pdescr = NULL;
uint8_t temp_data[4] = {0};
uint8_t type = 0;

for (uint16_t cid = 0; cid < num_device_parameters; cid++) {
    if (mbc_master_get_cid_info(master_handle, cid, &pdescr) == ESP_OK) {
        if (mbc_master_get_parameter(master_handle, pdescr->cid, temp_data, &type) == ESP_OK) {
            ESP_LOGI(TAG, "CID %u (%s) = 0x%08" PRIx32,
                     pdescr->cid, pdescr->param_key, *(uint32_t *)temp_data);
        }
    }
    vTaskDelay(pdMS_TO_TICKS(1));   // 轮询间隔
}
```

### 6. 销毁（顺序：删主站 → 断网络）

```c
mbc_master_delete(master_handle);
example_disconnect();
esp_event_loop_delete_default();
esp_netif_deinit();
nvs_flash_deinit();
```

## MDNS 与地址解析

- `CONFIG_FMB_MDNS_INTEGRATION_ENABLE=y`（默认开）时，协议栈自动用 MDNS 解析表中的主机名（如 `mb_slave_tcp_01`）。
- 关闭 MDNS 但仍使用 MDNS 名时，日志会报：`E ... mb_port.tcp.master: ... mdns service is not supported.`，此时应改用字面量 IP。
- 每个从站需要在其主机名上启用 MDNS 服务，主站才能解析。

## 常见错误

| 错误 | 原因 | 解决方法 |
|---|---|---|
| 启动时报 `mdns service is not supported` | 关了 MDNS 集成却用了 MDNS 名 | 开 `CONFIG_FMB_MDNS_INTEGRATION_ENABLE=y` 或改用字面量 IP |
| `ESP_ERR_NOT_FOUND`（连接不上从站） | 从站表 UID 数与数据字典不一致，或网络不通 | 表条目数 = 字典中不同 UID 数；确认从站在线、IP/端口正确 |
| 表越界 / 崩溃 | 表缺少 `NULL` 终止 | 最后一项必须为 `NULL` |
| IPv6 解析失败 | 未启用 IPv6 / `addr_type` 写成 `MB_IPV4` | `CONFIG_EXAMPLE_CONNECT_IPV6=y` 且 `.tcp_opts.addr_type = MB_IPV6` |
| `ip_netif_ptr` 相关失败 | netif 未就绪就创建对象 | 先 `example_connect()` 再 `mbc_master_create_tcp` |

## 参考

- `espressif-repos/esp-modbus/examples/tcp/mb_tcp_master/main/tcp_master.c`
- `espressif-repos/esp-modbus/docs/en/port_initialization.rst`（TCP master 配置与从站表说明）
- `espressif-repos/esp-modbus/docs/en/master_api_overview.rst`
- `recipes/data_dictionary.md`、`recipes/serial_master.md`（CID 轮询逻辑相同）
