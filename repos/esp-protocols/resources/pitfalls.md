# esp-protocols 常见陷阱汇总

> 与 `SKILL.md` 的 "Critical Pitfalls" 互补，按组件组织，给可快速查阅的清单。所有结论来自仓库 docs（尤其 `docs/esp_modem/en/README.rst` 的 Known issues）与头文件注释。

## esp_modem

### 1. 创建顺序：netif → DCE
必须先 `esp_netif_new(&ESP_NETIF_DEFAULT_PPP())`，再把 netif 句柄传给 `esp_modem_new` / `esp_modem_new_dev`。NULL netif 会失败。

### 2. 文本输出缓冲截断
`esp_modem_get_imsi/imei/operator_name/module_name` 与 `esp_modem_at/at_raw` 的输出缓冲必须 >= `ESP_MODEM_C_API_STR_BUF_SIZE`（= `CONFIG_ESP_MODEM_C_API_STR_MAX`，默认 128），否则被截断。需更长输出时调大 Kconfig。

### 3. DATA 模式下不能直接发 AT
切到 `ESP_MODEM_MODE_DATA` 后 UART 是 PPP 流。要临时发 AT：
- 纯 COMMAND/DATA：用 `esp_modem_pause_net(dce, true/false)`，**不要**为发一条 AT 把整个 PPP 断掉。
- 或用 CMUX（命令/数据分通道）。

### 4. 硬件流控要双端配置
配置 `dte_config.uart_config.flow_control = ESP_MODEM_FLOW_CONTROL_HW` 后，还要 `esp_modem_set_flow_control(dce, 2, 2)` 通知模组侧（2/2 = HW）。

### 5. OTA over HTTPS 不稳定（UART buffer overflow）
UART 是中断驱动，TLS 计算密集时来不及处理收包。缓解顺序（来自 docs Known issues）：
1. UART ISR 放入 IRAM
2. 增大 `uart_config.rx_buffer_size`
3. 提高 `dte_config.task_priority`
4. 启用硬件流控
5. 仍不行：参考 `test_app/esp_modem/test/target_ota` 的两步式 OTA（先收齐再喂 mbedTLS）

### 6. CMUX 设备兼容性（来自 docs Known issues）
- **SIM7000 不支持 CMUX**，不要尝试 `ESP_MODEM_MODE_CMUX`。
- **A76xx 系列**用 2 字节 CMUX payload，会触发缓冲溢出：设 `CONFIG_ESP_MODEM_CMUX_DEFRAGMENT_PAYLOAD=n`（副作用：命令回复偶发分片，需重试）。
- **A7670** 退出 CMUX 异常，应用 docs 中给出的 `esp_modem_cmux.cpp` 补丁（识别 0xEF DISC 帧）。
- **CAVLI C16QS** 进入 CMUX 时 SABM 响应 0x3F（非 UA/DM），应用 docs 给出的进入序列补丁。

### 7. PPP_ESCAPE_BEFORE_EXIT 因设备而异
`CONFIG_ESP_MODEM_PPP_ESCAPE_BEFORE_EXIT=y` 会在 PPP→CMD 时发 `+++`，对 Quectel 更稳，但可能让 SIMCOM 出错。按模组选择。

### 8. Production vs Development 模式
默认 Production（用预生成 AT 命令头，编译快、IDE 友好）。仅当要修改 `esp_modem_command_declare.inc` 时才开 `CONFIG_ESP_MODEM_ENABLE_DEVELOPMENT_MODE=y`。**添加新模组用 custom module 即可，无需 development mode。**

### 9. URC 处理需显式启用
`esp_modem_set_urc()` 仅在 `CONFIG_ESP_MODEM_URC_HANDLER=y` 时可用。

## mdns

### 1. 必须先 init + hostname_set 再 service_add
未 `mdns_init()` 或未设 hostname 就 `mdns_service_add()` 会失败。

### 2. 查询结果必须 free
每次 `mdns_query*` 返回的链表用完必须 `mdns_query_results_free()`，否则内存泄漏。

### 3. 多实例需开 MULTIPLE_INSTANCE
要 `mdns_service_add` 多个同类型（如多个 `_http._tcp`）实例，需 `CONFIG_MDNS_MULTIPLE_INSTANCE=y`。

### 4. TXT 项 service_type/proto 必须与 add 一致
`mdns_service_txt_item_set` 等的 service_type/proto 必须和 `mdns_service_add` 时完全一致，否则找不到服务。

### 5. 任务优先级勿高于系统任务
`CONFIG_MDNS_TASK_PRIORITY` 默认 1；设太高会干扰系统任务（编译期会警告/报错）。

## esp_websocket_client

### 1. 事件回调内禁调 stop/destroy/close
这些 API 会与事件循环自死锁。用信号量/队列通知外部任务，在回调外销毁。

### 2. wss 必须配置服务器验证
- 证书包：`cfg.crt_bundle_attach = esp_crt_bundle_attach`（需 `MBEDTLS_CERTIFICATE_BUNDLE`）
- 自定义 CA：`cfg.cert_pem`（PEM，需 NUL 结尾）
- 双向认证：另设 `client_cert`/`client_key`（或 DS 外设 `client_ds_data`，需 `CONFIG_ESP_TLS_USE_DS_PERIPHERAL`）

### 3. 单帧超 buffer_size 用分片
`send_text/send_bin` 单次数据 > `buffer_size`（默认 1024）会被拒。用 `send_*_partial` + `send_cont_msg` + `send_fin` 分片发送。

### 4. 自动重连
默认开启。`disable_auto_reconnect=true` 自行管理；`reconnect_timeout_ms` 调整重连间隔；改间隔的最佳时机是 `WEBSOCKET_EVENT_DISCONNECTED`/`ERROR`。

### 5. ping/pong 超时
`ping_interval_sec`（默认 10s）、`pingpong_timeout_sec`；网络抖动可 `disable_pingpong_discon=true`。

### 6. 数据事件可能分多次到达
payload > buffer 会通过多个 `WEBSOCKET_EVENT_DATA` 投递，用 `payload_len`/`payload_offset` 拼装。

## eppp_link

### 1. 两端 IP 必须交叉
用 `EPPP_DEFAULT_SERVER_CONFIG()` / `EPPP_DEFAULT_CLIENT_CONFIG()`，不要手写把两端 our/their 写重。

### 2. 传输层两端必须一致
UART/SPI/SDIO/ETH 任选其一，两端 menuconfig 选择必须相同，引脚/波特/频率对应。

### 3. 自定义非 IP 数据用 add_channels
默认只跑 IP 流；要传 802.11 帧等自定义数据用 `eppp_add_channels()`。

## esp_dns

### 1. init/cleanup 协议必须匹配
`esp_dns_init_doh` → `esp_dns_cleanup_doh`，依此类推。混用会出错。

### 2. DoT/DoH 需证书
`tls_config.cert_pem` 或 `tls_config.crt_bundle_attach` 二选一；否则 TLS 握手失败。

### 3. DoH 的 url_path
服务器通常要求 `/dns-query` 等特定路径；填错会解析失败。
