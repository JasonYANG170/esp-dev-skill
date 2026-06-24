# USB Type-C 与 Power Delivery（PD 3.0）

> **适用摘要**: 用 TinyUSB Type-C / Power Delivery 栈（`tuc_` API，`src/typec/`）实现 USB PD 3.0 sink/source，解析 Source Capabilities（PDO）、按电压/电流选 PDO 并发 Request（RDO）、响应 Accept/Reject/PS_READY。

> **⚠️ WIP 限制**：`README.rst` 明确标注 PD 栈为 "WIP / Super early stage / Only for testing purpose / **Only support STM32 G4**"。`examples/typec/power_delivery/only.txt` 也限定 `mcu:STM32G4`。本配方基于当前可用 API 编写，PD 栈尚未稳定，跨平台移植需等待上游。

## 触发意图

- "USB PD / Power Delivery / PD 3.0"
- "USB Type-C / Type-C sink / source"
- "TinyUSB tuc_init / tuc_task"
- "PDO / RDO / Source Capabilities"
- "USB-C 协商电压电流"

## 前置条件

| 条件 | 要求 |
|---|---|
| 参考示例 | `examples/typec/power_delivery/`（独立 `examples/typec/` 树，自带 CMakeLists/CMakePresets） |
| 目标 MCU | **仅 STM32 G4**（WIP 限制，见 README 与 `only.txt`） |
| 配置项 | `CFG_TUC_ENABLED=1`（默认 0，见 `src/tusb_option.h`） |
| 头文件 | `src/typec/usbc.h`（应用 API：`tuc_init/task/msg_request` + 回调）、`src/typec/tcd.h`（控制器驱动移植层 + 事件）、`src/typec/pd_types.h`（`pd_header_t` / `pd_pdo_fixed_t` / `pd_rdo_fixed_variable_t` / 枚举） |
| 硬件 | 板子需带 USB-C 接口与 PD 控制器（TCPC）或片上 UCPD 外设（STM32 G4 有 UCPD） |

## 分步说明

### 1. PD 栈与 USB 数据栈是两套独立子系统

Type-C/PD 代码在 `src/typec/`，**完全独立于** `tud_`/`tuh_` 数据栈：

| 维度 | USB 数据栈 | Type-C/PD 栈 |
|---|---|---|
| API 前缀 | `tud_` / `tuh_` | `tuc_`（Type-C） + `tcd_`（控制器驱动层） |
| 配置开关 | `CFG_TUD_ENABLED` / `CFG_TUH_ENABLED` | `CFG_TUC_ENABLED` |
| 任务函数 | `tud_task()` / `tuh_task()` | `tuc_task()` |
| 初始化 | `tusb_init()` | `tuc_init(rhport, port_type)` |
| 目录 | `src/device/`、`src/host/` | `src/typec/` |
| 中断转发 | `tusb_int_handler()` | `tuc_int_handler()`（`tcd_int_handler` 别名） |

设备/主机和 PD 可以共存（Dual-Role Power 设备），但二者初始化与任务循环各自独立。

### 2. tusb_config.h 使能 Type-C 栈

```c
#define CFG_TUC_ENABLED   1
// 任务队列深度（默认 8）
// #define CFG_TUC_TASK_QUEUE_SZ  8
```

> 不需要 `CFG_TUD_*` / `CFG_TUH_*`，PD-only 设备可只开 `CFG_TUC_ENABLED`。

### 3. 初始化与主循环（sink 为例）

`tuc_init(rhport, port_type)` 的 `port_type` 取自 `pd_types.h` 枚举（取自 `examples/typec/power_delivery/src/main.c`）：

```c
// 端口角色
// TUSB_TYPEC_PORT_SRC  — 纯源（供电方）
// TUSB_TYPEC_PORT_SNK  — 纯sink（受电方）
// TUSB_TYPEC_PORT_DRP  — 双角色（动态切换）

int main(void) {
  board_init();
  board_led_write(true);

  tuc_init(0, TUSB_TYPEC_PORT_SNK);   // sink 角色

  while (1) {
    led_blinking_task();
    tuc_task();                        // PD 任务，必须在主循环频繁调用（同 tud_task）
  }
}
```

> 中断转发：在 USB-PD / UCPD 中断 ISR 里调用 `tuc_int_handler(rhport)`（即 `tcd_int_handler`）。控制器驱动（`tcd_*`）是移植层，各 MCU 在 `src/portable/` 或驱动里实现 `tcd_init/msg_send/msg_receive` 与事件上报（`tcd_event_handler`）。

### 4. 解析 Source Capabilities（PDO）并发 Request（RDO）

sink 收到 source 的广告能力时栈调 `tuc_pd_data_received_cb`，应用遍历 PDO 选合适的并回 RDO（示例完整逻辑）：

```c
#define VOLTAGE_MAX_MV       5000   // 安全：只接受 5V
#define CURRENT_MAX_MA       500
#define CURRENT_OPERATING_MA 100

bool tuc_pd_data_received_cb(uint8_t rhport, pd_header_t const *header,
                             uint8_t const *dobj, uint8_t const *p_end) {
  if (header->msg_type != PD_DATA_SOURCE_CAP) return true;

  uint8_t selected_pos = 1;   // 默认 safe5v（位置从 1 起）

  for (size_t i = 0; i < header->n_data_obj; i++) {
    TU_VERIFY(dobj < p_end);
    uint32_t const pdo = tu_le32toh(tu_unaligned_read32(dobj));

    switch ((pdo >> 30) & 0x03ul) {
      case PD_PDO_TYPE_FIXED: {
        pd_pdo_fixed_t const *fixed = (pd_pdo_fixed_t const *)&pdo;
        uint32_t voltage_mv = fixed->voltage_50mv * 50;       // 单位 50mV
        uint32_t current_ma = fixed->current_max_10ma * 10;   // 单位 10mA
        printf("[Fixed] %" PRIu32 " mV %" PRIu32 " mA\r\n", voltage_mv, current_ma);
        if (voltage_mv <= VOLTAGE_MAX_MV && current_ma >= CURRENT_MAX_MA) {
          selected_pos = i + 1;            // 记下满足要求的位置
        }
        break;
      }
      case PD_PDO_TYPE_BATTERY:  // pd_pdo_battery_t: voltage_min/max_50mv, power_max_250mw
      case PD_PDO_TYPE_VARIABLE: // pd_pdo_variable_t: voltage_min/max_50mv, current_max_10ma
      case PD_PDO_TYPE_APDO:     // pd_pdo_apdo_t: PPS，电压/电流可编程
        break;
    }
    dobj += 4;   // 每个 PDO 4 字节
  }

  // 构造 RDO 回复（Fixed/Variable 用 pd_rdo_fixed_variable_t）
  pd_rdo_fixed_variable_t rdo = {
      .current_extremum_10ma       = CURRENT_MAX_MA / 10,        // 最大电流
      .current_operate_10ma        = CURRENT_OPERATING_MA / 10,  // 工作电流
      .object_position             = selected_pos,               // 选中的 PDO 编号
      .usb_comm_capable            = 1,
      .give_back_flag              = 0,    // extremum 是最大值
      .capability_mismatch         = 0,
      .no_usb_suspend              = 0,
      .unchunked_ext_msg_support   = 0,
      .epr_mode_capable            = 0,
  };
  tuc_msg_request(rhport, &rdo);
  return true;
}
```

> **安全警告**（示例原话）：选高于 5V 的 PDO 前确认板子能承受该电压，否则可能损坏硬件、冒烟。`object_position` 从 1 开始（位置 1 通常是 safe5v）。

### 5. 响应协商控制消息

source 处理 RDO 后回 Accept/Reject/PS_READY，栈调 `tuc_pd_control_received_cb`：

```c
bool tuc_pd_control_received_cb(uint8_t rhport, pd_header_t const *header) {
  (void)rhport;
  switch (header->msg_type) {
    case PD_CTRL_ACCEPT:    printf("PD Request Accepted\r\n");   break;  // 即将转换功率
    case PD_CTRL_REJECT:    printf("PD Request Rejected\r\n");   break;  // 协商失败
    case PD_CTRL_PS_READY:  printf("PD Power Ready\r\n");        break;  // 电源就绪，可正常用电
    default: break;
  }
  return true;
}
```

### 6. 数据结构速查（`pd_types.h`）

**`pd_header_t`**（2 字节，PD 消息头）：`msg_type`、`n_data_obj`（数据对象数，0=控制消息）、`id`（消息序号）、`power_role`、`data_role`、`spec_rev` 等。

**PDO 类型**（source 广告，4 字节，高 2 位是类型）：
| 类型 | 结构 | 关键字段单位 |
|---|---|---|
| `PD_PDO_TYPE_FIXED` | `pd_pdo_fixed_t` | `voltage_50mv`(50mV)、`current_max_10ma`(10mA) |
| `PD_PDO_TYPE_BATTERY` | `pd_pdo_battery_t` | `voltage_min/max_50mv`、`power_max_250mw`(250mW) |
| `PD_PDO_TYPE_VARIABLE` | `pd_pdo_variable_t` | `voltage_min/max_50mv`、`current_max_10ma` |
| `PD_PDO_TYPE_APDO` | `pd_pdo_apdo_t` | PPS：`voltage_min/max_100mv`、`current_max_50ma` |

**RDO**（sink 请求，4 字节）：`pd_rdo_fixed_variable_t`：`object_position`（选中的 PDO）、`current_operate_10ma`、`current_extremum_10ma`、`give_back_flag`、`usb_comm_capable`、`capability_mismatch`。

### 7. 控制器驱动移植层（`tcd_*`）

`tcd.h` 定义移植接口（MCU 厂商或板卡实现）：`tcd_init`、`tcd_msg_send`、`tcd_msg_receive`、`tcd_int_handler`，以及事件上报 `tcd_event_handler`（事件类型 `TCD_EVENT_CC_CHANGED` / `TCD_EVENT_RX_COMPLETE` / `TCD_EVENT_TX_COMPLETE`）。应用一般不直接调 `tcd_*`，只通过 `tuc_*` API；移植新 PD 控制器时实现这些。

## 常见错误

| 错误 | 原因 | 解决方法 |
|---|---|---|
| 非 STM32 G4 板编译/链接失败 | PD 栈仅支持 STM32 G4（WIP） | README 与 `only.txt` 限定；跨平台需等上游完善或自行实现 `tcd_*` 移植层 |
| `tuc_init` 链接不到 | 未开 `CFG_TUC_ENABLED` | 默认 0，需在 `tusb_config.h` 置 1 |
| sink 不发 RDO / 协商失败 | `tuc_pd_data_received_cb` 没回 Request | 收到 `PD_DATA_SOURCE_CAP` 后必须调 `tuc_msg_request()` 回 RDO |
| 板子损坏 / 冒烟 | 选了超过板子承受的电压 PDO | 严格按板子规格限电压（示例用 5000mV）；高电压前确认硬件 |
| `object_position` 错位 | 当成 0 起 | PDO 位置从 **1** 起（1=safe5v） |
| 电压/电流数值差 50 倍 / 10 倍 | 单位换算错 | Fixed：电压 50mV、电流 10mA；Battery 功率 250mW；APDO 电压 100mV、电流 50mA |
| PD 事件不触发 | 主循环没喂 `tuc_task` 或 ISR 没转发 | 同 USB 数据栈：`tuc_task()` 频繁调用，UCPD ISR 里调 `tuc_int_handler()` |
| 拿不到 source cap | 端口角色错或 CC 没连 | sink 用 `TUSB_TYPEC_PORT_SNK`；确认 USB-C 线方向与 source 上电 |
| 双角色不切换 | 用了纯 SRC/SNK | 用 `TUSB_TYPEC_PORT_DRP`（DRP 支持动态切换，但 WIP 阶段能力有限） |

## 参考

- `examples/typec/power_delivery/src/main.c` — sink 完整示例：解析 PDO + 选 PDO + 发 RDO + 响应控制消息
- `examples/typec/power_delivery/only.txt` — `mcu:STM32G4`（平台限制）
- `examples/typec/power_delivery/` — 独立 typec 示例树（CMakeLists/CMakePresets/Makefile）
- `src/typec/usbc.h` — 应用 API（`tuc_init/task/msg_request`、`tuc_pd_data_received_cb`、`tuc_pd_control_received_cb`）
- `src/typec/tcd.h` — 控制器驱动移植层（`tcd_*`、`tcd_event_handler`、`tcd_event_t`）
- `src/typec/pd_types.h` — 端口角色枚举、`pd_header_t`、PDO 结构（`pd_pdo_fixed_t` 等）、RDO 结构（`pd_rdo_fixed_variable_t`）、消息类型（`PD_DATA_SOURCE_CAP`、`PD_CTRL_ACCEPT` 等）
- `README.rst` "Power Delivery Stack" 段 — WIP / 仅 STM32 G4 声明
