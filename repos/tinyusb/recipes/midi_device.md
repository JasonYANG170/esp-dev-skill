# MIDI 音乐设备接口类

> **适用摘要**: 用 TinyUSB MIDI 类实现 USB MIDI 设备（MIDI 输入/输出），包括 4 字节 USB-MIDI 事件包格式、cable number、Note On/Off 收发、`tud_midi_packet_read/write` 与流式 `tud_midi_stream_write`。

> Version: selected repo/component version; align with the user project and dependency manifest.
> Evidence: `repos/tinyusb/resources/`, source/examples in `repos/tinyusb/`, and this recipe path `repos/tinyusb/recipes/midi_device.md`.
> Validation: draft metadata added from repository routing; verify APIs, Kconfig symbols, and component dependencies against the selected version.

## 触发意图

- "USB MIDI 设备"
- "TinyUSB MIDI"
- "tud_midi_packet_write / tud_midi_stream_write"
- "USB 音乐键盘 / MIDI 控制器"
- "Note On / Note Off USB"

## 前置条件

| 条件 | 要求 |
|---|---|
| 参考示例 | `examples/device/midi_test/`、`examples/device/midi_test_freertos/`、`examples/host/midi_rx/`（主机侧） |
| 配置项 | `CFG_TUD_MIDI=1`、`CFG_TUD_MIDI_EP_BUFSIZE`（默认 FS=64/HS=512） |
| 头文件 | `src/class/midi/midi_device.h`（API）、`src/class/midi/midi.h`（Code Index Number 枚举）、`src/device/usbd.h`（`TUD_MIDI_DESCRIPTOR`） |
| 主机端 | Linux：qsynth+qjackctl；Windows：MIDI-OX；macOS：SimpleSynth |

## 分步说明

### 1. tusb_config.h 使能 MIDI

```c
#define CFG_TUD_MIDI             1
#define CFG_TUD_MIDI_EP_BUFSIZE  (TUD_OPT_HIGH_SPEED ? 512 : 64)   // 端点缓冲
```

> 旧名 `CFG_TUD_MIDI_EPSIZE` 仍兼容但会告警，建议改用 `_EP_BUFSIZE`。

### 2. USB-MIDI 事件包格式（4 字节，区别于 MIDI 线缆协议）

USB-MIDI 用 4 字节包承载 1 个 MIDI 事件，第 0 字节高 4 位是 cable number，低 4 位是 Code Index Number（CIN）：

```
字节0: [ Cable Number (4 bit) | CIN (4 bit) ]
字节1: MIDI 状态字节 (mstatus)
字节2: MIDI 数据 1
字节3: MIDI 数据 2
```

CIN 告诉接收方剩余字节有效长度（取自 `src/class/midi/midi.h`）：

| CIN | 枚举 | 含义 | 有效数据字节 |
|---|---|---|---|
| 8 | `MIDI_CIN_NOTE_OFF` | Note Off | 3 |
| 9 | `MIDI_CIN_NOTE_ON` | Note On | 3 |
| 11 | `MIDI_CIN_CONTROL_CHANGE` | 控制变化 | 3 |
| 12 | `MIDI_CIN_PROGRAM_CHANGE` | 音色变化 | 2 |
| 13 | `MIDI_CIN_CHANNEL_PRESSURE` | 通道压力 | 2 |
| 14 | `MIDI_CIN_PITCH_BEND_CHANGE` | 弯音 | 3 |
| 4 | `MIDI_CIN_SYSEX_START` | SysEx 开始/继续 | 3 |
| 5/6/7 | `MIDI_CIN_SYSEX_END_*` | SysEx 结束（1/2/3 字节） | 1/2/3 |

> `tud_midi_stream_write()` 会自动按 MIDI 状态字节算 CIN 并打包，应用通常无需手填 CIN；`tud_midi_packet_write()` 则要求调用方提供完整 4 字节包。

### 3. 发送 Note On / Note Off（流式 API）

最简模式：`tud_midi_stream_write(cable_num, midi_bytes, len)` 自动按状态字节分类（取自 `examples/device/midi_test/src/main.c`）：

```c
void midi_task(void) {
  static uint32_t start_ms = 0;
  uint8_t const cable_num = 0;   // USB 端点关联的 MIDI jack
  uint8_t const channel   = 0;   // 0 = 通道 1

  // 重要：始终要读走主机发来的输入，否则发送端会阻塞（即使你只是丢弃）
  while (tud_midi_available()) {
    uint8_t packet[4];
    tud_midi_packet_read(packet);
  }

  if (board_millis() - start_ms < 286) return;   // ~286ms 一拍
  start_ms += 286;

  // Note On：状态 0x90|channel，音符，力度 127
  uint8_t note_on[3]  = { 0x90 | channel, note_sequence[note_pos], 127 };
  tud_midi_stream_write(cable_num, note_on, 3);

  // 上一音符 Note Off：状态 0x80|channel，音符，力度 0
  uint8_t note_off[3] = { 0x80 | channel, note_sequence[previous], 0 };
  tud_midi_stream_write(cable_num, note_off, 3);
}
```

> **务必读输入**：MIDI 接口描述符总是同时声明 IN/OUT jack，即使你不处理输入也要把接收 FIFO 读空，否则主机端的某些发送软件会阻塞。示例的 `while(tud_midi_available()) tud_midi_packet_read(...)` 即为此。

### 4. 收发原始 4 字节包

需要精确控制 CIN（如 SysEx 分包）时用包式 API：

```c
// 发：手工拼 4 字节包
uint8_t pkt[4] = {
  (uint8_t)((cable_num << 4) | MIDI_CIN_NOTE_ON),   // cable<<4 | CIN
  0x90 | channel,                                    // status
  note,                                              // data1
  127                                                // data2 (velocity)
};
tud_midi_packet_write(pkt);

// 批量写（一次写多个 4 字节包，返回成功写入的包数）
uint32_t n = tud_midi_packet_write_n(packets, n_packets);

// 收：读一个 4 字节包
uint8_t packet[4];
if (tud_midi_packet_read(packet)) {
  uint8_t cable = packet[0] >> 4;
  uint8_t cin   = packet[0] & 0x0F;
  // packet[1..3] 为 MIDI 字节
}
```

### 5. 接收回调

主机发来数据时栈触发回调（任务上下文），可在其中处理或置标志：

```c
void tud_midi_rx_cb(uint8_t itf) {
  (void)itf;
  // 在此调 tud_midi_packet_read / tud_midi_stream_read 取数据
}
```

### 6. 多实例 / 多 cable

`CFG_TUD_MIDI > 1` 时用带 `itf` 的 `tud_midi_n_*` API；单个 MIDI 接口内的多根 jack 用 `cable_num` 区分（描述符里 `_numcables` 控制每个端点关联的 jack 数，由 `TUD_MIDI_DESC_HEAD` / `TUD_MIDI_DESC_EP` 自动展开）。

### 7. 描述符（单 cable）

```c
// TUD_MIDI_DESC_LEN 已含 head + jack + 2 个端点
TUD_MIDI_DESCRIPTOR(ITF_NUM_MIDI, 0, EPNUM_MIDI_OUT, EPNUM_MIDI_IN, CFG_TUD_MIDI_EP_BUFSIZE),
```

`TUD_MIDI_DESCRIPTOR(itfnum, stridx, epout, epin, epsize)` 自动展开成：标准音频类接口 + MIDI 流接口 + 1 个 Embedded Jack In/Out + 1 对 bulk 端点。多 cable 时改用 `TUD_MIDI_DESC_HEAD + TUD_MIDI_DESC_JACK_DESC + TUD_MIDI_DESC_EP` 手工拼接。

## 常见错误

| 错误 | 原因 | 解决方法 |
|---|---|---|
| 主机看不到 MIDI 端口 | 描述符缺 jack / endpoint 描述符 | 用 `TUD_MIDI_DESCRIPTOR` 整体展开；接口 subclass 必须是 `AUDIO_SUBCLASS_MIDI_STREAMING` |
| 发送端（如 DAW）卡死 | 没读输入 FIFO | 即使不发也 `while(tud_midi_available()) tud_midi_packet_read()` 读空 |
| 音符不响 | CIN 错或 cable 错 | 用 `tud_midi_stream_write` 自动算 CIN；`cable_num` 与描述符 jack 编号对应（默认 0） |
| SysEx 收发乱 | 用了 3 字节 stream API | SysEx 必须用包式 API，按 `MIDI_CIN_SYSEX_START/END_*` 分包 |
| 力度/通道错 | 状态字节位运算错 | Note On = `0x90 \| channel`（channel 0-15），力度在字节 3 |
| 多实例串台 | 单实例 API 用错实例 | `CFG_TUD_MIDI>1` 时用 `tud_midi_n_*(itf, ...)` |
| 丢包 | EP_BUFSIZE 太小或一帧写太多 | 增大 `CFG_TUD_MIDI_EP_BUFSIZE`；批量用 `tud_midi_packet_write_n` |

## 参考

- `examples/device/midi_test/src/main.c` — Note On/Off 旋律发送（stream API）
- `examples/device/midi_test_freertos/` — 同上 + FreeRTOS 版
- `examples/host/midi_rx/` — 主机接收 MIDI（含 `README_midi_host.md`）
- `src/class/midi/midi_device.h` — MIDI 设备 API（`tud_midi_packet_read/write`、stream、`tud_midi_rx_cb`）
- `src/class/midi/midi.h` — Code Index Number 枚举（`MIDI_CIN_*`）
- `src/device/usbd.h` — `TUD_MIDI_DESCRIPTOR` / `TUD_MIDI_DESC_HEAD` / `TUD_MIDI_JACKID_*`
