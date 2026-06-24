# ESP-AMP 分层架构与数据流

> 来自 `espressif-repos/esp-amp/README.md` 与各组件 docs。本仓库无显式状态机文档；本文件汇总分层模型、SysInfo/Event 数据结构、subcore 生命周期与 printf/panic 流程。

## 分层架构（自底向上）

```
┌──────────────────────────────────────────────┐
│  Application: RPC（基于 RPMsg，远程过程调用）  │  ← 应用层
├──────────────────────────────────────────────┤
│  Transport: RPMsg（基于一对 Virtqueue，多端点）│
├──────────────────────────────────────────────┤
│  Link: Virtqueue / Queue（Packed Virtqueue，   │  ← 传输/链路层
│        单生产者-单消费者无锁环）               │
├──────────────────────────────────────────────┤
│  Sync: Event（32-bit 原子位图，单向通知）      │
├──────────────────────────────────────────────┤
│  Notify: Software Interrupt（PMU/INTMTX 核间） │
├──────────────────────────────────────────────┤
│  Base: Shared Memory / SysInfo（HP/RTC RAM）   │  ← 基础层
├──────────────────────────────────────────────┤
│  Port: Platform API + Environment API         │  ← 抽象层
│        （抽象 SoC 差异与 FreeRTOS/bare-metal） │
└──────────────────────────────────────────────┘
```

- 上层依赖下层；可选择不同层组合满足需求。
- Port 层抽象 SoC（`esp_amp_platform_*`）与运行环境（`esp_amp_env_*`），上层无需关心 PMU vs INTMTX、FreeRTOS vs bare-metal。

## SysInfo 数据结构

```
SysInfo = [ entry, entry, ... ]
entry = (id: uint16, size: uint16, addr: uint32)   // 每条目三元组
```
- 默认最多 16 条目（可配）。
- ESP-AMP 内部已占用 6 个保留 ID（见 `api_reference.md`）。
- 无 free API；分配后保留到系统复位。
- 两个池：`SYS_INFO_CAP_HP`（HP RAM，支持原子）/ `SYS_INFO_CAP_RTC`（RTC RAM，不支持原子）。

## Virtqueue 数据流（master → remote 单向）

```
1. master: esp_amp_queue_alloc_try()   → 预留 available 缓冲，返回指针
2. master: 原地写入数据
3. master: esp_amp_queue_send_try()    → 标记 used(free)，触发 notify_cb（→ sw_intr_trigger）
                                      [此后 master 不可再访问该缓冲]
4. remote: esp_amp_queue_recv_try()    → 预留 used(free) 缓冲，返回指针
5. remote: 原地读取/处理数据
6. remote: esp_amp_queue_free_try()    → 标记 available，归还 master
                                      [此后 remote 不可再访问该缓冲]
```
- master 与 remote 由 `is_master` 参数决定，与 maincore/subcore 角色无关。
- 仅支持 Packed Virtqueue（比 Split Virtqueue 更小更快）。

## RPMsg 数据流（双向，多端点复用）

- 一个 `esp_amp_rpmsg_dev_t` 内部用两个 Virtqueue（TX/RX）。
- 多个 endpoint 复用同一对 Virtqueue（按 `dst_addr` 路由，类似 TCP/IP 端口）。

发送两种模式：
```
零拷贝：create_message() → 原地写 → send_nocopy()    [必须成对]
拷贝  ：send()                                          [单独使用，不可与 create 混用]
```
接收：endpoint 回调自动触发（中断或轮询），用完必须 `destroy()`。

## Event 数据流（单向位图通知）

```
notifier: esp_amp_event_notify_by_id(id, mask)
   → atomic_fetch_or 到 SysInfo 中的 32-bit 原子整数
   → FreeRTOS: xEventGroupSetBitsFromISR 唤醒等待任务（需先 bind_handle）
   → bare-metal: 等待侧 poll/wait

waiter: esp_amp_event_wait_by_id(id, mask, clear, all, timeout)
   → FreeRTOS: xEventGroupWaitBits（阻塞）
   → bare-metal: atomic_cmp_exchange 忙等
```
- 单向：一个 event 只能从一个核通知另一个核。双向同步需建两个 event。
- `EVENT_SUBCORE_READY` 走内置保留事件，用便捷宏 `esp_amp_event_notify/wait`。

## subcore 生命周期状态流

```
[maincore] esp_amp_init()
           ↓
           创建 SysInfo/Event/RPMsg 资源（maincore only）
           ↓
           esp_amp_load_sub() / esp_amp_load_sub_from_partition()  → 固件入 HP RAM
           ↓
           esp_amp_start_subcore()  → 启动 subcore
           ↓
[subcore]  esp_amp_init() → IPC sub_init
           ↓
           esp_amp_event_notify(EVENT_SUBCORE_READY)  ←─── 握手
           ↓                                                ↑
[maincore] esp_amp_event_wait(EVENT_SUBCORE_READY) ──────┘ 等到后开始通信
           ↓
           业务运行（双向 IPC）
           ↓
           （异常）subcore panic
              → 转储栈/寄存器到专用区 → 触发 sw_intr 到 maincore
              → maincore 停止 subcore → supplicant 任务调 panic_handler_default（弱函数）
           ↓
           esp_amp_stop_subcore()  （可选停止）
```

## subcore printf 路由决策流

```
subcore 调 printf()
   ↓ 实际为 esp_amp_printf() → esp_amp_putchar()
   ↓
   ├─ CONFIG_ESP_AMP_ROUTE_SUBCORE_PRINT=y 且 system virtqueue 已初始化
   │     → virtqueue_send_char() → maincore supplicant → maincore 控制台
   ├─ 上述关闭 + LP subcore
   │     → lp_subcore_send_char() → LP UART
   ├─ 上述关闭 + HP subcore
   │     → hp_subcore_send_char() → HP UART1
   └─ system virtqueue 未初始化（panic 早期）
         → esp_amp_early_printf() → 直接写 maincore UART tx fifo
```

## 内存布局（HP RAM，按目标）

| 目标 | subcore+共享+panic 可用 HP RAM | 分配方向 | panic dump |
|---|---|---|---|
| ESP32-C6 | 63KB（0x4087F560 向下） | 高→低，最高 4KB 保留 panic | 顶部 4KB |
| ESP32-C5 | 较 C6 小（HP RAM 更小） | 同上 | 顶部 4KB |
| ESP32-P4 | 256KB（0x4FF40000~0x4FF80000），cache 仅支持 128K/256K | 0x4FF80000 向下 | 顶部 |

> 未用 HP RAM 自动归还 maincore heap。subcore 栈：C5/C6 在 RTC RAM，P4 在 HP RAM。

## 与 OpenAMP 的区别（来自 FAQ）

ESP-AMP 受 OpenAMP 启发但更轻量，专为 LP core（C5/C6 仅 16KB RTC RAM）设计。OpenAMP 功能丰富但体积大、难移植到资源受限核。
