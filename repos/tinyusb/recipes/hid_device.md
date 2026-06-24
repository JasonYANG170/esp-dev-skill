# HID 设备（键盘/鼠标/手柄）

> **适用摘要**: 用 TinyUSB HID 类实现 USB HID 设备，包括 report descriptor、键盘/鼠标/游戏手柄 report 上报、多 report 链式续发与 `tud_hid_set_report_cb`。

## 触发意图

- "USB HID 设备"
- "TinyUSB 键盘/鼠标"
- "tud_hid_keyboard_report"
- "HID report descriptor"
- "USB 手柄/游戏控制器"

## 前置条件

| 条件 | 要求 |
|---|---|
| 参考示例 | `examples/device/hid_composite/src/main.c`、`examples/device/hid_boot_interface/` |
| 配置项 | `CFG_TUD_HID=1`（实例数）、`CFG_TUD_HID_EP_BUFSIZE` |

## 分步说明

### 1. tusb_config.h 使能 HID

```c
#define CFG_TUD_HID               1
#define CFG_TUD_HID_EP_BUFSIZE    64
```

### 2. 上报前先查 ready（端点忙则丢包）

```c
static void send_hid_report(uint8_t report_id, uint32_t btn) {
  if (!tud_hid_ready()) return;    // 端点未就绪直接跳过

  switch (report_id) {
    case REPORT_ID_KEYBOARD: {
      uint8_t keycode[6] = { 0 };
      keycode[0] = HID_KEY_A;
      tud_hid_keyboard_report(REPORT_ID_KEYBOARD, 0, keycode);  // 按下
      // 释放：tud_hid_keyboard_report(REPORT_ID_KEYBOARD, 0, NULL);
      break;
    }
    case REPORT_ID_MOUSE: {
      tud_hid_mouse_report(REPORT_ID_MOUSE, 0x00, delta, delta, 0, 0);
      break;
    }
    case REPORT_ID_CONSUMER_CONTROL: {
      uint16_t vol_down = HID_USAGE_CONSUMER_VOLUME_DECREMENT;
      tud_hid_report(REPORT_ID_CONSUMER_CONTROL, &vol_down, 2);
      break;
    }
    case REPORT_ID_GAMEPAD: {
      hid_gamepad_report_t report = { .x = 0, .y = 0, .z = 0, .rz = 0,
                                      .rx = 0, .ry = 0, .hat = 0, .buttons = 0 };
      tud_hid_report(REPORT_ID_GAMEPAD, &report, sizeof(report));
      break;
    }
  }
}
```

### 3. 多 report 链式续发（用 `tud_hid_report_complete_cb`）

```c
// 一帧内要发多个 report：先发第 1 个，后续在完成回调里顺序发
void hid_task(uint32_t btn) {
  send_hid_report(REPORT_ID_KEYBOARD, btn);
}

void tud_hid_report_complete_cb(uint8_t instance, uint8_t const *report, uint16_t len) {
  (void)instance; (void)len;
  uint8_t next_id = report[0] + 1u;       // report[0] 即 report id
  if (next_id < REPORT_ID_COUNT) {
    send_hid_report(next_id, board_button_read());
  }
}
```

### 4. 必需回调：report descriptor、get/set report

```c
// 必须实现：返回该实例的 HID report descriptor
uint8_t const *tud_hid_descriptor_report_cb(uint8_t instance) {
  (void)instance;
  return desc_hid_report;
}

// 主机读 report（GET_REPORT）
uint16_t tud_hid_get_report_cb(uint8_t instance, uint8_t report_id,
                               hid_report_type_t report_type,
                               uint8_t *buffer, uint16_t reqlen) {
  (void)instance; (void)report_id; (void)report_type; (void)buffer; (void)reqlen;
  return 0;
}

// 主机写 report（SET_REPORT，如 LED 灯、输出报告）
void tud_hid_set_report_cb(uint8_t instance, uint8_t report_id,
                           hid_report_type_t report_type,
                           uint8_t const *buffer, uint16_t bufsize) {
  (void)instance; (void)report_id; (void)report_type; (void)buffer; (void)bufsize;
}

// 可选：主机切换 boot/report 协议
void tud_hid_set_protocol_cb(uint8_t instance, uint8_t protocol) { (void)instance; (void)protocol; }
// 可选：主机设置 idle rate；返回 true 表示接受
bool tud_hid_set_idle_cb(uint8_t instance, uint8_t idle_rate) { (void)instance; (void)idle_rate; return true; }
```

### 5. 鼠标/键盘单次上报 API 速查

```c
tud_hid_keyboard_report(report_id, modifier, keycode[6]);          // 6 键
tud_hid_mouse_report(report_id, buttons, dx, dy, v_wheel, h_wheel); // 相对
tud_hid_abs_mouse_report(report_id, buttons, x, y, v, h);          // 绝对
tud_hid_gamepad_report(report_id, x,y,z,rz,rx,ry, hat, buttons);
tud_hid_report(report_id, data, len);                              // 通用 report
```

## 常见错误

| 错误 | 原因 | 解决方法 |
|---|---|---|
| 按键不上报 | 端点忙被覆盖 | 发前查 `tud_hid_ready()`；多 report 用 `report_complete_cb` 链式 |
| 主机识别为未知设备 | report descriptor 缺失或错误 | 必须实现 `tud_hid_descriptor_report_cb`，且长度与 `TUD_HID_DESCRIPTOR` 的 `_report_desc_len` 一致 |
| 键盘无释放事件 | 只发按下没发释放 | 按下后补发 `keycode=NULL` 的 report |
| `tud_hid_set_report_cb` 报错缺失 | 该回调为必需 | 即使不处理也要提供空实现 |
| 主机不切到 report 协议 | 默认 boot 协议 | 用 `tud_hid_get_protocol()` 查询，按需在 `set_protocol_cb` 里响应 |

## 参考

- `examples/device/hid_composite/src/main.c` — 键盘/鼠标/消费者/手柄/触控笔多 report
- `examples/device/hid_boot_interface/` — boot 接口键盘/鼠标
- `examples/device/hid_generic_inout/` — 通用 IN/OUT report
- `src/class/hid/hid_device.h` — HID 设备 API 与回调声明
