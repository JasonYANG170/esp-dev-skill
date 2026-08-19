# Touchlink Commissioning（initiator 与 target）

> **适用摘要**: 讲解 BDB Touchlink commissioning——initiator 主动扫描并拉起 target 入网，target 等待被发起。两者均通过 `ezb_bdb_start_top_level_commissioning` 配合对应模式与信号完成。

> Version: selected repo/component version; align with the user project and dependency manifest.
> Evidence: `repos/esp-zigbee-sdk/resources/`, source/examples in `repos/esp-zigbee-sdk/`, and this recipe path `repos/esp-zigbee-sdk/recipes/touchlink.md`.
> Validation: draft metadata added from repository routing; verify APIs, Kconfig symbols, and component dependencies against the selected version.

## 触发意图

- "Touchlink"
- "Zigbee 触联"
- "touchlink initiator / target"
- "靠近即配网"
- "Hue 风格配网"

## 前置条件

| 条件 | 要求 |
|---|---|
| 头文件 | `#include "esp_zigbee.h"`（含 `ezbee/touchlink.h`、`ezbee/bdb.h`） |
| 参考项目 | `examples/touchlink/touchlink_initiator/`、`examples/touchlink/touchlink_target/` |

## 分步说明

### 1. initiator：在 FIRST_START 启动 TOUCHLINK_INITIATOR

```c
case EZB_ZDO_SIGNAL_SKIP_STARTUP:
    ezb_bdb_start_top_level_commissioning(EZB_BDB_MODE_INITIALIZATION);
    break;
case EZB_BDB_SIGNAL_DEVICE_FIRST_START:
case EZB_BDB_SIGNAL_DEVICE_REBOOT: {
    ezb_bdb_comm_status_t status = *((ezb_bdb_comm_status_t *)ezb_app_signal_get_params(app_signal));
    if (status == EZB_BDB_STATUS_SUCCESS) {
        if (ezb_bdb_is_factory_new()) {
            ezb_bdb_start_top_level_commissioning(EZB_BDB_MODE_TOUCHLINK_INITIATOR);
        }
    } else {
        alarm_timer_schedule(esp_zigbee_alarm_bdb_commissioning, EZB_BDB_MODE_INITIALIZATION, 1000);
    }
} break;
case EZB_BDB_SIGNAL_TOUCHLINK_INITIATOR_FINISHED: {
    ezb_bdb_comm_status_t status = *((ezb_bdb_comm_status_t *)ezb_app_signal_get_params(app_signal));
    ESP_LOGI(TAG, "Touchlink initiator finished: status(0x%02x)", status);
} break;
```

### 2. target：启动 TOUCHLINK_TARGET

```c
case EZB_BDB_SIGNAL_DEVICE_FIRST_START:
case EZB_BDB_SIGNAL_DEVICE_REBOOT: {
    ezb_bdb_comm_status_t status = *((ezb_bdb_comm_status_t *)ezb_app_signal_get_params(app_signal));
    if (status == EZB_BDB_STATUS_SUCCESS) {
        if (ezb_bdb_is_factory_new()) {
            ezb_bdb_start_top_level_commissioning(EZB_BDB_MODE_TOUCHLINK_TARGET); // 等待被发起
        }
    }
} break;
case EZB_BDB_SIGNAL_TOUCHLINK_TARGET_FINISHED: {
    ezb_bdb_comm_status_t status = *((ezb_bdb_comm_status_t *)ezb_app_signal_get_params(app_signal));
    ESP_LOGI(TAG, "Touchlink target finished: status(0x%02x)", status);
} break;
```

### 3. Touchlink master key 与权限回调（`ezbee/touchlink.h`）

可选：设置 Touchlink master key、注册动作权限与 identify 回调。

```c
#include "ezbee/touchlink.h"

uint8_t key[16] = { /* 16 字节 master key */ };
ezb_touchlink_set_master_key(key);

ezb_touchlink_action_permission_handler_register(my_permission_cb);   // 返回是否允许某 action
ezb_touchlink_identify_handler_register(my_identify_cb);              // 收到 identify 时的回调
```

### 4. 取消 Touchlink target

```c
ezb_bdb_cancel_touchlink_target();   // 仅在 target 流程中有效
```

### 5. BDB 状态（`ezbee/bdb.h`，Touchlink 相关）

| 状态 | 含义 |
|---|---|
| `EZB_BDB_STATUS_SUCCESS` | Touchlink 成功 |
| `EZB_BDB_STATUS_NO_SCAN_RESPONSE` | initiator 扫描无响应 |
| `EZB_BDB_STATUS_NOT_AA_CAPABLE` | initiator 不具备地址分配能力 |
| `EZB_BDB_STATUS_TARGET_FAILURE` | target 在 join 阶段失败 |
| `EZB_BDB_STATUS_NO_NETWORK` | 网络启动失败 |

## 常见错误

| 错误 | 原因 | 解决方法 |
|---|---|---|
| initiator 一直 `NO_SCAN_RESPONSE` | 附近无 target / target 未启动 | target 先进入 `TOUCHLINK_TARGET`，物理靠近 |
| `NOT_PERMITTED` | 在集中式网络或错误状态发起 | 仅分布式网络/工厂新设备可 Touchlink |
| target 收不到 | master key 不一致 | 双方用同一 Touchlink master key |
| 想中途中止 | target 已被选中 | `ezb_bdb_cancel_touchlink_target()` |
| 设备入网后不掉线 | 正常 | Touchlink 成功即等价于入网 |

## 参考项目

- `examples/touchlink/touchlink_initiator/main/touchlink_initiator.c`
- `examples/touchlink/touchlink_target/main/touchlink_target.c`
- `components/esp-zigbee-lib/include/ezbee/touchlink.h`
