# Modbus TCP 从站

> **适用摘要**: 在 ESP32 系列芯片（Wi-Fi 或以太网）上构建 Modbus TCP 从站，覆盖 netif 初始化、`mbc_slave_create_tcp`、多连接、keep-alive、事件循环与销毁。

## 触发意图

- "Modbus TCP slave"
- "Modbus TCP 从站"
- "Modbus 以太网从机"
- "Modbus TCP 多连接"
- "Wi-Fi Modbus 从机"

## 前置条件

| 条件 | 要求 |
|---|---|
| 参考示例 | `espressif-repos/esp-modbus/examples/tcp/mb_tcp_slave/` |
| 共享参数组件 | `espressif-repos/esp-modbus/examples/mb_example_common/` |
| Kconfig | `CONFIG_FMB_COMM_MODE_TCP_EN=y`（依赖 `LWIP_ENABLE`）；`CONFIG_FMB_TCP_PORT_DEFAULT`；`CONFIG_FMB_TCP_PORT_MAX_CONN`；`CONFIG_FMB_TCP_KEEP_ALIVE_TOUT_SEC` |
| 网络 | `example_connect()` 完成 Wi-Fi/以太网连接，`get_example_netif()` 可用 |

## 分步说明

### 1. 启动网络服务（应用层）

来自 `examples/tcp/mb_tcp_slave/main/tcp_slave.c`：

```c
static esp_err_t init_services(void) {
    esp_err_t result = nvs_flash_init();
    if (result == ESP_ERR_NVS_NO_FREE_PAGES || result == ESP_ERR_NVS_NEW_VERSION_FOUND) {
        ESP_ERROR_CHECK(nvs_flash_erase());
        result = nvs_flash_init();
    }
    result = esp_netif_init();
    result = esp_event_loop_create_default();
    result = example_connect();           // Wi-Fi 或以太网（menuconfig 选择）
#if CONFIG_EXAMPLE_CONNECT_WIFI
    result = esp_wifi_set_ps(WIFI_PS_NONE); // 降低 Wi-Fi 省电带来的延迟
#endif
    return result;
}
```

### 2. 创建 TCP 从站对象

```c
#define MB_TCP_PORT_NUMBER  (CONFIG_FMB_TCP_PORT_DEFAULT)
#define MB_SLAVE_ADDR       (CONFIG_MB_SLAVE_ADDR)
static void *slave_handle = NULL;

mb_communication_info_t tcp_slave_config = {
    .tcp_opts.port = MB_TCP_PORT_NUMBER,           // 默认 502
    .tcp_opts.mode = MB_TCP,
    .tcp_opts.addr_type = MB_IPV4,                 // 或 MB_IPV6
    .tcp_opts.ip_addr_table = NULL,                // 绑定任意地址
    .tcp_opts.ip_netif_ptr = (void *)get_example_netif(),
    .tcp_opts.uid = MB_SLAVE_ADDR,
};
ESP_ERROR_CHECK(mbc_slave_create_tcp(&tcp_slave_config, &slave_handle));
```

### 3. 注册寄存器区域（与串行从站一致）

```c
mb_register_area_descriptor_t reg_area = {0};
reg_area.type = MB_PARAM_HOLDING;
reg_area.start_offset = HOLD_OFFSET(holding_data0);
reg_area.address = (void *)&holding_reg_params.holding_data0;
reg_area.size = (HOLD_OFFSET(holding_data4) - HOLD_OFFSET(holding_data0)) << 1;
reg_area.access = MB_ACCESS_RW;
ESP_ERROR_CHECK(mbc_slave_set_descriptor(slave_handle, reg_area));
// 同理注册 INPUT / COIL / DISCRETE ...
```

### 4. 启动并进入事件循环

```c
mbc_slave_start(slave_handle);

mb_param_info_t reg_info;
for (;;) {
    (void)mbc_slave_check_event(slave_handle, MB_READ_WRITE_MASK);
    esp_err_t err = mbc_slave_get_param_info(slave_handle, &reg_info, MB_PAR_INFO_GET_TOUT);
    if (err == ESP_OK && (reg_info.type & (MB_EVENT_HOLDING_REG_WR|MB_EVENT_HOLDING_REG_RD))) {
        ESP_LOGI(TAG, "HOLDING %s ADDR:%u SIZE:%u",
                 (reg_info.type & MB_READ_MASK) ? "READ":"WRITE",
                 (unsigned)reg_info.mb_offset, (unsigned)reg_info.size);
    }
}
```

### 5. 销毁

```c
mbc_slave_delete(slave_handle);
example_disconnect();
esp_event_loop_delete_default();
esp_netif_deinit();
nvs_flash_deinit();
```

## 多连接与 keep-alive 说明

- 最大并发连接数由 `CONFIG_FMB_TCP_PORT_MAX_CONN`（默认 5）控制，超出会被拒绝。
- keep-alive 超时由 `CONFIG_FMB_TCP_KEEP_ALIVE_TOUT_SEC`（默认 4 秒）控制；超时后从站关闭死连接，日志形如：
  `E ... mb_port.tcp.slave: ... communication fail, err= -11`
- 每多一个主站连接，从站的请求处理时间都会变长，进而抬高整网 RTT。

## 常见错误

| 错误 | 原因 | 解决方法 |
|---|---|---|
| 从站启动后主站连不上 | `ip_netif_ptr` 为 NULL 或 netif 未 up | 先 `example_connect()` 再创建对象；传入 `get_example_netif()` |
| 主站偶发超时 | keep-alive 超时太短或主站 RTT 较大 | keep-alive 应大于主站超时；必要时调大两端超时 |
| 日志 `handling time ... exceeds slave response time in master` | 竞态：从站处理时间 > 主站超时 | 降低从站负载或调大 `CONFIG_FMB_MASTER_TIMEOUT_MS_RESPOND` |
| 端口被占 | 502 已被其他服务占用 | 改 `CONFIG_FMB_TCP_PORT_DEFAULT` 或 `.tcp_opts.port` |
| Wi-Fi 模式下延迟抖动大 | Wi-Fi 省电模式 | `esp_wifi_set_ps(WIFI_PS_NONE)` |

## 参考

- `espressif-repos/esp-modbus/examples/tcp/mb_tcp_slave/main/tcp_slave.c`
- `espressif-repos/esp-modbus/docs/en/slave_api_overview.rst`
- `espressif-repos/esp-modbus/docs/en/overview_messaging_and_mapping.rst`（多连接 / keep-alive / 竞态章节）
- `recipes/serial_slave.md`（寄存器区域注册逻辑相同）
