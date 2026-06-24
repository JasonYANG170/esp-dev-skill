# ESP-AMP 陷阱汇总

> 来自官方 docs 与示例代码的实战经验。每条配 WRONG/CORRECT 对照。与 `SKILL.md` 的 Critical Pitfalls 互补，这里补充更细的边角情况。

## 初始化与握手

### P1. `esp_amp_init()` 必须在所有 ESP-AMP API 之前

```c
// ❌
esp_amp_sys_info_alloc(ID, size, SYS_INFO_CAP_HP);   // SysInfo 未初始化
esp_amp_init();

// ✅
esp_amp_init();                                       // 内部 init SysInfo/SwIntr/Event
esp_amp_sys_info_alloc(ID, size, SYS_INFO_CAP_HP);
```

### P2. maincore 资源必须在 `start_subcore()` 之前创建

SysInfo alloc、Event create、RPMsg `main_init`、Virtqueue `main_init` 全部在 `esp_amp_start_subcore()` 之前完成；subcore 启动后只能 `get`/`sub_init`。

### P3. 必须握手 `EVENT_SUBCORE_READY`

```c
// ❌ start_subcore 后立即发消息，subcore 可能还没初始化 IPC
// ✅
assert((esp_amp_event_wait(EVENT_SUBCORE_READY, true, true, 10000) & EVENT_SUBCORE_READY) == EVENT_SUBCORE_READY);
```
subcore 侧：`esp_amp_init()` → IPC 初始化 → `esp_amp_event_notify(EVENT_SUBCORE_READY)`。

## Event

### P4. FreeRTOS 端未绑定 EventGroup 导致任务永不唤醒

```c
// ❌ 直接 wait，任务永远阻塞
// ✅
EventGroupHandle_t eg = xEventGroupCreate();
esp_amp_event_bind_handle(SYS_INFO_ID_SUBCORE_EVENT, eg);
// 之后 xEventGroupWaitBits(eg, ...) 或 esp_amp_event_wait_by_id(...)
```

### P5. wait/clear 非原子丢事件

设 `clear_on_exit=true`。timeout 返回时不清除（语义安全）。

### P6. 单 event 双向使用 / 同核 notify+clear

Event 是单向的。双向同步建两个 event。set 与 clear 必须分属不同核，否则丢失顺序信息。

### P7. bit_mask 用了高 8 位

FreeRTOS 保留高 8 位，仅用低 24 位。

## RPMsg

### P8. `send()` 与 `create_message()` 混用导致缓冲泄漏

```c
// ❌ send 内部已 alloc，create_message 的缓冲永不归还
// ✅ 零拷贝：create_message + send_nocopy 成对
// ✅ 拷贝：send 单独使用
```

### P9. 接收端不 `destroy()` 导致缓冲耗尽

```c
int ept_cb(void *data, uint16_t len, uint16_t src, void *arg) {
    process(data);
    esp_amp_rpmsg_destroy(&rpmsg_dev, data);   // 必须
    return 0;
}
```

### P10. notify/poll 两端不互补

一端 `poll=false` 则对端 `notify=true`；`poll=false` 侧必须随后 `esp_amp_rpmsg_intr_enable()`。两端都 poll=false 或都不 notify，消息永远不处理。

### P11. ISR 回调里调非 ISR-safe API

中断模式下端点回调在 ISR 执行。用 `esp_amp_env_in_isr()` 判断，仅 ISR-safe API；用 `ESP_DRAM_LOGx` 替代 `ESP_LOGx`。`create_endpoint/delete/rebind/search` 严禁 ISR 上下文。

### P12. 数据超过 `queue_item_size` 被拒

`create_message` 返回 NULL、`send`/`send_nocopy` 返回 -1。拆包或初始化时设大 `queue_item_size`；用 `esp_amp_rpmsg_get_max_size()` 查询。

## Virtqueue

### P13. alloc/send、recv/free 不成对

缓冲永久卡住。master: alloc→send；remote: recv→free。

### P14. 用非 alloc/recv 的指针调 send/free

未定义行为。send 的 buffer 必须来自 alloc_try；free 的必须来自 recv_try。

### P15. 角色 master/remote 调错 API

master: alloc/send；remote: recv/free。`intr_enable` 仅 remote 调用。

### P16. notify_func 的 sw_intr_id 与 remote `intr_enable` 不一致

对端收不到中断。两端 id 必须一致。

## Software Interrupt

### P17. maincore handler 缺 `IRAM_ATTR`

cache miss 时失败。maincore 加 `IRAM_ATTR`，TAG 用 `DRAM_ATTR`，日志用 `ESP_DRAM_LOGx`。subcore 不需要（整段固件已在内部 RAM）。

### P18. 返回值语义混淆

maincore 返回 1=唤醒高优先级任务（公共处理函数据此 `portYIELD_from_ISR`）；subcore 返回值被忽略。

### P19. 误用保留 ID

仅用 `SW_INTR_ID_0`~`SW_INTR_ID_15`；`SW_INTR_RESERVED_ID_*`（含 RPMSG/EVENT/SYS_SVC/PANIC）为内部用。

## 构建与工程结构

### P20. unified build 下顶层 CMake 未在 project() 前 include subcore_config.cmake

`SUBCORE_APP_NAME`/`SUBCORE_PROJECT_DIR` 未定义。必须 `project()` 之前 include。

### P21. subcore main 组件改名

ESP-AMP 不支持。subcore 的 main 必须名为 `main`。

### P22. subcore component 与 maincore 同名冲突

加 `sub_` 前缀。unified build 下用 `if(NOT SUBCORE_BUILD) idf_component_register() return() endif()` 守卫，仅为生成 sdkconfig。

### P23. 嵌入模式符号找不到

`_binary_${SUBCORE_APP_NAME}_bin_start`。检查：是否 unified build、`build/subcore/*.bin` 是否生成、符号名是否匹配 `build/*.bin.S`、map 文件 `.rodata.embedded` 段。

### P24. separate build 两侧 sdkconfig 不一致

ESP-AMP 相关项必须手动一致。建议两边 `sdkconfig.defaults` 同步。

### P25. partitions.csv 的 type/subtype 与 `esp_amp_add_subcore_project` / `esp_partition_find_first` 不一致

对齐 `TYPE data SUBTYPE 0x40`。烧录 offset 取 `partitions.csv` 中 sub_core 条目的 Offset。

## subcore 运行时

### P26. 未启用 heap 就 malloc

构建不报错，运行时 malloc 返回 -1。设 `CONFIG_ESP_AMP_SUBCORE_ENABLE_HEAP=y` + `CONFIG_ESP_AMP_SUBCORE_HEAP_SIZE`。

### P27. subcore 打印浮点

HP/LP subcore 均不支持浮点打印。LP core 浮点全软件，建议 LP subcore 仅用整数。

### P28. stack 不足覆盖 heap/bss

subcore 不支持栈溢出检测。栈可能无声覆盖 heap/bss。设足够大 `CONFIG_ESP_AMP_SUBCORE_STACK_SIZE_MIN`。

### P29. subcore panic 无输出

启用 `CONFIG_ESP_AMP_SYSTEM_ENABLE_SUPPLICANT=y`。

## Light Sleep（LP subcore）

### P30. HP core 睡眠时 LP subcore 访问 HP RAM/外设 → 挂起

用 `ESP_AMP_PM_SKIP_LIGHT_SLEEP_ENTER/EXIT` 成对包裹。

### P31. skip/resume 不成对 → maincore 永远不再睡

每处 ENTER 必须有对应 EXIT。

### P32. ISR 用 `IRAM_ATTR`（放 HP RAM，睡眠时不可执行）

light sleep 时 ISR 改用 `RTC_IRAM_ATTR`；数据 `RTC_DATA_ATTR`；常量 `RTC_RODATA_ATTR`。

### P33. heap 操作阻止睡眠

heap 在 HP RAM。light sleep 启用时避免 malloc；必要时包裹 skip/resume。

### P34. `esp_amp_event_poll()` 在 LP subcore 忙等阻止睡眠

light sleep 下改用软件中断做 HP→LP 通知。

## 外设

### P35. 双核并发访问同一 HP 外设

`driver_init()` 复位寄存器，破坏另一核正在使用的外设。同一外设单次 init→deinit 周期内只由一个核访问。

### P36. LP subcore 上 HP 外设中断不触发

当前不支持。LP subcore 上 HP 外设仅轮询。HP 外设中断需 HP subcore（P4）。

### P37. subcore 用 IDF driver 组件失败

driver 假设 FreeRTOS。subcore 用 `_ll.h`（HP 外设）或 ulp 组件（LP 外设）。
