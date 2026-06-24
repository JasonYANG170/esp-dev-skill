# 发送 ZCL 命令（on/off、level、color、通用 read/write/report）

> **适用摘要**: 汇总常用 ZCL 命令的发送方法——on/off toggle/on/off、level move-to-level、color move-to-hue-and-saturation，以及通用 read/write/config-report 命令的请求结构与调用。

## 触发意图

- "发 on/off 命令"
- "调亮度 level control"
- "调颜色 color control"
- "读/写远端属性"
- "ZCL 命令怎么发"

## 前置条件

| 条件 | 要求 |
|---|---|
| 头文件 | `#include "esp_zigbee.h"`（已含 `ezbee/zcl.h` 及各 cluster 头） |
| 参考项目 | `examples/home_automation_devices/on_off_switch/`、`color_dimmer_switch/` |

## 分步说明

### 1. 命令控制块通用形态

每个命令结构都含 `cmd_ctrl`，内含 `dst_addr.addr_mode`、`src_ep`、`cluster_id`、`fc.direction` 等。

| 地址模式（`ezbee/core_types.h`） | 含义 |
|---|---|
| `EZB_ADDR_MODE_NONE` | 由绑定表决定目的（须先 bind） |
| `EZB_ADDR_MODE_SHORT` | 指定网络短地址（`EZB_ADDR_SHORT(addr)` 宏） |
| `EZB_ADDR_MODE_GROUP` | 组播（`EZB_ADDR_GROUP(gid)`） |
| `EZB_ADDR_MODE_EXT` | 扩展地址（`EZB_ADDR_EXT(eui64)`） |

### 2. on/off cluster（`ezbee/zcl/cluster/on_off.h`）

```c
#include "ezbee/zcl/cluster/on_off.h"

ezb_zcl_on_off_cmd_t cmd = {
    .cmd_ctrl = {
        .dst_addr.addr_mode = EZB_ADDR_MODE_NONE,   // 走绑定
        .src_ep             = SWITCH_EP,
    },
};
esp_zigbee_lock_acquire(portMAX_DELAY);
ezb_zcl_on_off_on_cmd_req(&cmd);        // 开
// ezb_zcl_on_off_off_cmd_req(&cmd);    // 关
// ezb_zcl_on_off_toggle_cmd_req(&cmd); // 切换
esp_zigbee_lock_release();
```

### 3. level control（`ezbee/zcl/cluster/level.h`）

```c
#include "ezbee/zcl/cluster/level.h"

ezb_zcl_level_move_to_level_cmd_t cmd = {
    .cmd_ctrl = { .dst_addr.addr_mode = EZB_ADDR_MODE_NONE, .src_ep = SWITCH_EP },
    .payload  = { .level = 200, .transition_time = 5 },   // level 0..254, 时间 1/10s
};
esp_zigbee_lock_acquire(portMAX_DELAY);
ezb_zcl_level_move_to_level_cmd_req(&cmd);
// ezb_zcl_level_move_to_level_with_on_off_cmd_req(&cmd); // 同时带 on/off
esp_zigbee_lock_release();
```

### 4. color control（`ezbee/zcl/cluster/color_control.h`）

```c
#include "ezbee/zcl/cluster/color_control.h"

ezb_zcl_color_control_move_to_hue_and_saturation_cmd_t cmd = {
    .cmd_ctrl = { .dst_addr.addr_mode = EZB_ADDR_MODE_NONE, .src_ep = SWITCH_EP },
    .payload  = { .hue = 240, .saturation = 100, .transition_time = 5 },
};
esp_zigbee_lock_acquire(portMAX_DELAY);
ezb_zcl_color_control_move_to_hue_and_saturation_cmd_req(&cmd);
// 其他可用：move_to_color_cmd_req / move_to_color_temperature_cmd_req /
//           move_hue_cmd_req / step_color_cmd_req / color_loop_set_cmd_req 等
esp_zigbee_lock_release();
```

> color_control 命令函数较多（move_to_hue、enhanced_move_to_hue、move_to_color、move_to_color_temperature、step_color、color_loop_set 等），完整列表见 `components/esp-zigbee-lib/include/ezbee/zcl/cluster/color_control.h`，payload 字段以该头为准。

### 5. 通用 read / write 属性（`ezbee/zcl/zcl_general_cmd.h`）

```c
#include "ezbee/zcl/zcl_general_cmd.h"

// 读远端 on/off 属性（指定目的短地址 0x0001）
uint16_t attr_id = EZB_ZCL_ATTR_ON_OFF_ON_OFF_ID;
ezb_zcl_read_attr_cmd_t read_cmd = {
    .cmd_ctrl = {
        .dst_addr.addr_mode = EZB_ADDR_MODE_SHORT,
        .dst_addr.u.short_addr = 0x0001,
        .src_ep     = SWITCH_EP,
        .cluster_id = EZB_ZCL_CLUSTER_ID_ON_OFF,
    },
    .payload = { .attr_number = 1, .attr_field = &attr_id },
};
esp_zigbee_lock_acquire(portMAX_DELAY);
ezb_zcl_read_attr_cmd_req(&read_cmd);
esp_zigbee_lock_release();
```

> `ezb_zcl_read_attr_cmd_t` 结构：`{ ezb_zcl_cmd_ctrl_t cmd_ctrl; struct { uint8_t attr_number; uint16_t *attr_field; } payload; }`。`ezb_zcl_write_attr_cmd_req`（payload 含 `ezb_zcl_attribute_t *attr_field`）与 `ezb_zcl_config_report_cmd_req` 结构类似，详见 `zcl_general_cmd.h`。读属性响应在 `EZB_ZCL_CORE_READ_ATTR_RSP_CB_ID` 回调里取。

### 6. 收到命令的默认响应

SDK 会自动处理大部分命令；应用层可通过 `EZB_ZCL_CORE_DEFAULT_RSP_CB_ID` 观察对端返回的状态码。

## 常见错误

| 错误 | 原因 | 解决方法 |
|---|---|---|
| 命令发出但无响应 | `EZB_ADDR_MODE_NONE` 但未绑定 | 先 ZDO bind，或改用 `EZB_ADDR_MODE_SHORT` 显式寻址 |
| 颜色命令编译报错 | 函数名拼错 | 以 `color_control.h` 实际声明为准 |
| level 不变化 | 灯未处于 on 状态 | 用 `move_to_level_with_on_off` 同时开灯 |
| 对端不回 default response | 命令的 `dis_default_rsp` 被置位 | 检查 frame control 字段 |
| 读属性返回 unsupported | cluster/attr 不存在于对端 | 确认对端 server cluster 与 attr_id |

## 参考项目

- `examples/home_automation_devices/on_off_switch/main/on_off_switch.c` — on/off toggle
- `examples/home_automation_devices/color_dimmer_switch/main/color_dimmer_switch.c` — level + color
- `components/esp-zigbee-lib/include/ezbee/zcl/cluster/on_off.h`、`level.h`、`color_control.h`
- `components/esp-zigbee-lib/include/ezbee/zcl/zcl_general_cmd.h` — 通用 read/write/report
