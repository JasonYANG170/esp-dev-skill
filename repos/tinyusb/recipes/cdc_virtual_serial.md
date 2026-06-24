# CDC 虚拟串口

> **适用摘要**: 用 TinyUSB CDC 类实现 USB 虚拟串口，包括配置使能、收发读写、DTR/RTS 线状态、行编码回调。

## 触发意图

- "USB 虚拟串口"
- "TinyUSB CDC"
- "tud_cdc_read / tud_cdc_write"
- "USB 转串口设备"

## 前置条件

| 条件 | 要求 |
|---|---|
| 参考示例 | `examples/device/cdc_msc/src/main.c`（CDC 收发回显） |
| 配置项 | `CFG_TUD_CDC=1`、`CFG_TUD_CDC_RX/TX_BUFSIZE`、`CFG_TUD_CDC_EP_BUFSIZE` |

## 分步说明

### 1. tusb_config.h 使能 CDC

```c
#define CFG_TUD_CDC               1
#define CFG_TUD_CDC_NOTIFY        1   // 通知端点（默认开）

// RX/TX FIFO 大小（高速设备建议 512，全速 64）
#define CFG_TUD_CDC_RX_BUFSIZE    (TUD_OPT_HIGH_SPEED ? 512 : 64)
#define CFG_TUD_CDC_TX_BUFSIZE    (TUD_OPT_HIGH_SPEED ? 512 : 64)

// 端点传输缓冲（越大越快）
#define CFG_TUD_CDC_EP_BUFSIZE    (TUD_OPT_HIGH_SPEED ? 512 : 64)
```

> 多路 CDC：设 `CFG_TUD_CDC=N`，并使用 `tud_cdc_n_*` 系列带实例号 API。

### 2. 轮询读取并回显（主循环任务）

```c
void cdc_task(void) {
  if (tud_cdc_available()) {
    char buf[64];
    uint32_t count = tud_cdc_read(buf, sizeof(buf));

    // 回显
    tud_cdc_write(buf, count);
    tud_cdc_write_flush();   // 必须刷新才真正发送
  }
}

// 在主循环里调用
while (1) {
  tud_task();
  cdc_task();
}
```

### 3. 收到数据即处理（中断式回调）

```c
// 主机给设备发数据时由栈在任务上下文调用（非 ISR）
void tud_cdc_rx_cb(uint8_t itf) {
  (void)itf;
  // 可在此读出并处理；注意此回调在 tud_task() 上下文
  uint8_t buf[64];
  uint32_t n = tud_cdc_read(buf, sizeof(buf));
  // ...
}

// 发送完成
void tud_cdc_tx_complete_cb(uint8_t itf) { (void)itf; }
```

### 4. 线状态与波特率回调（终端连接/断开感知）

```c
// DTR/RTS 变化（多数终端连接时会置 DTR=1）
void tud_cdc_line_state_cb(uint8_t itf, bool dtr, bool rts) {
  (void)itf; (void)rts;
  if (dtr) {
    // 终端已连接
  } else {
    // 终端断开
  }
}

// 主机设置波特率/校验/数据位
void tud_cdc_line_coding_cb(uint8_t itf, cdc_line_coding_t const *coding) {
  (void)itf;
  // coding->bit_rate, coding->parity, coding->data_bits, coding->stop_bits
}
```

### 5. 查询连接/可用量

```c
if (tud_cdc_connected()) { /* DTR 已置位 */ }

uint32_t avail = tud_cdc_available();        // RX FIFO 可读字节
uint32_t wroom = tud_cdc_write_available();  // TX FIFO 可写字节
bool peek_ok = tud_cdc_peek(&ch);            // 偷看一字节不消费
tud_cdc_read_flush();                        // 丢弃 RX FIFO
```

## 常见错误

| 错误 | 原因 | 解决方法 |
|---|---|---|
| 主机收到空数据 | `tud_cdc_write` 后没 `write_flush` | 写完调 `tud_cdc_write_flush()` |
| 高速吞吐低 | `*_EP_BUFSIZE` / FIFO 太小 | 高速设 512，必要时调更大 FIFO |
| 多路 CDC 串扰 | 用了单实例 API | 多实例用 `tud_cdc_n_read(itf, ...)` 等 `_n_` API |
| `tud_cdc_rx_cb` 里阻塞 | 回调在任务上下文，阻塞会卡 `tud_task` | 回调里只读 FIFO，耗时操作放主循环 |
| 设备管理器感叹号 | VID/PID 与其他组合冲突 | 用 `PID_MAP()` 位图生成唯一 PID |

## 参考

- `examples/device/cdc_msc/src/main.c` — CDC 回显 + line state 回调
- `examples/device/cdc_dual_ports/` — 双 CDC 实例
- `src/class/cdc/cdc_device.h` — CDC 设备 API 与回调声明
