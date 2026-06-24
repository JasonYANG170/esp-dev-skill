# 实现 OTA 升级

> **适用摘要**: 选择并实现 ESP-AT 的三种 OTA 方案之一（`AT+USEROTA` 自有服务器、`AT+CIUPDATE` iot.espressif.cn、`AT+WEBSERVER` 浏览器/小程序），完成固件或用户分区升级。

## 触发意图

- "OTA 升级"
- "AT+USEROTA"
- "AT+CIUPDATE"
- "AT+WEBSERVER"
- "固件升级"
- "远程升级 AT 固件"

## 前置条件

| 条件 | 要求 |
|---|---|
| Kconfig | `CONFIG_AT_OTA_SUPPORT=y`（默认开） |
| Flash | 含 OTA 分区（factory bin 默认带） |
| 网络 | 设备已连 Wi-Fi（`AT+CWJAP`） |

## 分步说明

ESP-AT 提供三种 OTA 指令，按场景选择：

| 指令 | HTTP 服务器 | 升级范围 | 适用场景 |
|---|---|---|---|
| `AT+USEROTA` | 自有 HTTP 服务器（URL 指定） | 仅 app 分区 | 有自己的 HTTP 服务器，需指定 URL |
| `AT+CIUPDATE` | iot.espressif.cn（默认） | app + 用户分区（at_customize.csv） | 用官方服务器，或升级自定义分区 |
| `AT+WEBSERVER` | 浏览器/微信小程序上传 | 仅 app 分区 | 不依赖网络状态，更便捷 |

> `AT+USEROTA` 是用户自定义指令，可在 `components/at/src/at_user_cmd.c` 修改实现。
> `AT+CIUPDATE` 实现在 `components/at/src/at_ota_cmd.c`，`AT+WEBSERVER` 实现在 `components/at/src/at_web_server_cmd.c`，HTML 页面在 `components/fs_image/index.html`。

### 方案 1：AT+USEROTA（自有服务器）

把固件放到自有 HTTP 服务器，设备直接从 URL 拉取：

```
AT+USEROTA="http://my.server.com/firmware/esp-at.bin"
```

仅升级 app 分区。升级失败时原固件仍可运行（ESP-AT 把新固件存到备用 OTA 分区）。

> 若升级非官方固件，升级后可能无法再用 `AT+CIUPDATE` 升级（除非在 iot.espressif.cn 建设备）。

### 方案 2：AT+CIUPDATE（iot.espressif.cn）

1. 在 http://iot.espressif.cn 注册并登录（新用户加入功能目前需联系 Espressif 销售）。
2. Device → Create 创建设备，获得 key。
3. 用 key 在 menuconfig 配置 OTA token：
   - `AT` → `AT OTA token`（HTTP）
   - 若用 SSL OTA，还要配 `The SSL token for AT OTA`。
4. Product → ROM Deploy：填 version、corename，把 bin 重命名为 `ota.bin` 并上传，保存为当前版本。
5. 升级用户分区时，bin 文件名用 `at_customize.csv` 中的 `Name`（如 `factory_param.bin`）。
6. 设备联网后执行：

```
AT+CIUPDATE
```

注意事项（来自文档）：

- 升级 app：bin 名必须为 `ota.bin`。
- 升级用户分区：bin 名为分区 `Name`。
- 用户分区无备份，升级需谨慎。
- **若计划后续用自定义固件 + `AT+CIUPDATE`，初始版本就应把 OTA token 配成自有 token。**

### 方案 3：AT+WEBSERVER（浏览器/小程序）

启用 Web Server（`CONFIG_AT_WEB_SERVER_SUPPORT=y`），通过浏览器或微信小程序上传固件：

```
AT+WEBSERVER=1,<port>,<connection_timeout>
```

详情见 `AT+WEBSERVER` 指令集与 `docs/en/AT_Command_Examples/Web_server_AT_Examples`。自定义 HTML 见 `components/fs_image/index.html`。

### 关键原则（来自文档）

- 三种指令**仅 app 区**有备份（OTA 分区），用户分区**无备份**。
- 升级非官方固件后，`AT+CIUPDATE` 能力可能丢失（除非自有 token + 自有设备）。

## 常见错误

| 错误 | 原因 | 解决方法 |
|---|---|---|
| `AT+USEROTA` 失败 | URL 不可达或非 HTTP | 确认设备联网、URL 正确、服务器可访问 |
| `AT+CIUPDATE` 升级失败 | token 配置错或设备未在服务器注册 | 重新配 OTA token，确认 iot.espressif.cn 设备已建 |
| 升级后无法再用 `AT+CIUPDATE` | 刷了非官方固件 | 自建设备+自有 token，初始版本就用自有 token |
| `AT+WEBSERVER` 报错 | `CONFIG_AT_WEB_SERVER_SUPPORT` 未开 | menuconfig 开启；需要时开 captive portal 并把 `LWIP_MAX_SOCKETS` 调到 14 以上 |
| OTA bin 名错误导致不升级 | 名字不符合规则 | app 用 `ota.bin`，用户分区用 `Name.bin` |
| 用户分区升级变砖 | 用户分区无备份 | 谨慎升级，必要时先完整备份 |

## 参考

- 仓库文档：`docs/en/Compile_and_Develop/How_to_implement_OTA_update.rst`
- Web 示例：`docs/en/AT_Command_Examples/Web_server_AT_Examples`
- 源码：`components/at/src/at_user_cmd.c`、`at_ota_cmd.c`、`at_web_server_cmd.c`
- HTML：`components/fs_image/index.html`
