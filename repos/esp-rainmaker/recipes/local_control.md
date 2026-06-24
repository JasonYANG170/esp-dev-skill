# 本地控制（Local Control）

> **适用摘要**: 启用 RainMaker 本地控制服务，让用户在同一 Wi-Fi/Thread 网络内无需互联网即可控制节点，配置 PoP、安全等级（sec0/sec1/sec2），以及 chal_resp 端点。

## 触发意图

- "RainMaker 本地控制"
- "local control 没网也能控"
- "PoP proof of possession"
- "sec0/sec1/sec2"
- "challenge-response 本地"

## 前置条件

| 条件 | 要求 |
|---|---|
| Kconfig | `CONFIG_ESP_RMAKER_LOCAL_CTRL_FEATURE_ENABLE=y` |
| 互斥 | 不能与 `CONFIG_ESP_RMAKER_ON_NETWORK_CHAL_RESP_ENABLE` 同时启用 |
| 参考示例 | 各示例 `event_handler` 中含 `RMAKER_EVENT_LOCAL_CTRL_STARTED` |

## 分步说明

### 1. 启用 Kconfig（`sdkconfig.defaults`）

```text
CONFIG_ESP_RMAKER_LOCAL_CTRL_FEATURE_ENABLE=y
# 可选：RainMaker 启动时自动启用，无需在代码里调 esp_rmaker_local_ctrl_enable()
CONFIG_ESP_RMAKER_LOCAL_CTRL_AUTO_ENABLE=y

# 安全等级（默认 sec1）
CONFIG_ESP_RMAKER_LOCAL_CTRL_SECURITY_1=y
# HTTP 端口（默认 8080）
# CONFIG_ESP_RMAKER_LOCAL_CTRL_HTTP_PORT=8080
# 任务栈大小（默认 6144）
# CONFIG_ESP_RMAKER_LOCAL_CTRL_STACK_SIZE=6144
```

> `LOCAL_CTRL_FEATURE_ENABLE` 会自动 select `ESP_HTTPS_SERVER_ENABLE`。注意：本地控制只使用 Wi-Fi 层安全，同网客户端理论上可控，通信为明文 HTTP（见 Kconfig 注释）。

### 2. 启用/禁用本地控制

```c
#include <esp_rmaker_core.h>

/* 显式启用（若未用 AUTO_ENABLE） */
esp_rmaker_local_ctrl_enable();

/* 停止并释放内存、从节点移除服务 */
esp_rmaker_local_ctrl_disable();

/* 查询是否已启动 */
if (esp_rmaker_local_ctrl_service_started()) { /* ... */ }
```

监听事件：
```c
case RMAKER_EVENT_LOCAL_CTRL_STARTED:
    ESP_LOGI(TAG, "Local Control Started. Service: %s", (char *)event_data);
    break;
case RMAKER_EVENT_LOCAL_CTRL_STOPPED:
    ESP_LOGI(TAG, "Local Control Stopped.");
    break;
```

### 3. 自定义 PoP（Proof of Possession）

```c
/* 必须在 esp_rmaker_local_ctrl_enable() 之前调用；否则内部随机生成并存 NVS */
esp_rmaker_local_ctrl_set_pop("abcd1234");   /* 典型 8 位字母数字 */

/* 传 NULL 清除已设的自定义 PoP */
esp_rmaker_local_ctrl_set_pop(NULL);
```

### 4. 安全等级选择

| Kconfig | 安全等级 | 说明 |
|---|---|---|
| `ESP_RMAKER_LOCAL_CTRL_SECURITY_0` | sec0 | 无加密（明文） |
| `ESP_RMAKER_LOCAL_CTRL_SECURITY_1` | sec1（默认） | 自动 select `ESP_PROTOCOMM_SUPPORT_SECURITY_VERSION_1` |
| `ESP_RMAKER_LOCAL_CTRL_SECURITY_2` | sec2 | 需 `ESP_PROTOCOMM_SUPPORT_SECURITY_VERSION_2` |

### 5. 本地控制的请求源

本地控制下发的 write 回调中 `ctx->src` 为 `ESP_RMAKER_REQ_SRC_LOCAL`（Wi-Fi）或 `ESP_RMAKER_REQ_SRC_BLE_LOCAL`（BLE）：

```c
if (ctx && ctx->src == ESP_RMAKER_REQ_SRC_LOCAL) {
    ESP_LOGI(TAG, "Command from local control");
}
```

### 6. 本地控制中的 chal_resp 端点（可选）

```text
# 需同时启用 challenge-response 与本地控制
CONFIG_ESP_RMAKER_ENABLE_CHALLENGE_RESPONSE=y
CONFIG_ESP_RMAKER_LOCAL_CTRL_FEATURE_ENABLE=y
CONFIG_ESP_RMAKER_LOCAL_CTRL_CHAL_RESP_ENABLE=y
```

```c
/* 在收到 RMAKER_EVENT_LOCAL_CTRL_STARTED 之后启用 chal_resp 端点 */
esp_rmaker_local_ctrl_enable_chal_resp(NULL);   /* NULL 用 node_id 作 mDNS 实例名 */
/* ... 映射完成后可禁用 ... */
esp_rmaker_local_ctrl_disable_chal_resp();
```

> 启用 chal_resp 端点会注册 protocomm handler 并通过 mDNS 广播 `_esp_rmaker_chal_resp._tcp`（Wi-Fi transport）。这与 on-network chal_resp 服务互斥（共用 protocomm_httpd 单例）。

### 7. 标准本地控制服务 helper（自定义集成）

```c
#include <esp_rmaker_standard_services.h>

esp_rmaker_device_t *lc_serv = esp_rmaker_create_local_control_service(
        "LocalControl", pop, sec_type, NULL);
```

标准参数 helper：
```c
esp_rmaker_local_control_pop_param_create(ESP_RMAKER_DEF_LOCAL_CONTROL_POP, "abcd1234");
esp_rmaker_local_control_type_param_create(ESP_RMAKER_DEF_LOCAL_CONTROL_TYPE, 1);
```

## 常见错误

| 错误 | 原因 | 解决方法 |
|---|---|---|
| 本地控制起不来 | Kconfig 未启用 | 设 `LOCAL_CTRL_FEATURE_ENABLE=y` |
| 与 on-network chal_resp 冲突 | 共用 protocomm_httpd 单例 | 二选一，关掉其一 |
| PoP 自定义不生效 | 在 enable 之后调用 | 必须在 `esp_rmaker_local_ctrl_enable()` 之前 |
| sec2 编译失败 | 未启用 protocomm sec2 | 启用 `ESP_PROTOCOMM_SUPPORT_SECURITY_VERSION_2` 或用 sec1 |
| 本地控制状态错乱 | AUTO_ENABLE 与手动 enable 重复 | 用 AUTO_ENABLE 就别再手动调 enable |

## 参考

- `components/esp_rainmaker/include/esp_rmaker_core.h` — local_ctrl API
- `components/esp_rainmaker/Kconfig.projbuild` — Local Control 菜单
- `examples/switch/main/app_main.c` — `RMAKER_EVENT_LOCAL_CTRL_STARTED` 处理
- `recipes/claiming_and_provisioning.md` — challenge-response 总览
