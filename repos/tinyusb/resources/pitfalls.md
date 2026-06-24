# TinyUSB 高频陷阱汇总

> 与 `SKILL.md` 的 Critical Pitfalls 互���，此处按“现象”分类，便于排查。所有 API/宏均来自真实源码。

## 1. 设备完全无法枚举

- **主循环没调 `tud_task()`** → USB 事件堆在队列里不处理。修：while(1) 里加 `tud_task()`，RTOS 里跑高优先级任务。
- **USB ISR 没转发** → DCD 收不到硬件事件。修：在 USB 中断函数里调 `tusb_int_handler(rhport, true)`。
- **`CFG_TUD_ENABLED` 不是 1** 或 **`CFG_TUSB_MCU` 未定义** → 栈未初始化/编译报错。

## 2. CDC 数据收发问题

- **写了收不到** → `tud_cdc_write()` 后没 `tud_cdc_write_flush()`。
- **大数据丢包** → `CFG_TUD_CDC_TX_BUFSIZE` / `EP_BUFSIZE` 太小；高速设备设 512。
- **多路 CDC 串扰** → 单实例 API 用错；多实例用 `tud_cdc_n_read(itf,...)` 等 `_n_` API。
- **`tud_cdc_rx_cb` 里阻塞** → 回调在 `tud_task()` 上下文，阻塞会卡整个栈；只读 FIFO，耗时操作外移。

## 3. HID 上报问题

- **按键不上报** → 端点忙，发前必须 `tud_hid_ready()`。
- **一帧发多个 report 互相覆盖** → 用 `tud_hid_report_complete_cb(instance, report, len)` 链式续发。
- **识别为未知设备** → `tud_hid_descriptor_report_cb()` 缺失或长度与 `TUD_HID_DESCRIPTOR` 的 `report_desc_len` 不符。
- **键盘按住不释放** → 按下后要补发 `keycode=NULL` 的 report。
- **必需回调缺失** → `tud_hid_get_report_cb` / `tud_hid_set_report_cb` 即使空处理也必须实现。

## 4. MSC U 盘问题

- **Windows“请插入磁盘”** → 容量 < 8KB 或 `tud_msc_test_unit_ready_cb` 返回 false。最小 8KB（如 16 块 × 512）。
- **写后重启丢数据** → 后端是 RAM；要持久化改用 Flash，在 `tud_msc_write10_cb` 里擦写。
- **设备管理器代码 10** → `read10/write10` 越界或返回值错；加边界检查，失败用 `tud_msc_set_sense()` 并返回 -1。
- **`capacity_cb` 没设出参** → 必须 `*block_count = ...; *block_size = ...;`。
- **只读盘仍被写** → `tud_msc_is_writable_cb()` 应返回 false。

## 5. 描述符问题

- **`CONFIG_TOTAL_LEN` 与实际不符** → 用 `TUD_*_DESC_LEN` 求和，别硬编码。
- **CDC 接口重叠** → CDC-ACM 用 IAD 占 **2 个接口**（通知 + 数据），`ITF_NUM_TOTAL` 要含两者。
- **字符串乱码** → 字符串描述符须 UTF-16LE，首字节 = `TUSB_DESC_STRING<<8 | (2*chr+2)`。
- **端点冲突** → 部分 MCU（LPC17xx、CXD56，或定义了 `TUD_ENDPOINT_ONE_DIRECTION_ONLY`）端点号方向固定，按示例条件分支。
- **device class 与接口类不匹配** → CDC 用 `TUSB_CLASS_MISC` + IAD；单类可用 0x00 由接口定义。
- **VID/PID 复用旧驱动** → 多类组合用 `PID_MAP()` 位图生成唯一 PID。

## 6. Vendor / WebUSB

- **`tud_vendor_read/write` 链接失败** → `CFG_TUD_VENDOR_RX_BUFSIZE=0`（零缓冲）时这些 API 不可用；数据走 `tud_vendor_rx_cb(itf, buffer, bufsize)`。
- **WebUSB 不识别** → 缺 BOS 描述符（`tud_descriptor_bos_cb`）或 `tud_vendor_control_xfer_cb`。
- **WinUSB 驱动加载错** → 缺 MS OS 2.0 兼容描述符（参考 `examples/device/webusb_serial/`）。

## 7. 主机栈问题

- **设备插上无反应** → `tuh_task()` 没在主循环。
- **Hub 下设备不枚举** → `CFG_TUH_HUB=1`。
- **HID 只收一帧** → `tuh_hid_report_received_cb` 末尾要再调 `tuh_hid_receive_report()`。
- **MSC 读不到容量** → 挂载后先 `tuh_msc_test_unit_ready` → `tuh_msc_read_capacity` → 再读写。
- **ESP32 主机不工作** → ESP32 无原生主机控制器，用 MAX3421E（`CFG_TUH_MAX3421=1`，via SPI），并用 `tuh_max3421_reg_write` 配 GPIO。
- **异步传输回调里阻塞** → 回调只做轻量处理；重活放主循环。

## 8. 构建/工具链问题

- **`#error CFG_TUSB_MCU must be defined`** → 由 board.mk / CMake 注入；ESP-IDF 由 IDF 配置。
- **找不到 `hw/mcu/<vendor>`** → 先 `python tools/get_deps.py <family>`（或 `-b <board>`）。
- **espressif / rp2040 用 make 报错** → 这两家族仅支持 CMake，改用 `cmake -DBOARD=...`。
- **与厂商 USB 中间件冲突** → 在 STM32CubeMX 里禁用 USB 代码生成，让 TinyUSB 接管（文档原话）。

## 9. 并发/线程安全（来自 `docs/reference/concurrency.rst`）

- **应用回调在任务上下文执行**，不是 ISR（除非标 `xfer_isr`）；可在回调里安全地入队 TX 数据。
- **类驱动可被多个任务同时调用**，访问共享 FIFO 时栈内部用信号量/互斥量；应用不要在 ISR 里调类驱动。
- **DCD/HCD 寄存器与包内存必须 `volatile`**；移植驱动里全局变量必须 `static`。
- **`in_isr` 参数**：带此参数的核心函数提醒“可能在中断上下文”，要小心。

## 10. 术语陷阱

- **IN/OUT 永远是设备视角**：IN = 设备→主机，OUT = 主机→设备。`tud_cdc_tx_complete_cb` 即“设备发完数据”。
- **`tud_*` 设备 / `tuh_*` 主机**，不要混用。
- **`tu_*` 是内部工具**，应用一般不调用。
