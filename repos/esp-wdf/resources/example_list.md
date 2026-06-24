# ESP-WDF 示例索引（example_list）

> 所有路径均为 esp-wdf 仓库内的真实路径，描述基于示例源码与 README。`cd` 到对应目录后执行 `idf.py build`（或 `idf.py build aot`）。

## 基础

| 示例路径 | 描述 | 入口形态 |
|---|---|---|
| `examples/hello_world` | 最小示例，`main()` 中 `printf("Hello World!\n")`，验证编译链与 WASM 产物 | 标准 main |
| `examples/coremark` | CoreMark 基准测试，评估 WASM/AOT 性能 | 标准 main |

## simple（WAMR App Framework）

| 示例路径 | 描述 |
|---|---|
| `examples/simple/timer` | 用 `api_timer_create`/`api_timer_restart` 创建周期定时器，回调中 `printf`（需 `EXPORT_TIMER`） |
| `examples/simple/event_publisher` | 用 `attr_container` + `api_publish_event` 周期发布 `"alert/overheat"` 事件 |
| `examples/simple/event_subscriber` | 用 `api_subscribe_event` 订阅事件并在回调中 `attr_container_dump` |
| `examples/simple/request_handler` | 用 `api_register_resource_handler` 注册 `/url1`、`/url2`，构造 `attr_container` 响应并 `api_response_send` |
| `examples/simple/request_sender` | 用 `init_request` + `api_send_request` 向 `/app/request_handler/url1` 异步发请求，回调打印响应 |

## peripherals（VFS/ioctl 外设）

| 示例路径 | 描述 |
|---|---|
| `examples/peripherals/gpio/gpio_simple` | `open("/dev/gpio/<pin>")` + `ioctl(GPIOCSCFG)` 配置 + `write` 翻转电平 |
| `examples/peripherals/uart/uart_simple` | `open("/dev/uart/0" 或 "/dev/usbserjtag")` + `write` 发送字符串 |
| `examples/peripherals/i2c/i2c_bh1750` | I2C 主机读 BH1750 光照传感器：`I2CIOCSCFG` 配置 + `I2CIOCRDWR`/`I2CIOCEXCHANGE` 收发 |
| `examples/peripherals/i2c/i2c_tt21100` | I2C + GPIO READY 门控读 TT21100 触控芯片：`open("/dev/i2c/0")`+`open("/dev/gpio/<ready>")`，轮询 READY 低有效时两步 I2C 读（先读 2 字节长度字，再读该长度的触点/按键报告）；自带 Kconfig `TT21100_READY_PIN_NUM`、`I2C_SDA_SCL_PIN_PULLUP`（见 `recipes/i2c_touch_gpio_irq.md`） |
| `examples/peripherals/spi/spi_master_simple` | SPI 主机：`SPIIOCSCFG` 配置 + `SPIIOCEXCHANGE` 周期发送 |
| `examples/peripherals/spi/spi_slave_simple` | SPI 从机收发 |
| `examples/peripherals/ledc/ledc_simple` | LEDC：`LEDCIOCSCFG` 配置通道 + `LEDCIOCSSETDUTY` 渐变占空比 |

## file_system / multi_thread

| 示例路径 | 描述 |
|---|---|
| `examples/file_system` | VFS 文件读写：`open("/storage/hello.txt")` + `write/read/lseek/close` |
| `examples/multi_thread` | POSIX 线程：`pthread_create/join` + `pthread_mutex_t`/`pthread_cond_t` |

## protocols（网络）

| 示例路径 | 描述 |
|---|---|
| `examples/protocols/sockets/socket-api/send_recv` | BSD socket 收发（通用） |
| `examples/protocols/sockets/socket-api/tcp_client` | TCP 客户端（含 setsockopt 演示） |
| `examples/protocols/sockets/socket-api/tcp_server` | TCP 服务端 |
| `examples/protocols/sockets/tcp_client` | 简化 TCP 客户端（`__wasi__` 下含 `wasi_socket_ext.h`） |
| `examples/protocols/sockets/tcp_server` | 简化 TCP 服务端 |
| `examples/protocols/esp_http_client` | ESP HTTP Client：GET/POST、证书、事件处理 |
| `examples/protocols/mqtt/generic` | ESP-MQTT：连接、订阅 `/topic/qos0`、`/topic/qos1`、发布 |

## provisioning / rainmaker（云与配网）

| 示例路径 | 描述 |
|---|---|
| `examples/provisioning/wifi_prov_mgr` | Wi-Fi 配网（softap/ble/qr），通过 Wi-Fi Provisioning WASM 适配 |
| `examples/rainmaker/switch` | ESP RainMaker 开关设备：`app_wifi_init` + `esp_rmaker_node_init` + 标准 Switch 设备 + 写回调 |

## gui

| 示例路径 | 描述 |
|---|---|
| `examples/gui/lv_demos` | LVGL 演示入口（README 指引，运行于 ESP-WASMachine，含 LVGL WASM 适配 API） |
