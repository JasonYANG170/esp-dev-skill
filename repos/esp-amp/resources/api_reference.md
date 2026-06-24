# ESP-AMP API 速查（按组件分组）

> 所有签名来自 `espressif-repos/esp-amp/components/esp_amp/include/*.h` 与 `port/include/*.h`、`system/include/esp_amp_system.h`。`#include "esp_amp.h"` 即聚合所有组件头（maincore 额外含 `esp_amp_system.h`，由 `IS_MAIN_CORE` 控制）。

## 初始化（顶层）

```c
// Header: esp_amp.h
int esp_amp_init(void);   // retval 0=成功, -1=失败。内部完成 SysInfo/SwIntr/Event 初始化
```

## SysInfo / 共享内存（esp_amp_sys_info.h）

```c
typedef enum {
    SYS_INFO_RESERVED_ID = 0xff00,
    SYS_INFO_RESERVED_ID_EVENT_MAIN,  // 保留：main core event（HP）
    SYS_INFO_RESERVED_ID_EVENT_SUB,   // 保留：sub core event（HP）
    SYS_INFO_RESERVED_ID_VQUEUE,      // 保留：packed virtqueue 数据与缓冲（HP）
    SYS_INFO_RESERVED_ID_SYSTEM,      // 保留：system service
    SYS_INFO_RESERVED_ID_PM,          // 保留：power management（RTC）
    SYS_INFO_ID_MAX = 0xffff,
} esp_amp_sys_info_id_t;

typedef enum {
    SYS_INFO_CAP_HP = 0,    // HP RAM 共享内存（支持原子操作）
    SYS_INFO_CAP_RTC = 1,   // RTC RAM 共享内存（不支持原子操作，仅 LP subcore）
} esp_amp_sys_info_cap_t;

// 仅 maincore，start_subcore 之前调用；返回 NULL 失败
void *esp_amp_sys_info_alloc(uint16_t info_id, uint16_t size, esp_amp_sys_info_cap_t cap);

// subcore 按 ID 获取（size 可传 NULL）
void *esp_amp_sys_info_get(uint16_t info_id, uint16_t *size, esp_amp_sys_info_cap_t cap);

int  esp_amp_sys_info_init(void);   // 内部由 esp_amp_init 调用
void esp_amp_sys_info_dump(void);   // 调试用
```
> 用户 ID 范围：`0x0000`~`0xfeff`；`0xff00`~`0xffff` 保留。SysInfo 无 free API，分配后保留到复位。

## Software Interrupt（esp_amp_sw_intr.h）

```c
typedef enum {
    SW_INTR_ID_0 = 0, ..., SW_INTR_ID_15,
    SW_INTR_RESERVED_ID_16, ..., SW_INTR_RESERVED_ID_RPMSG,
    SW_INTR_RESERVED_ID_EVENT, SW_INTR_ID_MAX = 31,
} esp_amp_sw_intr_id_t;

// handler 返回值：1=唤醒高优先级任务（maincore，公共处理函数据此 portYIELD_from_ISR）；subcore 忽略
typedef int (* esp_amp_sw_intr_handler_t)(void *arg);

int  esp_amp_sw_intr_add_handler(esp_amp_sw_intr_id_t intr_id, esp_amp_sw_intr_handler_t handler, void *arg);
void esp_amp_sw_intr_delete_handler(esp_amp_sw_intr_id_t intr_id, esp_amp_sw_intr_handler_t handler);
void esp_amp_sw_intr_trigger(esp_amp_sw_intr_id_t intr_id);   // 触发对端核
void esp_amp_sw_intr_handler_dump(void);                       // 调试
int  esp_amp_sw_intr_init(void);                               // 内部由 esp_amp_init 调用
```
> 最多 32 个源（高段保留）。handler 表大小由 `CONFIG_ESP_AMP_SW_INTR_HANDLER_TABLE_LEN`（默认 8）控制。maincore handler 需 `IRAM_ATTR`，subcore 不需要。

## Event（esp_amp_event.h）

```c
// 通知对端（位 OR 到原子整数；mask 仅低 24 位）
uint32_t esp_amp_event_notify_by_id(uint16_t sysinfo_id, uint32_t bit_mask);

// 等待（FreeRTOS 阻塞 / bare-metal 忙等；返回清除前的 bitmask）
uint32_t esp_amp_event_wait_by_id(uint16_t sysinfo_id, uint32_t bit_mask,
                                  bool clear_on_exit, bool wait_for_all, uint32_t timeout_ms);

// 清除（仅在等待侧调用，慎用，可能丢事件）
uint32_t esp_amp_event_clear_by_id(uint16_t sysinfo_id, uint32_t bit_mask);

// 便捷宏（走内置保留事件 SYS_INFO_RESERVED_ID_EVENT_MAIN / _SUB）
#if IS_MAIN_CORE
#define esp_amp_event_notify(bit_mask)            // notify subcore
#define esp_amp_event_wait(mask, clear, all, t)   // wait subcore 事件
#else
#define esp_amp_event_notify(bit_mask)            // notify maincore
#define esp_amp_event_wait(mask, clear, all, t)   // wait maincore 事件
#endif

#if IS_ENV_BM
#define esp_amp_event_poll_by_id(id, mask, clear, all)  esp_amp_event_wait_by_id(id, mask, clear, all, 0)
#define esp_amp_event_poll(mask, clear, all)            esp_amp_event_wait_by_id(SYS_INFO_RESERVED_ID_EVENT_MAIN, mask, clear, all, 0)
#endif

#if !IS_ENV_BM   // 仅 FreeRTOS
int  esp_amp_event_bind_handle(uint16_t sysinfo_id, void *event_handle);   // 绑定 EventGroupHandle
void esp_amp_event_unbind_handle(uint16_t sysinfo_id);
void esp_amp_event_table_dump(void);   // 调试
#endif

#if IS_MAIN_CORE
int esp_amp_event_create(uint16_t sysinfo_id);   // 仅 maincore，subcore 启动前
#endif

int esp_amp_event_init(void);   // 内部由 esp_amp_init 调用
```
> 单向：单个 event 只能单向通知。FreeRTOS 端必须 `bind_handle` 否则 wait 永不唤醒。表大小 `CONFIG_ESP_AMP_EVENT_TABLE_LEN`（默认 8）。

## Virtqueue / Queue（esp_amp_queue.h）

```c
typedef struct esp_amp_queue_desc_t { uint32_t addr; uint16_t len; uint16_t flags; } esp_amp_queue_desc_t;
typedef int (*esp_amp_queue_cb_t)(void*);   // callback（remote 收数据）或 notify（master 发完通知对端）
typedef struct esp_amp_queue_t { /* ... */ bool master; esp_amp_queue_cb_t callback_fc, notify_fc; void* priv_data; /* ... */ } esp_amp_queue_t;

// master core only
int esp_amp_queue_alloc_try(esp_amp_queue_t *queue, void **buffer, uint16_t size);   // 申请缓冲
int esp_amp_queue_send_try(esp_amp_queue_t *queue, void *buffer, uint16_t size);     // 发送

// remote core only
int esp_amp_queue_recv_try(esp_amp_queue_t *queue, void **buffer, uint16_t *size);   // 接收
int esp_amp_queue_free_try(esp_amp_queue_t *queue, void *buffer);                    // 释放

// remote core only：使能软件中断接收
int esp_amp_queue_intr_enable(esp_amp_queue_t *queue, esp_amp_sw_intr_id_t sw_intr_id);

// 低级初始化（一般用下面两个封装）
int esp_amp_queue_init_buffer(esp_amp_queue_conf_t* conf, uint16_t queue_len, uint16_t queue_item_size,
                              esp_amp_queue_desc_t* desc, void* buffer);
int esp_amp_queue_create(esp_amp_queue_t* queue, esp_amp_queue_conf_t* conf,
                         esp_amp_queue_cb_t cb_func, void* priv_data, bool is_master);

#if IS_MAIN_CORE
// maincore 初始化：queue_len 必须 2 的幂；is_master 决定 cb_func 角色
int esp_amp_queue_main_init(esp_amp_queue_t* queue, uint16_t queue_len, uint16_t queue_item_size,
                            esp_amp_queue_cb_t cb_func, void* priv_data, bool is_master,
                            esp_amp_sys_info_id_t sysinfo_id);
#endif

// subcore 初始化
int esp_amp_queue_sub_init(esp_amp_queue_t* queue, esp_amp_queue_cb_t cb_func, void* priv_data,
                           bool is_master, esp_amp_sys_info_id_t sysinfo_id);
```
> 单生产者-单消费者。master: alloc→send；remote: recv→free（均成对）。`ESP_ERR_*` 返回码（`ESP_OK` / `ESP_ERR_NO_MEM` / `ESP_ERR_NOT_FOUND` / `ESP_ERR_NOT_SUPPORTED` / `ESP_ERR_NOT_ALLOWED` / `ESP_ERR_INVALID_ARG`）。

## RPMsg（esp_amp_rpmsg.h）

```c
#define ESP_AMP_RPMSG_DATA_DEFAULT      (uint16_t)(0x0)
#define ESP_AMP_RPMSG_RESERVED_EPT_SYS_PRT (uint16_t)(UINT16_MAX)

typedef int (*esp_amp_ept_cb_t)(void* msg_data, uint16_t data_len, uint16_t src_addr, void* rx_cb_data);
typedef struct esp_amp_rpmsg_ept_t { esp_amp_ept_cb_t rx_cb; void* rx_cb_data; struct esp_amp_rpmsg_ept_t* next_ept; uint16_t addr; } esp_amp_rpmsg_ept_t;
typedef struct esp_amp_rpmsg_dev_t { esp_amp_queue_t* rx_queue; esp_amp_queue_t* tx_queue; esp_amp_rpmsg_ept_t* ept_list; esp_amp_queue_ops_t queue_ops; } esp_amp_rpmsg_dev_t;

// 初始化（maincore 必须先于 subcore）
int esp_amp_rpmsg_main_init(esp_amp_rpmsg_dev_t* dev, uint16_t queue_len, uint16_t queue_item_size, bool notify, bool poll);
int esp_amp_rpmsg_sub_init(esp_amp_rpmsg_dev_t* dev, bool notify, bool poll);
// 多设备时用 _by_id 指定不同 sysinfo_id
int esp_amp_rpmsg_main_init_by_id(esp_amp_rpmsg_dev_t* dev, esp_amp_queue_t vq[], uint16_t queue_len, uint16_t queue_item_size, bool notify, bool poll, esp_amp_sys_info_id_t sysinfo_id);
int esp_amp_rpmsg_sub_init_by_id(esp_amp_rpmsg_dev_t* dev, esp_amp_queue_t vq[], bool notify, bool poll, esp_amp_sys_info_id_t sysinfo_id);
// poll=false 时必须调用
int esp_amp_rpmsg_intr_enable(esp_amp_rpmsg_dev_t* dev);

// 端点管理（严禁 ISR 上下文）
esp_amp_rpmsg_ept_t* esp_amp_rpmsg_create_endpoint(esp_amp_rpmsg_dev_t* dev, uint16_t ept_addr, esp_amp_ept_cb_t cb, void* cb_data, esp_amp_rpmsg_ept_t* ept_ctx);
esp_amp_rpmsg_ept_t* esp_amp_rpmsg_delete_endpoint(esp_amp_rpmsg_dev_t* dev, uint16_t ept_addr);
esp_amp_rpmsg_ept_t* esp_amp_rpmsg_rebind_endpoint(esp_amp_rpmsg_dev_t* dev, uint16_t ept_addr, esp_amp_ept_cb_t cb, void* cb_data);
esp_amp_rpmsg_ept_t* esp_amp_rpmsg_search_endpoint(esp_amp_rpmsg_dev_t* dev, uint16_t ept_addr);
// 轮询模式主动 poll（返回 0=还有消息）
int esp_amp_rpmsg_poll(esp_amp_rpmsg_dev_t* dev);

// 发送（零拷贝：create_message + send_nocopy 必须成对）
void* esp_amp_rpmsg_create_message(esp_amp_rpmsg_dev_t* dev, uint32_t nbytes, uint16_t flags);
int   esp_amp_rpmsg_send_nocopy(esp_amp_rpmsg_dev_t* dev, esp_amp_rpmsg_ept_t* ept, uint16_t dst_addr, void* data, uint16_t data_len);
// 拷贝：send 单独使用，不可与 create_message 混用
int   esp_amp_rpmsg_send(esp_amp_rpmsg_dev_t* dev, esp_amp_rpmsg_ept_t* ept, uint16_t dst_addr, void* data, uint16_t data_len);
// 接收端用完必须 destroy（仅接收端调用）
int   esp_amp_rpmsg_destroy(esp_amp_rpmsg_dev_t* dev, void* msg_data);
// 查询单条最大数据长度
uint16_t esp_amp_rpmsg_get_max_size(esp_amp_rpmsg_dev_t* dev);
```
> notify/poll 互补：一端 poll=false 则对端 notify=true。中断模式下回调在 ISR 上下文，仅 ISR-safe API。

## RPC（esp_amp_rpc.h）

```c
// 错误码
#define ESP_AMP_RPC_OK 0 / ESP_AMP_RPC_FAIL -1
#define ESP_AMP_RPC_ERR_INVALID_ARG -2 / _INVALID_SIZE -3 / _NO_MEM -4 / _EXIST -5 / _NOT_FOUND -6 / _TIMEOUT -7 / _INVALID_STATE -8

// 命令状态
#define ESP_AMP_RPC_STATUS_OK 0x0000 / _SERVER_BUSY 0x0001 / _INVALID_CMD 0x0002 / _EXEC_FAILED 0x0003 / _PENDING 0x0004

typedef void *esp_amp_rpc_server_t;
typedef void *esp_amp_rpc_client_t;
typedef struct esp_amp_rpc_cmd_t esp_amp_rpc_cmd_t;
typedef void (*esp_amp_rpc_app_cb_t)(esp_amp_rpc_client_t, esp_amp_rpc_cmd_t *, void *);
typedef void (*esp_amp_rpc_cmd_handler_t)(esp_amp_rpc_cmd_t *cmd);   // server 侧

struct esp_amp_rpc_cmd_t {
    uint16_t cmd_id; uint16_t status;
    uint16_t req_len; uint16_t resp_len;     // resp_len 是 resp_data 缓冲容量
    uint8_t *req_data; uint8_t *resp_data;
    esp_amp_rpc_app_cb_t cb; void *cb_arg;
};

typedef struct { uint16_t msg_id, cmd_id, status, msg_len; uint8_t msg_data[0]; } esp_amp_rpc_pkt_t;
typedef struct { uint16_t cmd_id; esp_amp_rpc_cmd_handler_t handler; } esp_amp_rpc_service_t;

// 存储类型（用户静态分配）
typedef uint8_t esp_amp_rpc_client_stg_t[sizeof(esp_amp_rpc_client_inst_t)];
typedef uint8_t esp_amp_rpc_server_stg_t[sizeof(esp_amp_rpc_server_inst_t)];

typedef struct {
    uint16_t client_id, server_id; esp_amp_rpmsg_dev_t *rpmsg_dev; esp_amp_rpc_client_stg_t *stg;
    esp_amp_rpc_app_poll_cb_t poll_cb; void *poll_arg;
} esp_amp_rpc_client_cfg_t;

typedef struct {
    uint16_t server_id; uint8_t queue_len, srv_tbl_len; esp_amp_rpmsg_dev_t *rpmsg_dev;
    esp_amp_rpc_server_stg_t *stg; uint16_t req_buf_len, resp_buf_len;
    uint8_t *req_buf, *resp_buf, *srv_tbl_stg;
} esp_amp_rpc_server_cfg_t;

// client
esp_amp_rpc_client_t esp_amp_rpc_client_init(esp_amp_rpc_client_cfg_t *cfg);
void  esp_amp_rpc_client_deinit(esp_amp_rpc_client_t client);
int   esp_amp_rpc_client_execute_cmd(esp_amp_rpc_client_t client, esp_amp_rpc_cmd_t *cmd);  // 非阻塞
int   esp_amp_rpc_client_abort_cmd(esp_amp_rpc_client_t client, esp_amp_rpc_cmd_t *cmd);    // 超时必须调用
void  esp_amp_rpc_client_poll(esp_amp_rpc_client_t client);

// server
esp_amp_rpc_server_t esp_amp_rpc_server_init(esp_amp_rpc_server_cfg_t *cfg);
void  esp_amp_rpc_server_deinit(esp_amp_rpc_server_t server);
int   esp_amp_rpc_server_add_service(esp_amp_rpc_server_t server, uint16_t cmd_id, esp_amp_rpc_cmd_handler_t handler);
int   esp_amp_rpc_server_del_service(esp_amp_rpc_server_t server, uint16_t cmd_id);
#if !IS_ENV_BM
int   esp_amp_rpc_server_run(esp_amp_rpc_server_t server, uint32_t timeout_ms);   // FreeRTOS 在任务上下文处理排队命令
#endif
```
> 同一时刻仅 1 个 pending 请求。client 非线程安全。超时必须 `abort_cmd` 防止迟到响应改写栈上 cmd。

## System / 生命周期（esp_amp_system.h，仅 maincore 可见大部分）

```c
#if IS_MAIN_CORE
esp_err_t esp_amp_load_sub_from_partition(const esp_partition_t* sub_partition);
esp_err_t esp_amp_load_sub(const void* sub_bin);
int       esp_amp_start_subcore(void);          // 0=成功, -1=失败
void      esp_amp_stop_subcore(void);
int       esp_amp_system_panic_init(void);
bool      esp_amp_subcore_panic(void);
void      esp_amp_subcore_panic_handler_default(void);   // 弱函数，可覆盖
#endif
int esp_amp_system_init(void);
```

## Port 层（esp_amp_platform.h）

```c
int      esp_amp_platform_get_core_id(void);     // 读 mhartid CSR
void     esp_amp_platform_delay_ms(uint32_t time);
void     esp_amp_platform_delay_us(uint32_t time);
int64_t  esp_amp_platform_get_time_ms(void);     // subcore 用 mcycle，maincore 用 esp_system_get_time
void     esp_amp_platform_intr_enable(void);     // 全局中断
void     esp_amp_platform_intr_disable(void);
void     esp_amp_platform_sw_intr_enable(void);  // 仅软件中断
void     esp_amp_platform_sw_intr_disable(void);
int      esp_amp_platform_sw_intr_install(void);
void     esp_amp_platform_sw_intr_trigger(void); // 对端核
void     esp_amp_platform_sw_intr_clear(void);
void     esp_amp_platform_memory_barrier(void);
```

## Env 层（esp_amp_env.h）

```c
void esp_amp_env_enter_critical(void);   // 支持嵌套，配对调用
void esp_amp_env_exit_critical(void);
int  esp_amp_env_in_isr(void);           // 1=ISR, 0=非 ISR
int  esp_amp_env_queue_create(void **queue, uint32_t queue_len, uint32_t item_size);
int  esp_amp_env_queue_send(void *queue, void *data, uint32_t timeout_ms);
int  esp_amp_env_queue_recv(void *queue, void *data, uint32_t timeout_ms);
void esp_amp_env_queue_delete(void *queue);
```

## Light Sleep 宏（subcore 侧）

```c
// 来自 docs/light_sleep.md（system PM 组件）
#define ESP_AMP_PM_SKIP_LIGHT_SLEEP_ENTER() esp_amp_system_pm_subcore_skip_light_sleep()  // 增计数，必要时触发 sw_intr 唤醒 maincore
#define ESP_AMP_PM_SKIP_LIGHT_SLEEP_EXIT()  esp_amp_system_pm_subcore_resume_light_sleep() // 减计数，归零后允许 maincore 再睡
```
> 仅 LP subcore + `CONFIG_ESP_AMP_SYSTEM_AUTO_LIGHT_SLEEP_SUPPORT_ENABLE=y`。访问 HP RAM/外设前后成对调用。
