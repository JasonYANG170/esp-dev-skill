# Claiming 与配网

> **适用摘要**: 选择并配置 Claiming 类型（Self / Assisted / No Claim），完成 Wi-Fi/Thread 配网、Proof of Possession（PoP）以及用户-节点映射（含挑战-响应）。

## 触发意图

- "RainMaker claiming 怎么配"
- "self claim / assisted claim"
- "配网 PoP"
- "用户-节点映射"
- "challenge-response 配网"
- "RainMaker over Thread"

## 前置条件

| 条件 | 要求 |
|---|---|
| 芯片 | Claim 类型须与芯片匹配（见下表） |
| Kconfig | `CONFIG_ESP_RMAKER_CLAIM_TYPE` 等（见 `Kconfig.projbuild`） |
| 参考示例 | `examples/switch/`（默认含配网流程） |

## Claiming 类型与芯片矩阵

| 类型 | Kconfig 值 | 芯片限制 |
|---|---|---|
| No Claim (`ESP_RMAKER_NO_CLAIM`) | 0 | 无限制；MQTT 凭据需预烧（私有部署） |
| Self Claim (`ESP_RMAKER_SELF_CLAIM`) | 1 | **不支持** ESP32 / ESP32-C2 |
| Assisted Claim (`ESP_RMAKER_ASSISTED_CLAIM`) | 2 | 需 `BT_ENABLED` 且非 ESP32-S2 |

默认：`default ESP_RMAKER_ASSISTED_CLAIM if BT_ENABLED && !IDF_TARGET_ESP32S2`，否则 `ESP_RMAKER_SELF_CLAIM`。

## 分步说明

### 1. 选择 Claiming 类型（`sdkconfig.defaults`）

```text
# 自助 claiming（适用于 ESP32-S3/C3/C6/H2 等）
CONFIG_ESP_RMAKER_SELF_CLAIM=y

# 或辅助 claiming（需蓝牙）
# CONFIG_ESP_RMAKER_ASSISTED_CLAIM=y

# 密钥类型：默认 ECDSA P-256（推荐）
CONFIG_ESP_RMAKER_CLAIM_KEY_ECDSA=y
```

### 2. 配网（通过示例共用组件 `app_network`）

`app_network`（`examples/common/rmaker_app_network/`）封装了 Wi-Fi/Thread 配网，统一接口：

```c
#include <app_network.h>

/* 初始化（在 esp_rmaker_node_init 之前） */
app_network_init();

/* ... 创建节点、设备、enable 服务、esp_rmaker_start() ... */

/* 启动配网或连接 */
app_network_start(POP_TYPE_RANDOM);  // PoP 类型：随机
```

监听配网事件（`APP_NETWORK_EVENT` base）：

```c
ESP_ERROR_CHECK(esp_event_handler_register(APP_NETWORK_EVENT, ESP_EVENT_ANY_ID,
        &event_handler, NULL));

/* 在 event_handler 中： */
if (event_base == APP_NETWORK_EVENT) {
    switch (event_id) {
        case APP_NETWORK_EVENT_QR_DISPLAY:
            ESP_LOGI(TAG, "Provisioning QR : %s", (char *)event_data);
            break;
        case APP_NETWORK_EVENT_PROV_TIMEOUT:
            ESP_LOGI(TAG, "Provisioning Timed Out. Please reboot.");
            break;
        case APP_NETWORK_EVENT_PROV_RESTART:
            ESP_LOGI(TAG, "Provisioning has restarted due to failures.");
            break;
    }
}
```

### 3. 自定义制造数据（设备类型/子类型）

```c
app_network_set_custom_mfg_data(MFG_DATA_DEVICE_TYPE_SWITCH, MFG_DATA_DEVICE_SUBTYPE_SWITCH);
```

> `MFG_DATA_*` 宏由示例 `app_priv.h` 定义，用于配网广播中标识设备类型。

### 4. Claiming 事件监听

```c
/* RMAKER_EVENT base */
case RMAKER_EVENT_CLAIM_STARTED:
    ESP_LOGI(TAG, "RainMaker Claim Started.");
    break;
case RMAKER_EVENT_CLAIM_SUCCESSFUL:
    ESP_LOGI(TAG, "RainMaker Claim Successful.");
    break;
case RMAKER_EVENT_CLAIM_FAILED:
    ESP_LOGI(TAG, "RainMaker Claim Failed.");
    break;
```

### 5. 用户-节点映射（传统方式）

`esp_rmaker_user_mapping.h` 提供 API（注意：启用 `CONFIG_ESP_RMAKER_ENABLE_CHALLENGE_RESPONSE=y` 后，传统 user mapping 自动禁用，由 chal_resp 取代）：

```c
#include <esp_rmaker_user_mapping.h>

/* 在配网管理器 init 之后、start_provisioning 之前创建端点 */
esp_rmaker_user_mapping_endpoint_create();
/* 在 start_provisioning 之后立即注册 */
esp_rmaker_user_mapping_endpoint_register();

/* 或在已配网后手动触发映射 */
esp_rmaker_start_user_node_mapping(user_id, secret_key);

/* 查询映射状态 */
esp_rmaker_user_mapping_state_t st = esp_rmaker_user_node_mapping_get_state();
/* ESP_RMAKER_USER_MAPPING_RESET / STARTED / REQ_SENT / DONE */
```

监听 `RMAKER_EVENT_USER_NODE_MAPPING_DONE` 事件。

### 6. 挑战-响应（Challenge-Response，默认启用）

`CONFIG_ESP_RMAKER_ENABLE_CHALLENGE_RESPONSE=y`（默认）。节点用私钥对客户端发来的 challenge 签名：

```c
/* 用 RainMaker 私钥签名 challenge（response 需调用者释放） */
void *response; size_t outlen;
esp_rmaker_node_auth_sign_msg(challenge, inlen, &response, &outlen);
/* ... 使用 response ... */
free(response);
```

> 注意：该特性要求节点已 claimed。

### 7. RainMaker over Thread（可选）

```text
# sdkconfig.defaults
CONFIG_ESP_RMAKER_NETWORK_OVER_THREAD=y
CONFIG_OPENTHREAD_ENABLED=y
```

> Thread 模式下 SRP 服务租约间隔由 `CONFIG_ESP_RMAKER_LOCAL_CTRL_LEASE_INTERVAL_SECONDS`（默认 180 秒）控制。

## 常见错误

| 错误 | 原因 | 解决方法 |
|---|---|---|
| Self Claim 编译失败 | 芯片不支持（ESP32/ESP32-C2） | 改用 Assisted Claim 或 No Claim |
| Assisted Claim 不可用 | 无 BT（如 ESP32-S2）或未启用 BT | 用 Self Claim 或加 BT |
| 配网二维码不显示 | 未注册 `APP_NETWORK_EVENT` handler | 注册事件并打印 `APP_NETWORK_EVENT_QR_DISPLAY` |
| `chal_resp` 编译报错 | 节点未 claimed | challenge-response 要求节点已 claim |
| 传统 user mapping 无效 | `ENABLE_CHALLENGE_RESPONSE=y` 自动禁用传统流程 | 用 chal_resp，或关掉该配置 |
| MQTT host 错误 | 使用 No Claim 但未设 host | 设 `CONFIG_ESP_RMAKER_READ_MQTT_HOST_FROM_CONFIG=y` + `CONFIG_ESP_RMAKER_MQTT_HOST` |

## 参考

- `components/esp_rainmaker/Kconfig.projbuild` — Claim/PKI/MQTT 配置
- `components/esp_rainmaker/include/esp_rmaker_user_mapping.h`
- `examples/common/rmaker_app_network/` — 配网组件
- `examples/thread_br/` — Thread 边界路由器示例
- `recipes/getting_started.md` — 完整初始化骨架
