# USBTMC 测试测量设备类（SCPI/VISA 仪器）

> **适用摘要**: 用 TinyUSB USBTMC 类把设备暴露成 USB 测试测量仪器（电源、万用表、示波器），让 pyvisa / NI-VISA / Linux usbtmc 通过 SCPI 命令读写。含 USB488 模式、status byte（STB/MAV/SRQ）、`*IDN?` 查询、Bulk-IN/OUT 消息收发与清除/中止回调。

> Version: selected repo/component version; align with the user project and dependency manifest.
> Evidence: `repos/tinyusb/resources/`, source/examples in `repos/tinyusb/`, and this recipe path `repos/tinyusb/recipes/usbtmc_device.md`.
> Validation: draft metadata added from repository routing; verify APIs, Kconfig symbols, and component dependencies against the selected version.

## 触发意图

- "USB 仪器 / SCPI over USB"
- "TinyUSB USBTMC / USB488"
- "pyvisa / NI-VISA 设备"
- "tud_usbtmc_transmit_dev_msg_data"
- "*IDN? / 状态字节 STB / SRQ"

## 前置条件

| 条件 | 要求 |
|---|---|
| 参考示例 | `examples/device/usbtmc/`（含 `usbtmc_app.c` 应用层、`visaQuery.py` 主机测试脚本） |
| 配置项 | `CFG_TUD_USBTMC=1`、`CFG_TUD_USBTMC_ENABLE_488=1`（默认开）、`CFG_TUD_USBTMC_ENABLE_INT_EP=1` |
| 头文件 | `src/class/usbtmc/usbtmc_device.h`（应用回调与 `tud_usbtmc_transmit_dev_msg_data`）、`src/class/usbtmc/usbtmc.h`（消息结构、状态码）、`src/device/usbd.h`（`TUD_USBTMC_DESC`） |
| 主机工具 | `pyvisa` + `pyvisa-py`（后端 usbtmc）或 NI-VISA |

## 分步说明

### 1. tusb_config.h 使能 USBTMC（USB488 + 中断端点）

```c
#define CFG_TUD_USBTMC                1
#define CFG_TUD_USBTMC_ENABLE_488     1   // USB488 模式：支持 trigger / STB / SRQ
#define CFG_TUD_USBTMC_ENABLE_INT_EP  1   // 中断 IN 端点：发服务请求通知（SRQ）
```

> USB488 是 USBTMC 的子集，额外定义了 IEEE-488.2 风格的状态字节、trigger、SRQ，绝大多数 SCPI 仪器都用它。`CFG_TUD_USBTMC_ENABLE_488=1` 时 `tud_usbtmc_get_capabilities_cb` 返回 `usbtmc_response_capabilities_488_t`。

### 2. 能力声明（`tud_usbtmc_get_capabilities_cb`）

主机枚举时读能力描述符。取自 `examples/device/usbtmc/src/usbtmc_app.c`：

```c
static usbtmc_response_capabilities_488_t const tud_usbtmc_app_capabilities = {
    .USBTMC_status = USBTMC_STATUS_SUCCESS,
    .bcdUSBTMC     = USBTMC_VERSION,           // 0x0100
    .bmIntfcCapabilities = {
        .listenOnly = 0,                       // 既听又说
        .talkOnly   = 0,
        .supportsIndicatorPulse = 1            // 支持 indicator pulse（LED 闪）
    },
    .bmDevCapabilities = { .canEndBulkInOnTermChar = 0 },
#if CFG_TUD_USBTMC_ENABLE_488
    .bcdUSB488 = USBTMC_488_VERSION,
    .bmIntfcCapabilities488 = { .supportsTrigger = 1, .supportsREN_GTL_LLO = 0, .is488_2 = 1 },
    .bmDevCapabilities488   = { .SCPI = 1, .SR1 = 0, .RL1 = 0, .DT1 = 0 }
#endif
};

usbtmc_response_capabilities_488_t const * tud_usbtmc_get_capabilities_cb(void) {
  return &tud_usbtmc_app_capabilities;
}
```

### 3. 接收 SCPI 命令（Bulk-OUT）

主机发 Bulk-OUT 消息。先 `msgBulkOut_start_cb`（开始），分片到达 `msg_data_cb`（`transfer_complete=true` 表示整条命令收完）。**关键：每次处理完都要调 `tud_usbtmc_start_bus_read()` 准备接收下一包**（取自示例）：

```c
bool tud_usbtmc_msgBulkOut_start_cb(usbtmc_msg_request_dev_dep_out const *msgHeader) {
  (void)msgHeader;
  buffer_len = 0;
  if (msgHeader->TransferSize > sizeof(buffer)) return false;
  return true;
}

bool tud_usbtmc_msg_data_cb(void *data, size_t len, bool transfer_complete) {
  if (len + buffer_len < sizeof(buffer)) {
    memcpy(&buffer[buffer_len], data, len);
    buffer_len += len;
  } else {
    return false;   // 溢出
  }
  queryState = transfer_complete;
  idnQuery = 0;
  if (transfer_complete && len >= 4
      && (!strncmp("*idn?", data, 4) || !strncmp("*IDN?", data, 4))) {
    idnQuery = 1;   // 特殊处理 *IDN?
  }
  tud_usbtmc_start_bus_read();   // 必调，否则后续接收停摆
  return true;
}
```

> `transfer_complete` 只表示一个 USBTMC 消息传完（由 Bulk-OUT 头里的 `bmTransferAttributes.EOM` 标记），不代表传输结束。长命令可能分多包到达。

### 4. 发送响应（Bulk-IN + `tud_usbtmc_transmit_dev_msg_data`）

主机发 Bulk-IN 请求时栈调 `msgBulkIn_request_cb`。应用用 `tud_usbtmc_transmit_dev_msg_data(data, len, endOfMessage, usingTermChar)` 把数据排入发送队列：

```c
bool tud_usbtmc_msgBulkIn_request_cb(usbtmc_msg_request_dev_dep_in const *request) {
  msgReqLen = request->TransferSize;
  if (queryState == 0 || buffer_tx_ix == 0) {
    bulkInStarted = 1;     // 先记下，待应用准备好数据后再发
  } else {
    size_t txlen = tu_min32(buffer_len - buffer_tx_ix, msgReqLen);
    tud_usbtmc_transmit_dev_msg_data(&buffer[buffer_tx_ix], txlen,
                                     (buffer_tx_ix + txlen) == buffer_len, false);
    buffer_tx_ix += txlen;
  }
  return true;             // 永远返回 true，否则会 stall
}

// 在主循环任务里（usbtmc_app_task_iter）异步发送，如 *IDN? 应答：
tud_usbtmc_transmit_dev_msg_data(idn, tu_min32(sizeof(idn)-1, msgReqLen), true, false);
```

`endOfMessage=true` 在 Bulk-IN 头里置 EOM 位，告诉主机响应结束。缓冲区持有引用直至 `tud_usbtmc_msgBulkIn_complete_cb()` 被调用，期间缓冲内容不能变。

### 5. 状态字节 / SRQ（USB488）

IEEE-488.2 状态字节位（示例定义）：`MAV=0x10`（Message Available）、`SRQ=0x40`、`SER=0x20`、`QUESTIONABLE=0x08`。主机 `read_stb()` 时栈调 `tud_usbtmc_get_stb_cb`：

```c
uint8_t tud_usbtmc_get_stb_cb(uint8_t *tmcResult) {
  uint8_t old_status = status;
  status &= (uint8_t)~IEEE4882_STB_SRQ;   // 读 STB 自动清 SRQ
  *tmcResult = USBTMC_STATUS_SUCCESS;
  return old_status;
}

// 主机发 *TRG / ASSERT TRIGGER
bool tud_usbtmc_msg_trigger_cb(usbtmc_msg_generic_t *msg) {
  (void)msg;
  status |= IEEE4882_STB_SRQ;   // trigger 置位 SRQ
  return true;
}
```

要主动通知主机（服务请求），用中断端点：

```c
// 发通知（首字节是 bNotify1，见 USBTMC 规范 Table 13）
tud_usbtmc_transmit_notification_data(data, len);
// 发送完成回调
bool tud_usbtmc_notification_complete_cb(void) { return true; }
```

### 6. 清除 / 中止回调

主机发 `INITIATE_ABORT/CLEAR` 或 `CLEAR_FEATURE` 时栈调下列回调，应用应复位内部状态：

```c
bool tud_usbtmc_initiate_clear_cb(uint8_t *tmcResult) {
  *tmcResult = USBTMC_STATUS_SUCCESS;
  queryState = 0; bulkInStarted = false; status = 0;
  return true;
}
bool tud_usbtmc_check_clear_cb(usbtmc_get_clear_status_rsp_t *rsp) {
  queryState = 0; bulkInStarted = false; status = 0;
  buffer_tx_ix = 0; buffer_len = 0;
  rsp->USBTMC_status = USBTMC_STATUS_SUCCESS;
  rsp->bmClear.BulkInFifoBytes = 0;
  return true;
}
void tud_usbtmc_bulkOut_clearFeature_cb(void) { tud_usbtmc_start_bus_read(); }
void tud_usbtmc_bulkIn_clearFeature_cb(void)  { /* 复位 Bulk-IN */ }
```

### 7. 描述符（含中断端点）

```c
// 接口 subclass = 0x03（application specific / USBTMC）
// protocol: TUD_USBTMC_PROTOCOL_USB488 (0x01) 或 _STD (0x00)
TUD_USBTMC_DESC(ITF_NUM_USBTMC, 64),   // bulk-OUT/IN + 中断 IN（开 INT_EP 时）
```

`TUD_USBTMC_DESC` 自动按 `CFG_TUD_USBTMC_ENABLE_INT_EP` 展开成 2 或 3 个端点。

### 8. 主机端验证（pyvisa）

仓库自带 `examples/device/usbtmc/visaQuery.py`，用 pyvisa 测试：

```python
import pyvisa
rm = pyvisa.ResourceManager('@py')          # pyvisa-py 后端
inst = rm.open_resource('USB::VID::PID::INSTR')
print(inst.query("*IDN?"))                   # -> "TinyUSB,ModelNumber,SerialNumber,FirmwareVer123456\r\n"
print(inst.query("MEAS:VOLT?"))              # 你的 SCPI 命令
inst.read_stb()                              # 读状态字节
```

## 常见错误

| 错误 | 原因 | 解决方法 |
|---|---|---|
| pyvisa 找不到设备 | 缺 USBTMC 后端 / udev 权限 | Linux 装 `python3-usbtmc` 或用 `pyvisa-py`；加 udev 规则；Windows 用 NI-VISA |
| 设备枚举但 query 超时 | 收数据后没调 `tud_usbtmc_start_bus_read` | 每个处理完的回调（`open/data/complete/clear`）末尾都要调，否则不再收下一包 |
| 响应被截断 | `tud_usbtmc_transmit_dev_msg_data` 的 `endOfMessage` 错 | 最后一片置 `endOfMessage=true`；中转分片置 false |
| 主机一直 NAK Bulk-IN | 没准备好数据就返回 true | 没数据时也返回 true（NAK 而非 stall），有数据后再调 `transmit_dev_msg_data` |
| STB 永远不清 | `get_stb_cb` 没清 SRQ 位 | 读 STB 应清 `IEEE4882_STB_SRQ` |
| 缓冲区数据错乱 | 传输未完成就改缓冲 | 缓冲引用持有到 `msgBulkIn_complete_cb` 触发，期间不可改 |
| trigger 不生效 | 没实现 `tud_usbtmc_msg_trigger_cb` 或没置 SRQ | USB488 模式下实现 trigger 回调并置状态位 |
| 长命令丢数据 | `msg_data_cb` 只看 `transfer_complete` | 非 `transfer_complete` 的分片也要累加进 buffer |

## 参考

- `examples/device/usbtmc/src/usbtmc_app.c` — 完整应用层（SCPI 处理、STB、query 状态机、*IDN?）
- `examples/device/usbtmc/src/main.c` — 主循环（调 `usbtmc_app_task_iter`）
- `examples/device/usbtmc/visaQuery.py` — pyvisa 主机端测试（`*IDN?`、echo、trigger）
- `src/class/usbtmc/usbtmc_device.h` — 应用回调、`tud_usbtmc_transmit_dev_msg_data`、`tud_usbtmc_start_bus_read`
- `src/class/usbtmc/usbtmc.h` — 消息结构、状态码（`USBTMC_STATUS_*`）、能力结构
- `src/device/usbd.h` — `TUD_USBTMC_DESC` / `TUD_USBTMC_PROTOCOL_USB488`
