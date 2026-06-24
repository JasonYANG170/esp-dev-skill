# esp-thread-br 陷阱合集

> 汇总 SKILL.md "Critical Pitfalls" 之外的补充陷阱，全部源自仓库 docs/qa、codelab、README 与源码注释。

## A. 架构与启动

### A1. Backbone Netif 必须在 border_router_init 前绑定
`esp_openthread_set_backbone_netif(get_example_netif())` 必须在 `esp_openthread_border_router_init()` 之前，否则 Wi-Fi/Ethernet 与 Thread 不互通。见 `border_router_launch.c::ot_br_init`。

### A2. Ethernet backbone 不能手动起
`!OPENTHREAD_BR_AUTO_START && EXAMPLE_CONNECT_ETHERNET` 在 `esp_ot_br.c` 中直接 `#error`。Ethernet 必须 `OPENTHREAD_BR_AUTO_START=y`。

### A3. AUTO_START 下 Wi-Fi 连不上会重启
`wifi_config_save_and_connect` 失败时 `esp_restart()` 重试。配错 SSID/PSK 会导致循环重启。

## B. 通信接口

### B1. ESP32-S3 UART 引脚驱动电流
GPIO17/GPIO18 驱动电流不一致（见 ESP32-S3 TRM 6.12），作 host UART RX/TX 易丢包 → `Wait for response timeout`。推荐 GPIO4/GPIO5（qa.rst 5.2）。

### B2. SPI 模式必须两端同步
只在 BR 开 `OPENTHREAD_RADIO_SPINEL_SPI` 而 ot_rcp 端不开 `OPENTHREAD_RCP_SPI`，握手失败。

### B3. ot_rcp 波特率与 ot-br-posix 不一致
`ot_rcp` 默认 460800，ot-br-posix `otbr-agent` 默认 115200。把 ESP ot_rcp 接 OTBR 时需改 `OTBR_AGENT_OPTS` 的 `uart-baudrate=460800`（qa.rst 5.3）。

## C. 网络/路由

### C1. OT 层 ping BR 自身 Wi-Fi 地址必丢包
设计行为：回包目的=BR 自己，lwIP 不再上送 OT 层（qa.rst 5.1）。要在 lwIP 层 ping。

### C2. 组播仅转发 admin-local(ff04) 及以上
link-local / realm-local 组播不跨 BR 转发。

### C3. Linux 主机必须开 accept_ra
`accept_ra=2`、`accept_ra_rt_info_max_plen=128`，否则收不到 BR 的 RA，无路由。

### C4. LWIP_IPV6_NUM_ADDRESSES 与 IDF 版本绑定
v5.3.1/v5.4 及之后 = 12，更早 = 8。BR 库内部固定。

## D. RCP 更新 / OTA

### D1. rcp_fw SPIFFS 必须挂载
`CONFIG_AUTO_UPDATE_RCP=y` 但没 `esp_vfs_spiffs_register(rcp_fw_conf)`，`esp_rcp_update()` 报 `ESP_ERR_NOT_FOUND`。

### D2. 手动 otrcp 前必须停 Thread + ifconfig down
否则 RCP 复位导致 Thread 网络崩溃。

### D3. OTA 镜像非裸 app
`ota_with_rcp_image` 是 bundle（filetag 0~5 + 0xff 头），不能直接 esptool 烧，必须运行时 `ota download` 触发解析。

### D4. 自签证书 CN 必须匹配
`openssl req` 的 Common Name 必须等于 `ota download` URL 的主机名，否则 TLS 校验失败；证书要替换 `server_certs/ca_cert.pem` 并 `fullclean`。

### D5. 4MB Flash 不够 OTA
官方 partitions 用 8MB；4MB 变体需把 `ota_0/ota_1` 改 1500K，且未必够未来 OTA with RCP。

## E. Web GUI / 服务发现

### E1. Web 资源依赖 web_storage SPIFFS
`esp_br_web_start("/spiffs")` 前必须挂 `web_storage` 分区，否则 GUI/REST 404。

### E2. mDNS 服务上限
默认 `CONFIG_MDNS_MAX_SERVICES=200`，服务多时若不调大会丢发现结果。

### E3. SRP 服务要先 autostart
`ot srp client service add` 后还需 `ot srp client autostart enable`，否则 BR SRP server 收不到。

## F. 构建 / 配置

### F1. eventfd 数量
启用 SPI/TREL 时 `max_eventfd` 要 +1，否则 `esp_vfs_eventfd_register` 后不够用。

### F2. RCP 镜像 filetag 顺序固定
`create_ota_image.py` 按固定 filetag 顺序生成，自定义镜像生成脚本必须遵守 `esp_rcp_firmware.h` 的 enum。

### F3. ESP32-C5 单天线
ESP32-C5 仅一根天线，Wi-Fi 与 Thread 不能同时收发，必须外接 RCP（`README_standalone_RCP.md` Note）。

## G. 升级回滚

### G1. verified flag 决定 idx
verified=true → idx=seq；verified=false → idx=1−seq。改错会刷错槽位。

### G2. 升级失败自动回滚
新 RCP 启动失败，BR 自动用备份镜像回滚。若不断重启说明新镜像本身有问题，应检查 ot_rcp 构建。
