# 路由器/终端（ZR/ZED）开关：入网 + ZDO 发现与绑定

> **适用摘要**: 实现一个 HA on/off switch（ZR 或 ZED），通过 STEERING 加入协调器网络，用 ZDO `Match_Desc_req` 发现远端灯，再用 `Bind_req` 绑定，绑定后用 `ezb_zcl_on_off_toggle_cmd_req` 一键控制（无需指定目的地址）。

> Version: selected repo/component version; align with the user project and dependency manifest.
> Evidence: `repos/esp-zigbee-sdk/resources/`, source/examples in `repos/esp-zigbee-sdk/`, and this recipe path `repos/esp-zigbee-sdk/recipes/router_switch.md`.
> Validation: draft metadata added from repository routing; verify APIs, Kconfig symbols, and component dependencies against the selected version.

## 触发意图

- "做一个 Zigbee 开关"
- "Zigbee 入网 / join network"
- "ZDO 发现设备 / match descriptor"
- "Zigbee 绑定 / bind"
- "switch 控制灯"

## 前置条件

| 条件 | 要求 |
|---|---|
| 硬件 | ESP32-H2/C6（ZR 或 ZED）；同一网络中已有一个 ZC on/off 灯 |
| 软件 | ESP-IDF v5.2+，`esp-zigbee-lib >=2.0.0` |
| 参考项目 | `examples/home_automation_devices/on_off_switch/` |
| 依赖组件 | `examples/utils/switch_driver/`（按键）、`examples/utils/alarm_timer/` |

## 分步说明

### 1. 设备配置（终端 ZED；改路由器见注）

ZR 与 ZED 的区别仅在 `device_type`：ZR 用 `EZB_NWK_DEVICE_TYPE_ROUTER`（带 `zczr_config.max_children`），ZED 用 `EZB_NWK_DEVICE_TYPE_END_DEVICE`（带 `zed_config.ed_timeout` + `keep_alive`）。

```c
// main/on_off_switch.h —— 终端（ZED）配置（来自示例）
#define ESP_ZIGBEE_HA_ON_OFF_SWITCH_EP_ID (1)

#define ESP_ZIGBEE_ZED_CONFIG()                         \
    {                                                   \
        .device_type = EZB_NWK_DEVICE_TYPE_END_DEVICE,  \
        .install_code_policy = false,                   \
        .zed_config = {                                 \
            .ed_timeout = EZB_NWK_ED_TIMEOUT_64MIN,     \
            .keep_alive = 4000,                         \
        },                                              \
    }
// 若做路由器（ZR）：
// #define ESP_ZIGBEE_ZR_CONFIG() \
//     { .device_type = EZB_NWK_DEVICE_TYPE_ROUTER, .install_code_policy = false, \
//       .zczr_config = { .max_children = 10, }, }
```

### 2. ZDO Match_Desc_req：发现网络里的 on/off 灯

`dst_nwk_addr = 0xFFFD`（广播 rx-on-when-idle），`profile_id = EZB_AF_HA_PROFILE_ID`，匹配输入 cluster `EZB_ZCL_CLUSTER_ID_ON_OFF`。

```c
static void zdo_find_ha_light_device_result(const ezb_zdo_match_desc_req_result_t *result, void *user_ctx)
{
    assert(result);
    if (result->error == EZB_ERR_NONE && result->rsp &&
        result->rsp->status == EZB_ZDP_STATUS_SUCCESS && result->rsp->match_length > 0) {
        for (size_t i = 0; i < result->rsp->match_length; i++) {
            zdo_bind_ha_light_device(result->rsp->nwk_addr_of_interest, result->rsp->match_list[i]);
        }
    }
}

static ezb_err_t zdo_find_ha_light_device(void)
{
    uint16_t cluster_list[1] = {EZB_ZCL_CLUSTER_ID_ON_OFF};
    ezb_zdo_match_desc_req_t req = {
        .dst_nwk_addr = 0xFFFD,
        .field = {
            .nwk_addr_of_interest = 0xFFFD,
            .profile_id           = EZB_AF_HA_PROFILE_ID,
            .num_in_clusters      = 1,
            .num_out_clusters     = 0,
            .cluster_list         = cluster_list,
        },
        .cb       = zdo_find_ha_light_device_result,
        .user_ctx = NULL,
    };
    return ezb_zdo_match_desc_req(&req);
}
```

### 3. ZDO Bind_req：把本地 switch endpoint 绑到远端灯 endpoint

`dst_nwk_addr` 设为本地短地址（绑定请求经本节点发送），`src_addr` 取本地扩展地址，目的扩展地址用 `ezb_address_extended_by_short` 由短地址反查。

```c
static void zdo_bind_ha_light_device_result(const ezb_zdp_bind_req_result_t *result, void *user_ctx)
{
    if (result->error == EZB_ERR_NONE && result->rsp && result->rsp->status == EZB_ZDP_STATUS_SUCCESS) {
        ESP_LOGI(TAG, "Bound HA light device successfully");
    }
}

static ezb_err_t zdo_bind_ha_light_device(uint16_t dst_short_addr, uint8_t dst_ep)
{
    ezb_zdo_bind_req_t bind_req = {
        .dst_nwk_addr = ezb_nwk_get_short_address(),
        .field = {
            .src_ep        = ESP_ZIGBEE_HA_ON_OFF_SWITCH_EP_ID,
            .cluster_id    = EZB_ZCL_CLUSTER_ID_ON_OFF,
            .dst_addr_mode = EZB_ADDR_MODE_EXT,
            .dst_ep        = dst_ep,
        },
        .cb       = zdo_bind_ha_light_device_result,
        .user_ctx = NULL,
    };
    ezb_nwk_get_extended_address(&bind_req.field.src_addr);
    ESP_RETURN_ON_ERROR(ezb_address_extended_by_short(dst_short_addr, &bind_req.field.dst_addr.extended_addr),
                        TAG, "get ext addr failed for 0x%04hx", dst_short_addr);
    return ezb_zdo_bind_req(&bind_req);
}
```

### 4. STEERING 成功后触发发现

```c
case EZB_BDB_SIGNAL_STEERING: {
    ezb_bdb_comm_status_t status = *((ezb_bdb_comm_status_t *)ezb_app_signal_get_params(app_signal));
    if (status == EZB_BDB_STATUS_SUCCESS) {
        ESP_LOGI(TAG, "Joined: PAN(0x%04hx) Short(0x%04hx)", ezb_nwk_get_panid(), ezb_nwk_get_short_address());
        zdo_find_ha_light_device();   // 入网成功后找灯
    } else {
        alarm_timer_schedule(esp_zigbee_alarm_bdb_commissioning, EZB_BDB_MODE_NETWORK_STEERING, 1000);
    }
} break;
```

### 5. 按键发 toggle 命令（绑定后无需目的地址）

`dst_addr.addr_mode = EZB_ADDR_MODE_NONE` 表示由本地绑定表决定目的地址。

```c
static void button_event_handler(switch_driver_handle_t handle)
{
    ESP_RETURN_ON_FALSE(handle != SWITCH_INV_HANDLE, , TAG, "Invalid switch handle");
    ezb_zcl_on_off_cmd_t cmd_req = {
        .cmd_ctrl = {
            .dst_addr.addr_mode = EZB_ADDR_MODE_NONE,
            .src_ep             = ESP_ZIGBEE_HA_ON_OFF_SWITCH_EP_ID,
        },
    };
    esp_zigbee_lock_acquire(portMAX_DELAY);
    ezb_zcl_on_off_toggle_cmd_req(&cmd_req);
    esp_zigbee_lock_release();
}
```

## 常见错误

| 错误 | 原因 | 解决方法 |
|---|---|---|
| 一直入不了网 | 信道不匹配 / 周围无 ZC | 主次信道掩码与 ZC 一致；确认 ZC 已 FORMATION |
| `EZB_BDB_STATUS_NOT_PERMITTED` | ZR/ZED 误用 FORMATION | 入网只能用 `EZB_BDB_MODE_NETWORK_STEERING` |
| 找不到灯 | match descriptor 无响应 | 灯的 on/off cluster 须是 server；`profile_id` 用 `EZB_AF_HA_PROFILE_ID` |
| bind 失败 | 扩展地址反查失败 | 确认 `ezb_address_extended_by_short` 传入正确的短地址 |
| toggle 无效 | 未绑定就发 NONE 地址命令 | 必须先成功 bind，再以 `EZB_ADDR_MODE_NONE` 发命令 |
| 终端掉线 | keep_alive 不足 / ed_timeout 过短 | `keep_alive=4000`，`ed_timeout` 至少 `EZB_NWK_ED_TIMEOUT_64MIN` |

## 参考项目

- `examples/home_automation_devices/on_off_switch/` — ZED switch + 发现绑定
- `examples/home_automation_devices/color_dimmer_switch/` — ZR 彩灯开关（`ESP_ZIGBEE_ZR_CONFIG`）
- `examples/sleepy_devices/light_sleep_end_device/` — 休眠 switch
