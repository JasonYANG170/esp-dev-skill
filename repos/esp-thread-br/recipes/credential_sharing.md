# Credential Sharing / ePSKc Commissioner

> **适用摘要**: 演示 Thread 1.4 Credential Sharing：在 BR 上生成临时密钥 ePSKc，通告 `meshcop-e` 服务，使一个 Thread Commissioner 经 DTLS 安全会话从 BR 检索或配置 Thread 网络凭据（Network Key / PSKd）。

## 触发意图
- "Credential Sharing"
- "ePSKc / ephemeral key"
- "meshcop-e"
- "Thread Commissioner 共享凭据"
- "Share Thread Network Credential"

## 前置条件

| 条件 | 要求 |
|---|---|
| 硬件 | M5Stack Thread Border Router（ESP32-S3 CoreS3 + ESP32-H2 模组），或自建带屏/带 Web GUI 的 BR |
| Kconfig | `CONFIG_OPENTHREAD_BR_START_WEB=y`（自动 `select OPENTHREAD_COMMISSIONER / OPENTHREAD_JOINER`） |
| M5Stack 专用 | `CONFIG_OPENTHREAD_EPHEMERALKEY_LIFE_TIME`（默认 100 秒）、`CONFIG_OPENTHREAD_EPHEMERALKEY_PORT`（默认 49180） |
| BR 状态 | 已连 Wi-Fi、Thread `leader`、Border Routing 处于 `running` |
| 参考 | `examples/m5stack_thread_border_router/README.md`、`docs/en/codelab/web-gui.rst`、`docs/en/index.rst` |

> 说明：Credential Sharing 是 `docs/en/index.rst` 中与 NAT64/TREL 并列的 Thread 1.4 顶层特性。`m5stack_thread_border_router` 示例专门演示此流程。

## 分步说明

### 1. 启用 Commissioner（Web Server 自动带入）

在 `basic_thread_border_router` / `m5stack_thread_border_router` 中开启 Web Server 即自动启用 Commissioner（见 `examples/common/thread_border_router/Kconfig.projbuild`）：

```
CONFIG_OPENTHREAD_BR_START_WEB=y     # 自动 select OPENTHREAD_COMMISSIONER + OPENTHREAD_JOINER
```

> `OPENTHREAD_COMMISSIONER` 由 `OPENTHREAD_BR_START_WEB` 自动 select，无需单独配置；手动选 Commissioner 会与 Web Server 重复。

### 2. 通过 M5Stack 触摸屏生成 ePSKc

`examples/common/thread_border_router_m5stack/src/br_m5stack_epskc_page.c` 实现。点击屏幕上的 **Share Thread Network Credential** 按钮，会：

1. 校验 Wi-Fi 已连接、Thread 角色 ≠ `disabled/detached`、Border Routing 状态 = `running`。
2. 生成 8 位随机数字 + Verhoeff 校验位（共 9 位）作为 ePSKc：
   ```c
   // br_m5stack_epskc_page.c —— generate_ephemeral_key()
   for (size_t i = 0; i < 8; ++i) {
       key_buf[i] = '0' + (esp_random() % 10);
   }
   otVerhoeffChecksumCalculate(key_buf, &checksum);   // openthread/verhoeff_checksum.h
   key_buf[8] = checksum;
   ```
3. 启动 Border Agent 临时密钥并通告 `meshcop-e`：
   ```c
   // create_ephemeral_key_page()
   otBorderAgentEphemeralKeyStart(esp_openthread_get_instance(), key_txt,
                                  CONFIG_OPENTHREAD_EPHEMERALKEY_LIFE_TIME * 1000,
                                  CONFIG_OPENTHREAD_EPHEMERALKEY_PORT);  // openthread/border_agent.h
   ```
4. 屏幕显示二维码 + 形如 `XXX-XXX-XXX` 的密钥；同时注册 meshcop-e 移除事件回调：
   ```c
   esp_openthread_register_meshcop_e_handler(br_m5stack_meshcop_e_remove_handler, false);  // for_publish=false → remove 事件
   ```

### 3. Commissioner 建立 DTLS 会话

外部 Thread Commissioner（如手机上的 Google Home / CHIP Tool / ot-commissioner）扫描二维码或手动输入 ePSKc，经 `meshcop-e` 服务发现 BR，使用 ePSKc 完成 DTLS 握手。成功后即可：

- **检索** 当前 Thread 网络凭据（Network Key、PAN ID、Channel 等）
- **配置** 新的 Network Key / PSKd，BR 收到后更新 Thread 网络

> ePSKc 仅在 `CONFIG_OPENTHREAD_EPHEMERALKEY_LIFE_TIME`（默认 100s）内有效；过期或点击屏幕上 **exit** 按钮会调用 `otBorderAgentEphemeralKeyStop(esp_openthread_get_instance())` 并移除 `meshcop-e` 服务。

### 4. 通过 Web GUI REST API 用凭据加网（commission / join）

Web Server 也提供 REST 接口，让管理员把 Joiner 凭据下发到 Commissioner BR（见 `components/esp_ot_br_server/src/esp_br_web_api.c`）：

**`POST /commission`** — 启动 Commissioner 并添加 Joiner（凭据为 PSKd）：
```bash
curl -X POST http://<BR-IPv4>:80/commission -H "Content-Type: application/json" \
  -d '{"pskd":"J01NME"}'
```
内部依次调用 `otCommissionerStart()` → 等待 `OT_COMMISSIONER_STATE_ACTIVE` → `otCommissionerAddJoiner(ins, NULL, pskd, 120)`（120 秒超时）。

**`POST /join_network`** — 凭据类型可选 Network Key 或 PSKd（见 `frontend/network.html`）：
```json
{
  "credentialType": "networkKeyType",   // 或 "pskdType"
  "networkKey": "...",                  // credentialType=networkKeyType 时
  "pskd": "..."                         // credentialType=pskdType 时
}
```
`esp_br_web_base.h` 定义常量：`CREDENTIAL_TYPE_NETWORK_KEY` / `CREDENTIAL_TYPE_PSKD`（`"pskdType"`）。`credentialType=networkKeyType` 走 `otJoinerStart(... networkKey ...)`；`pskdType` 走 `otJoinerStart(... pskd ...)`。

### 5. 验证凭据已共享

- M5Stack： Commissioner 建立 DTLS 后，`meshcop-e` 服务会被移除，屏幕的二维码页面通过 `br_m5stack_meshcop_e_remove_handler` 自动消失。
- 串口可观察 Commissioner / Joiner 事件日志（`handle_commissioner_join_event` 打印 `connect/finalize/end/remove`）。

## 常见错误

| 错误 | 原因 | 解决方法 |
|---|---|---|
| 按钮提示 "Wi-Fi is not connected" | Wi-Fi 未连 | 先完成 SoftAP 配网或手动 `wifi connect` |
| 按钮提示 "Border router is not running" | Border Routing 未就绪 | 确认 BR 已 `leader` 且骨干联网；等 `otBorderRoutingGetState()` 返回 `running` |
| ePSKc 生成为空 | Verhoeff 计算失败 | 极少见；确认 OpenThread 版本含 `otVerhoeffChecksumCalculate` |
| Commissioner 连不上 | `meshcop-e` 已过期或被 stop | 重新点按钮生成新 ePSKc；`LIFE_TIME` 内完成 |
| `POST /commission` 报错 | Commissioner 启动失败 | 确认 `OPENTHREAD_COMMISSIONER` 已被 `OPENTHREAD_BR_START_WEB` select 进来 |
| `credentialType` 无效 | 值不是 `networkKeyType`/`pskdType` | 用 `esp_br_web_base.h` 中的常量字符串 |
| M5Stack 编译报缺 `otBorderAgentEphemeralKeyStart` | OpenThread 版本过旧 | 使用 `idf.yml` 指定的 ESP-IDF v5.5.4 |

## 参考项目
- `examples/m5stack_thread_border_router/README.md` — Credential Sharing 示例说明
- `examples/m5stack_thread_border_router/` — M5Stack CoreS3 带 ePSKc UI 的 BR 示例
- `examples/common/thread_border_router_m5stack/src/br_m5stack_epskc_page.c` — ePSKc 生成 / `otBorderAgentEphemeralKeyStart` / meshcop-e 回调
- `examples/common/thread_border_router_m5stack/Kconfig.projbuild` — `OPENTHREAD_EPHEMERALKEY_LIFE_TIME` / `OPENTHREAD_EPHEMERALKEY_PORT`
- `examples/common/thread_border_router/Kconfig.projbuild` — `OPENTHREAD_BR_START_WEB` 自动 `select OPENTHREAD_COMMISSIONER/JOINER`
- `components/esp_ot_br_server/src/esp_br_web_api.c` — `handle_openthread_network_commission_request`（`/commission`）
- `components/esp_ot_br_server/private_include/esp_br_web_base.h` — `CREDENTIAL_TYPE_NETWORK_KEY` / `CREDENTIAL_TYPE_PSKD`
- OpenThread 栈 API：`otBorderAgentEphemeralKeyStart/Stop`（`openthread/border_agent.h`）、`otVerhoeffChecksumCalculate`（`openthread/verhoeff_checksum.h`）见 https://openthread.io/reference
- ESP-IDF：`esp_openthread_register_meshcop_e_handler`（`esp_openthread_netif_glue.h`）
