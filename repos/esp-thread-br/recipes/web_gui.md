# Web GUI 与 REST API（含 Home Assistant）

> **适用摘要**: 启用 BR Web Server，通过浏览器图形界面发现/组网/查状态，访问 REST API，并接入 Home Assistant。

## 触发意图
- "Web GUI"
- "REST API"
- "/node /topology available_network"
- "Home Assistant 接入 Thread"

## 前置条件

| 条件 | 要求 |
|---|---|
| Kconfig | `CONFIG_OPENTHREAD_BR_START_WEB=y`（自动 `select OPENTHREAD_COMMISSIONER/JOINER`） |
| 分区 | `web_storage` SPIFFS 已挂载 |
| 网络 | 主机与 BR 同 Wi-Fi/Ethernet |
| 参考 | `docs/en/codelab/web-gui.rst`、`docs/en/codelab/home_assistant.rst` |

## 分步说明

### 1. 启用 Web Server

```bash
idf.py menuconfig
# ESP Thread Border Router Example ->
#   [*] Enable the web server in Thread Border Router.   # OPENTHREAD_BR_START_WEB
```

### 2. 在 app_main 启动 Web Server

```c
#if CONFIG_OPENTHREAD_BR_START_WEB
    // 必须先挂 web_storage SPIFFS
    esp_vfs_spiffs_conf_t web_server_conf = {
        .base_path = "/spiffs", .partition_label = "web_storage",
        .max_files = 10, .format_if_mount_failed = false};
    ESP_RETURN_ON_ERROR(esp_vfs_spiffs_register(&web_server_conf), TAG, "mount web storage");
#endif

// ... 在 app_main:
#if CONFIG_OPENTHREAD_BR_START_WEB
    esp_br_web_start("/spiffs");   // components/esp_ot_br_server/include/esp_br_web.h
#endif
```

### 3. 访问 GUI

启动日志会打印地址：
```
otbr_web: <========server start========>
otbr_web: http://192.168.200.98
otbr_web: <============================>
```
浏览器打开 `http://<BR-IPv4>:80/index.html`，提供 Discover / Join / Form / Settings / Status / Topology 功能。

### 4. Thread REST API（兼容 ot-br-posix）

GET 请求（HTTP 端口 80）：

| API | 含义 |
|---|---|
| `/node` | 节点综合信息（NetworkName、ExtPanId、State、Rloc16、NumOfRouter、LeaderData） |
| `/node/rloc` `/node/rloc16` | RLOC 地址 / RLOC16 |
| `/node/ext-address` | 扩展地址 |
| `/node/state` | 节点状态 |
| `/node/network-name` | 网络名 |
| `/node/leader-data` | Leader 数据 |
| `/node/num-of-router` | 路由器数量 |
| `/node/ext-panid` | 扩展 PAN ID |
| `/node/ba-id` | Backbone router ID |
| `/node/dataset/active` `/node/dataset/pending` | dataset |
| `/diagnostics` | 诊断 |

示例：
```bash
curl http://192.168.200.98:80/node
# {"NetworkName":"OpenThread-4c68","ExtPanId":"f4f9437404558d34","State":4,"Rloc16":14336,...}
```

### 5. Web GUI 扩展 API

| API | 方法 | 含义 |
|---|---|---|
| `/available_network` | GET | 扫描周围 Thread 网络 |
| `/get_properties` | GET | 查 Thread 状态（地址、PAN、RCP 版本等） |
| `/node_information` | GET | 节点信息 |
| `/topology` | GET | 拓扑（路由表、子表、计数器） |
| `/join_network` | POST | 加入网络（networkKey / pskd 两种） |
| `/form_network` | POST | 形成网络 |
| `/add_prefix` `/delete_prefix` | POST | 增删 IPv6 前缀 |

`/form_network` JSON：
```json
{
  "networkName":"OpenThread-0x99", "networkKey":"00112233445566778899aabbccddeeff",
  "panId":"0x1234", "channel":16, "extPanId":"1111111122222222",
  "passphrase":"j01Nme", "prefix":"fd11:22::", "defaultRoute":1
}
```

### 6. 接入 Home Assistant

- HA App：`Settings → Devices & services → ADD INTEGRATION` → 添加 **OpenThread Border Router** 与 **Thread**。
- 在 OpenThread Border Router 集成点 `ADD SERVICE`，URL 填 `http://<BR-IPv4>:80`。
- 在 Thread 集成点 `CONFIGURE`，把 BR 选为首选网络，再 `Send Credentials to Home Assistant`。
- 之后可添加 Matter over Thread 设备。

## 常见错误

| 错误 | 原因 | 解决方法 |
|---|---|---|
| 打不开 GUI | 未启用 Web Server | `OPENTHREAD_BR_START_WEB=y` 且 `esp_br_web_start()` 调用 |
| Web 资源 404 | `web_storage` SPIFFS 未挂 | `init_spiffs()` 中注册 `web_storage` |
| HA 加服务失败 | BR 与 HA 不在同一局域网 | 同 Wi-Fi/Ethernet；URL 含端口 80 |
| REST 返回旧数据 | GUI 缓存 | 浏览器强刷或用 curl |

## 参考
- `docs/en/codelab/web-gui.rst`（3.5）
- `docs/en/codelab/home_assistant.rst`（3.7）
- `components/esp_ot_br_server/include/esp_br_web.h`
- `components/esp_ot_br_server/src/openapi.yaml`
- `examples/basic_thread_border_router/main/esp_ot_br.c`
