---
name: esp-amp-skill
description: >-
  AI Skill for ESP-AMP asymmetric multiprocessing firmware development on ESP32-C5/C6/P4.
  Used when users need to create, modify, or debug ESP-AMP projects, including
  maincore + subcore (LP/HP core) inter-core IPC: shared memory, software interrupt,
  event, virtqueue, RPMsg, RPC, build system configuration, subcore life-cycle
  management, and automatic light sleep power optimization.
  Trigger words: "ESP-AMP", "esp-amp", "AMP", "maincore", "subcore", "main core", "sub core", "LP core", "HP core", "inter-core", "IPC", "RPMsg", "virtqueue", "非对称多核", "主核", "副核", "跨核通信", "核间通信"
tags:
  - embedded
  - esp32
  - espressif
  - esp-idf
  - AMP
  - asymmetric-multiprocessing
  - multicore
  - RPMsg
  - RPC
  - IPC
license: Apache-2.0
compatibility: "Target chips: ESP32-C5 / ESP32-C6 (HP maincore + LP subcore), ESP32-P4 (HP maincore + HP subcore). Toolchain/build: ESP-IDF v5.3.1+ (C6/P4) or v5.5+ (C5), idf.py build with unified or separate build mode."
metadata:
  author: Community
  version: "1.0.0"
---

# esp-amp-skill

ESP-AMP 是 Espressif 提供的轻量级非对称多核（Asymmetric Multiprocessing，AMP）框架，运行于 ESP32 系列多核 SoC 之上。一个核（maincore，运行 IDF FreeRTOS）与另一个核（subcore，运行 bare-metal 或其他轻量环境）通过共享内存与核间中断协作。本 Skill 提供基于真实仓库文档与源码的场景化 recipes、API/配置速查与陷阱清单，帮助 AI agent 正确开发 ESP-AMP 固件。

## Core Principles

1. **先 `esp_amp_init()`，再创建资源** — maincore 与 subcore 两端都必须先调用 `esp_amp_init()`，再使用 SysInfo / Event / RPMsg 等组件。`esp_amp_init()` 内部完成 SysInfo、Software Interrupt、Event 的初始化。
2. **SysInfo 仅由 maincore 分配，subcore 只读取** — `esp_amp_sys_info_alloc()` 只能在 maincore 启动 subcore 之前调用；subcore 通过 `esp_amp_sys_info_get()` 按 ID 获取。SysInfo 一旦分配无法释放，会保留到系统复位。ID 范围 `0x0000`~`0xfeff` 用户可用，`0xff00`~`0xffff` 为 ESP-AMP 内部保留。
3. **`EVENT_SUBCORE_READY` 握手是必须的** — maincore `esp_amp_start_subcore()` 后必须 `esp_amp_event_wait(EVENT_SUBCORE_READY, true, true, 10000)` 等待 subcore 通知就绪，之后才能与其通信；subcore 必须在初始化完成后调用 `esp_amp_event_notify(EVENT_SUBCORE_READY)`。
4. **Event 是单向的** — 单个 ESP-AMP event 只能从一个核通知另一个核。若需双向同步，必须创建两个 event（maincore 用 `esp_amp_event_create()` 分配 SysInfo ID）。禁止在同一个核上同时 `notify()` 与 `clear()` 同一个 event。
5. **FreeRTOS 环境 Event 必须绑定 EventGroup** — 在 maincore（FreeRTOS）侧，用 `xEventGroupWaitBits()` 等待的 ESP-AMP event 必须通过 `esp_amp_event_bind_handle()` 绑定到 EventGroupHandle，否则阻塞任务永远不会被唤醒。
6. **RPMsg send/deliver 必须成对** — `esp_amp_rpmsg_create_message()` 必须与 `esp_amp_rpmsg_send_nocopy()` 成对使用；`esp_amp_rpmsg_send()` 必须单独使用（不要再调用 create_message）。接收端使用完消息缓冲后必须调用 `esp_amp_rpmsg_destroy()`。误用会导致缓冲泄漏（类似内存泄漏）。
7. **Virtqueue 是单生产者-单消费者** — `esp_amp_queue_alloc_try()`/`send_try()` 仅 master core 调用，`recv_try()`/`free_try()` 仅 remote core 调用。混用或不成对调用会导致缓冲永久不可用。
8. **subcore 软件中断处理函数返回值被忽略；maincore 必须返回是否需要上下文切换** — maincore 处理函数返回 1 表示唤醒了高优先级任务，公共处理函数据此调用 `portYIELD_from_ISR()`。maincore 中断处理函数需要 `IRAM_ATTR`，subcore 不需要（整段固件已在内部 RAM）。
9. **subcore 不支持浮点打印；LP subcore 不支持硬件 FPU** — HP/LP subcore 均不支持将浮点数打印到控制台。LP core 全部浮点运算为软件实现，建议 LP subcore 仅使用整数运算。
10. **shared memory 只能放在 HP RAM 才能保证原子性** — RTC RAM 不支持 CAS/原子操作，不能用于 virtqueue、event、software interrupt 这类需要原子性的对象。需要原子性的对象必须使用 `SYS_INFO_CAP_HP`。
11. **subcore 固件从 HP RAM 加载，maincore 与 subcore 不得在加载期间并发访问同一外设** — `driver_init()` 会复位外设寄存器，若该外设正被另一核使用会产生非预期结果。同一 HP 外设在单次操作周期内（init→deinit）只能由一个核访问。
12. **内存布局由 sdkconfig 决定，配置错误会导致 build 失败或运行时崩溃** — `CONFIG_ESP_AMP_HP_SHARED_MEM_SIZE`、`CONFIG_ESP_AMP_SUBCORE_USE_HP_MEM_SIZE`、`CONFIG_ESP_AMP_SUBCORE_STACK_SIZE_MIN` 必须满足目标芯片范围约束，且 maincore 与 subcore（separate build 时）必须保持一致。

## When to Use

**Applicable:**
- 创建新的 ESP-AMP 双核工程（maincore + subcore）
- 配置 unified build 或 separate build 构建系统
- 实现 maincore/subcore 间 IPC：共享内存、软件中断、Event、Virtqueue、RPMsg、RPC
- 管理 subcore 生命周期：加载固件、启动/停止、panic 处理、printf 路由
- LP subcore 自动 light sleep 低功耗优化与 RTC RAM 内存放置
- subcore 外设驱动开发（HP `_ll.h` 或 LP ULP 组件）

**Not applicable:**
- 单核 ESP-IDF 应用（无 subcore 需求）
- ESP32（双 HP 核）的 IDF 原生 SMP（直接用 FreeRTOS SMP，不需要 ESP-AMP）
- OpenAMP / rpmsg-lite 等其他 AMP 框架
- PCB 硬件设计或原理图

---

## Scenario Quick Reference (Recipes)

当用户意图匹配下列场景时，**先阅读对应 recipe**——其中包含完整调用链、分步说明、可复制代码与常见错误。

### 工程与构建

| recipe | 场景 |
|---|---|
| `recipes/unified_build.md` | Unified build：单条命令同时构建 maincore + subcore 固件，支持嵌入或分区存储 |
| `recipes/separate_build.md` | Separate build：分别构建两侧固件，手动烧录 subcore 到 flash 分区 |
| `recipes/lifecycle.md` | subcore 生命周期管理：加载固件、启动、停止、panic 处理、printf 路由 |

### 核间通信

| recipe | 场景 |
|---|---|
| `recipes/shared_memory.md` | 共享内存与 SysInfo：maincore 分配、subcore 按 ID 读取 |
| `recipes/software_interrupt.md` | 软件中断：注册处理函数、触发对端核、多 handler 复用 |
| `recipes/event.md` | Event 同步：创建、绑定 EventGroup、通知、等待/轮询 |
| `recipes/virtqueue.md` | Virtqueue 单向队列：master/remote 角色、alloc/send/recv/free |
| `recipes/rpmsg.md` | RPMsg 双向通信：设备初始化、端点创建、send/nocopy/destroy |
| `recipes/rpc.md` | RPC 框架：client/server 初始化、命令执行、超时与 abort |

### 低功耗与外设

| recipe | scenario |
|---|---|
| `recipes/light_sleep.md` | LP subcore 自动 light sleep：PM 配置、RTC RAM 内存放置、skip/resume 宏 |
| `recipes/subcore_peripheral.md` | subcore 外设驱动：HP `_ll.h`、LP ULP 组件、避免并发访问 |

---

## 支持的 SoC 与核配置

| SoC | 所需 IDF 版本 | maincore | subcore | subcore 环境 |
| :---: | :---: | :---: | :---: | :---: |
| ESP32-C5 | v5.5 及以上 | HP core | LP core | bare-metal |
| ESP32-C6 | v5.3.1 及以上 | HP core | LP core | bare-metal |
| ESP32-P4 | v5.3.1 及以上 | HP core | HP core | bare-metal（仅 HP，LP 暂不支持） |

> subcore 类型由 Kconfig 自动选择：C5/C6 为 `CONFIG_ESP_AMP_SUBCORE_TYPE_LP_CORE`，P4 为 `CONFIG_ESP_AMP_SUBCORE_TYPE_HP_CORE`（并自动 `select FREERTOS_UNICORE`）。详见 `resources/config_reference.md`。

## ESP-AMP 分层架构与组件

| 层 | 组件 | 共享内存池 | 原子性要求 | 备注 |
|---|---|---|---|---|
| 基础 | Shared Memory / SysInfo | HP RAM / RTC RAM | 分配阶段无并发即可 | RTC RAM 不支持原子操作 |
| 基础 | Software Interrupt | HP RAM | 必须（一对原子整数） | 最多 32 个源，PMU（LP）/ INTMTX（HP）触发 |
| 同步 | Event | HP RAM | 必须（32-bit 原子整数） | 单向；FreeRTOS 需绑定 EventGroup |
| 传输 | Virtqueue (Queue) | HP RAM（标志位在 RTC RAM） | 单生产者-单消费者环 | 仅支持 Packed Virtqueue |
| 传输 | RPMsg | HP RAM | 同 Virtqueue | 基于 Virtqueue，多端点复用 |
| 应用 | RPC | HP RAM | 同 RPMsg | 基于 RPMsg；同一时刻仅 1 个 pending 请求 |

> SysInfo 保留 ID：`SYS_INFO_RESERVED_ID_SW_INTR`、`SYS_INFO_RESERVED_ID_EVENT_MAIN`、`SYS_INFO_RESERVED_ID_EVENT_SUB`、`SYS_INFO_RESERVED_ID_VQUEUE`、`SYS_INFO_RESERVED_ID_SYSTEM`、`SYS_INFO_RESERVED_ID_PM`。

## subcore printf 与 panic 工作流

subcore 调用 `printf()` 实际执行 `esp_amp_printf()`，根据 Kconfig 与运行时状态路由到不同输出：

| 条件 | 实际函数 | 说明 |
|---|---|---|
| `CONFIG_ESP_AMP_ROUTE_SUBCORE_PRINT=y` 且 system virtqueue 已初始化 | `virtqueue_send_char()` | 路由到 maincore 控制台（需 `CONFIG_ESP_AMP_SYSTEM_ENABLE_SUPPLICANT=y`） |
| 上述关闭 + LP subcore | `lp_subcore_send_char()` | LP UART |
| 上述关闭 + HP subcore | `hp_subcore_send_char()` | HP UART1 |
| system virtqueue 未初始化（panic 早期） | `esp_amp_early_printf()` | 直接写 maincore UART tx fifo |

> subcore panic 时将栈与寄存器转储到专用内存区并触发对 maincore 的软件中断，maincore 停止 subcore 后调用 `esp_amp_subcore_panic_handler_default()`（弱函数，可被覆盖）。

---

## Critical Pitfalls (Must Read)

下列是最高频的错误，违反任何一条都会导致不可用固件。

### 1. maincore 必须 wait `EVENT_SUBCORE_READY` 后再通信

```c
// ❌ WRONG — start_subcore 后立即使用 IPC，subcore 可能尚未初始化
esp_amp_start_subcore();
esp_amp_rpmsg_send(...);   // subcore 的 rpmsg_dev 还没初始化

// ✅ CORRECT — 等待握手
ESP_ERROR_CHECK(esp_amp_start_subcore());
assert((esp_amp_event_wait(EVENT_SUBCORE_READY, true, true, 10000)
        & EVENT_SUBCORE_READY) == EVENT_SUBCORE_READY);
// subcore 侧必须在 esp_amp_init() 与 IPC 初始化完成后调用：
// esp_amp_event_notify(EVENT_SUBCORE_READY);
```

### 2. RPMsg `send()` 与 `create_message()` 不可混用

```c
// ❌ WRONG — send() 内部已分配缓冲，create_message 会泄漏
void *msg = esp_amp_rpmsg_create_message(&rpmsg_dev, 32, ESP_AMP_RPMSG_DATA_DEFAULT);
esp_amp_rpmsg_send(&rpmsg_dev, &ept, 0, "hi", 2);  // 缓冲永不归还

// ✅ CORRECT — 方式 A：nocopy（推荐零拷贝）
void *msg = esp_amp_rpmsg_create_message(&rpmsg_dev, 32, ESP_AMP_RPMSG_DATA_DEFAULT);
snprintf(msg, 30, "data %d", n);
esp_amp_rpmsg_send_nocopy(&rpmsg_dev, &ept, 0, msg, 30);

// ✅ CORRECT — 方式 B：send（拷贝，单独使用）
esp_amp_rpmsg_send(&rpmsg_dev, &ept, 0, "hi", 2);
```

### 3. 接收端必须 `esp_amp_rpmsg_destroy()` 否则缓冲泄漏

```c
// ❌ WRONG — 处理完不归还
int ept_cb(void *msg_data, uint16_t len, uint16_t src, void *arg) {
    printf("got: %s\n", (char*)msg_data);
    return 0;   // 缓冲永久丢失
}

// ✅ CORRECT
int ept_cb(void *msg_data, uint16_t len, uint16_t src, void *arg) {
    printf("got: %s\n", (char*)msg_data);
    esp_amp_rpmsg_destroy(&rpmsg_dev, msg_data);  // 必须归还
    return 0;
}
```

### 4. FreeRTOS 端 Event 必须绑定 EventGroup

```c
// ❌ WRONG — 不绑定直接 wait，任务永远阻塞
uint32_t bits = esp_amp_event_wait_by_id(ID, MASK, true, false, 10000);

// ✅ CORRECT — 先绑定 EventGroup
EventGroupHandle_t eg = xEventGroupCreate();
assert(esp_amp_event_bind_handle(SYS_INFO_ID_SUBCORE_EVENT, eg) == 0);
// 之后即可用 xEventGroupWaitBits(eg, ...) 或 esp_amp_event_wait_by_id(...)
```

### 5. RPMsg 一端 poll=false 则对端 notify 必须为 true

```c
// ❌ WRONG — 两端都不 notify 且都不 poll，消息永远不会被处理
// maincore: esp_amp_rpmsg_main_init(&dev, 8, 128, false, false);
// subcore:  esp_amp_rpmsg_sub_init(&dev, false, false);

// ✅ CORRECT — 至少一端 notify，另一端可 poll
// maincore（中断模式）:
esp_amp_rpmsg_main_init(&rpmsg_dev, 32, 128, true, false);  // notify=true, poll=false
esp_amp_rpmsg_intr_enable(&rpmsg_dev);                       // poll=false 必须调用
// subcore（轮询模式）:
esp_amp_rpmsg_sub_init(&rpmsg_dev, true, true);              // notify=true, poll=true
```

### 6. Virtqueue send/free、recv/free 必须成对

```c
// ❌ WRONG — master 只 alloc 不 send，或 remote 只 recv 不 free
void *buf;
esp_amp_queue_alloc_try(&queue, &buf, 64);
// 忘记 send_try，缓冲卡在“已分配未发送”状态

// ✅ CORRECT — master 成对
void *buf;
if (esp_amp_queue_alloc_try(&queue, &buf, 64) == ESP_OK) {
    memcpy(buf, payload, 64);
    esp_amp_queue_send_try(&queue, buf, 64);
}
// remote 成对
void *buf; uint16_t size;
if (esp_amp_queue_recv_try(&queue, &buf, &size) == ESP_OK) {
    process(buf, size);
    esp_amp_queue_free_try(&queue, buf);
}
```

### 7. 软件中断 handler 在 maincore 需要 `IRAM_ATTR`

```c
// ❌ WRONG — maincore 中断函数在 Flash，可能因 cache miss 失败
static int handler(void *arg) {
    printf("intr\n");
    return 0;
}

// ✅ CORRECT — IRAM_ATTR 强制加载到内部 RAM；日志用 ESP_DRAM_LOGx
static const DRAM_ATTR char TAG[] = "sw";
static IRAM_ATTR int handler(void *arg) {
    (void)arg;
    ESP_DRAM_LOGI(TAG, "handler called");
    return 0;  // maincore 返回值表示是否需要上下文切换
}
// subcore 不需要 IRAM_ATTR（整段固件已在内部 RAM），返回值被忽略
```

### 8. subcore main 组件不能改名；subcore component 加 `sub` 前缀

```cmake
# ❌ WRONG — subcore 把 main 组件改名（ESP-AMP 不支持），或与 maincore 同名组件冲突
# subcore/components/greeting/  （与 maincore 某组件重名会覆盖）

# ✅ CORRECT — main 保持名为 main；组件加 sub 前缀
# subcore/components/sub_greeting/CMakeLists.txt
if(NOT SUBCORE_BUILD)
    idf_component_register()   # unified build 下仅为生成 sdkconfig，不实际编译
    return()
endif()
idf_component_register(SRCS "greeting.c" INCLUDE_DIRS ".")
```

### 9. 嵌入 subcore 固件仅 unified build 支持

```cmake
# ❌ WRONG — separate build 下用 EMBED（不支持，符号找不到）
# 且 maincore 用 _binary_xxx_bin_start 但未用 unified build

# ✅ CORRECT — 嵌入需 unified build + CONFIG 控制
# maincore/CMakeLists.txt
if(CONFIG_SUBCORE_FIRMWARE_EMBEDDED)
    esp_amp_add_subcore_project(${SUBCORE_APP_NAME} ${SUBCORE_PROJECT_DIR} EMBED)
else()
    esp_amp_add_subcore_project(${SUBCORE_APP_NAME} ${SUBCORE_PROJECT_DIR} PARTITION TYPE data SUBTYPE 0x40)
endif()
```

```c
// 加载嵌入固件（maincore）
extern const uint8_t subcore_xxx_bin_start[] asm("_binary_${SUBCORE_APP_NAME}_bin_start");
extern const uint8_t subcore_xxx_bin_end[]   asm("_binary_${SUBCORE_APP_NAME}_bin_end");
ESP_ERROR_CHECK(esp_amp_load_sub(subcore_xxx_bin_start));

// 从分区加载（maincore）
const esp_partition_t *p = esp_partition_find_first(ESP_PARTITION_TYPE_DATA, 0x40, NULL);
ESP_ERROR_CHECK(esp_amp_load_sub_from_partition(p));
```

### 10. LP subcore 不要用 RTC RAM 做原子对象

```c
// ❌ WRONG — 把 virtqueue/event 放在 RTC RAM，CAS 不支持，一致性破坏
esp_amp_sys_info_alloc(ID, sizeof(event), SYS_INFO_CAP_RTC);

// ✅ CORRECT — 需要原子性的对象用 HP RAM
esp_amp_sys_info_alloc(ID, sizeof(event), SYS_INFO_CAP_HP);
// RTC RAM 仅用于：light sleep 期间需要保留且不要求原子性的数据
```

### 11. 启用 heap 才能在 subcore 调用 malloc

```c
// ❌ WRONG — 未设 CONFIG_ESP_AMP_SUBCORE_ENABLE_HEAP=y 就 malloc
// 构建不报错，但运行时 malloc() 始终返回 -1
void *p = malloc(64);

// ✅ CORRECT — 启用 heap 并配置大小
// sdkconfig: CONFIG_ESP_AMP_SUBCORE_ENABLE_HEAP=y
//            CONFIG_ESP_AMP_SUBCORE_HEAP_SIZE=4096
// light sleep 启用时避免 malloc（heap 在 HP RAM，会阻止睡眠）
```

### 12. light sleep 的 skip/resume 宏必须成对

```c
// ❌ WRONG — 只 enter 不 exit，maincore 永远无法再进 light sleep
ESP_AMP_PM_SKIP_LIGHT_SLEEP_ENTER();
read_hp_register();
// 漏掉 EXIT()

// ✅ CORRECT — 成对包裹访问 HP RAM/外设的代码段
ESP_AMP_PM_SKIP_LIGHT_SLEEP_ENTER();
read_hp_register();
ESP_AMP_PM_SKIP_LIGHT_SLEEP_EXIT();
// 展开为 esp_amp_system_pm_subcore_skip_light_sleep() / ..._resume_light_sleep()
```

### 13. subcore 中断服务函数应使用 `RTC_IRAM_ATTR`（light sleep 时）

```c
// ❌ WRONG — light sleep 期间 ISR 不在 RTC RAM，无法执行
IRAM_ATTR void lp_isr(void) { ... }   // IRAM_ATTR 放 HP RAM，light sleep 时 clock-gated

// ✅ CORRECT — light sleep 安全的 ISR 用 RTC_IRAM_ATTR
RTC_IRAM_ATTR void lp_isr(void) { ... }
// 数据用 RTC_DATA_ATTR，常量用 RTC_RODATA_ATTR
```

---

## Execution Workflow

| Step | Name | Description |
|------|------|-------------|
| 1 | Plan | 确认目标芯片（C5/C6 LP subcore，P4 HP subcore）、所需 IPC 层级（共享内存/Event/RPMsg/RPC）、功耗需求（是否 light sleep） |
| 2 | Recipe | 在 `recipes/` 查找匹配场景的 recipe，遵循其调用链 |
| 3 | Query | recipe 未覆盖的 API 查 `resources/api_reference.md`；配置项查 `resources/config_reference.md` |
| 4 | Validate | 核对：`esp_amp_init()` 调用、SysInfo ID 范围、maincore/subcore 配置一致、内存布局 sdkconfig |
| 5 | Confirm | 向用户呈现实现方案：双核 CMake 结构、IPC 层级、握手流程、sdkconfig 修改 |
| 6 | Execute | **新工程**：以最接近的 `examples/<name>` 为模板复制修改；**现有工程**：原地编辑 |
| 7 | Check | 核对：握手 wait、send/destroy 成对、Event 绑定、subcore main 未改名、原子对象在 HP RAM |
| 8 | Build | `idf.py set-target <target>` → `idf.py build`；separate build 需分别构建并手动 esptool 烧录 subcore |
| 9 | Debug | `idf.py flash monitor` 观察 maincore；subcore printf 可经 supplicant 路由或独立 UART；检查 subcore panic dump |

### Step 6 Detail — 工程创建策略

**目标目录无现有工程（首次创建）：**

1. 根据 IPC 需求选择最接近的 example 作为模板：
   - 基础握手/共享内存 → `examples/rpmsg_send_recv/`（含完整 unified build 结构）
   - Event 同步 → `examples/event/`
   - 软件中断 → `examples/software_interrupt/`
   - Virtqueue 直接使用 → `examples/virtqueue/`
   - RPC client/server → `examples/rpc/maincore_client_subcore_server/` 或 `examples/rpc/subcore_client_maincore_server/`
   - 构建方式 → `examples/build_system/unified_build/` 或 `examples/build_system/separate_build/`
   - 低功耗 → `examples/light_sleep/`（仅 C5/C6 LP subcore）
2. 完整复制 example 目录（保留 `maincore/`、`subcore/`、`common/`、`partitions.csv`、`sdkconfig.defaults`、顶层 `CMakeLists.txt`、`subcore/subcore_config.cmake` 结构）。
3. 修改：subcore app name（`subcore_config.cmake` 中的 `app_name`）、`partitions.csv` 的 sub_core 条目、IPC 代码、sdkconfig。
4. 向用户说明复制了哪个 example 以及原因。

**目标目录已有工程：** 原地编辑，不要覆盖未明确要求的文件。

---

## Failure Strategies

| Situation | Action |
|---|---|
| API 在 `resources/api_reference.md` 找不到 | 立即停止，告知用户该 API 可能不存在；查对应 `docs/<component>.md` 与 `include/esp_amp_*.h` |
| `_binary_${SUBCORE_APP_NAME}_bin_start not found` | 检查：是否 unified build、`build/subcore/*.bin` 是否生成、符号名是否匹配 `build/*.bin.S`、map 文件 `.rodata.embedded` 段 |
| subcore 任务永不唤醒（FreeRTOS Event） | 检查 `esp_amp_event_bind_handle()` 是否成功调用；未绑定的 event 无法唤醒阻塞任务 |
| RPMsg 缓冲耗尽（create_message 返回 NULL） | 检查接收端是否调用 `esp_amp_destroy()`；检查 `queue_len` 是否够大；检查 send/create 是否误混用 |
| subcore malloc 返回 -1 | 启用 `CONFIG_ESP_AMP_SUBCORE_ENABLE_HEAP=y` 并设 `CONFIG_ESP_AMP_SUBCORE_HEAP_SIZE`；light sleep 启用时避免 malloc |
| subcore build 失败“insufficient memory for stack” | 增大 `CONFIG_ESP_AMP_SUBCORE_STACK_SIZE_MIN` 或减小 `CONFIG_ESP_AMP_SUBCORE_USE_HP_MEM_SIZE` / 固件体积 |
| light sleep 不生效 | 确认 `CONFIG_PM_ENABLE=y` + `CONFIG_ESP_AMP_SYSTEM_AUTO_LIGHT_SLEEP_SUPPORT_ENABLE=y`；`esp_pm_configure()` 在 `EVENT_SUBCORE_READY` 之后调用；检查 skip/resume 是否成对 |
| subcore panic 后无输出 | 启用 `CONFIG_ESP_AMP_SYSTEM_ENABLE_SUPPLICANT=y`；panic 经软件中断到 maincore，由弱函数 `esp_amp_subcore_panic_handler_default()` 打印 |
| HP 外设在双核间行为异常 | 同一外设在单次 init→deinit 周期内只由一个核访问；maincore `driver_init()` 会复位寄存器，影响正在使用它的 subcore |

## References

- 场景 recipes → `recipes/` 目录
- API 速查（按组件分组） → `resources/api_reference.md`
- Kconfig/配置项速查 → `resources/config_reference.md`
- 陷阱汇总 → `resources/pitfalls.md`
- 真实 example 索引 → `resources/example_list.md`
- 分层架构/状态机 → `resources/architecture.md`
- 仓库原始文档 → `espressif-repos/esp-amp/docs/*.md`（`shared_memory.md` / `software_interrupt.md` / `event.md` / `queue.md` / `rpmsg.md` / `rpc.md` / `build_system.md` / `memory_layout.md` / `system.md` / `light_sleep.md` / `peripheral.md` / `port.md` / `subcore_build_tips.md`）
