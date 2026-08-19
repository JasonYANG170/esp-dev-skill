# 从站事件循环与参数访问通知

> **适用摘要**: 在 Modbus 从站中使用 `mbc_slave_check_event` 阻塞等待主机访问，用 `mbc_slave_get_param_info` 取出被访问寄存器的详细信息（时间戳、偏移、类型、地址、大小），按事件掩码分类处理 Holding/Input/Coil/Discrete 访问。

> Version: selected repo/component version; align with the user project and dependency manifest.
> Evidence: `repos/esp-modbus/resources/`, source/examples in `repos/esp-modbus/`, and this recipe path `repos/esp-modbus/recipes/slave_events.md`.
> Validation: draft metadata added from repository routing; verify APIs, Kconfig symbols, and component dependencies against the selected version.

## 触发意图

- "从站怎么知道主机读了寄存器"
- "mbc_slave_check_event 用法"
- "Modbus 从站事件处理"
- "mbc_slave_get_param_info"
- "从站响应主机写操作"

## 前置条件

| 条件 | 要求 |
|---|---|
| 参考示例 | `examples/serial/mb_serial_slave/main/serial_slave.c`、`examples/tcp/mb_tcp_slave/main/tcp_slave.c` |
| 已注册区域 | 已为相关 `mb_param_type_t` 调用 `mbc_slave_set_descriptor` |
| 已启动 | `mbc_slave_start` 已成功 |
| Kconfig | `CONFIG_FMB_CONTROLLER_NOTIFY_QUEUE_SIZE`（默认 20）控制通知队列长度 |

## 分步说明

### 1. 事件位掩码（来自 `esp_modbus_common.h`）

```c
typedef enum {
    MB_EVENT_NO_EVENTS     = 0x00,
    MB_EVENT_HOLDING_REG_WR = BIT0,   // 写 Holding
    MB_EVENT_HOLDING_REG_RD = BIT1,   // 读 Holding
    MB_EVENT_INPUT_REG_RD   = BIT3,   // 读 Input
    MB_EVENT_COILS_WR       = BIT4,   // 写 Coils
    MB_EVENT_COILS_RD       = BIT5,   // 读 Coils
    MB_EVENT_DISCRETE_RD    = BIT6,   // 读 Discrete
    MB_EVENT_STACK_STARTED  = BIT7,
    MB_EVENT_STACK_CONNECTED= BIT8,
} mb_event_group_t;
```

### 2. 常用掩码组合（来自从站示例）

```c
#define MB_READ_MASK  (MB_EVENT_INPUT_REG_RD | MB_EVENT_HOLDING_REG_RD \
                       | MB_EVENT_DISCRETE_RD | MB_EVENT_COILS_RD)
#define MB_WRITE_MASK (MB_EVENT_HOLDING_REG_WR | MB_EVENT_COILS_WR)
#define MB_READ_WRITE_MASK (MB_READ_MASK | MB_WRITE_MASK)
#define MB_PAR_INFO_GET_TOUT  (10)   // 读通知队列的超时（滴答）
```

### 3. `mb_param_info_t` 字段（来自 `esp_modbus_slave.h`）

```c
typedef struct {
    uint32_t        time_stamp;   // 事件时间戳（微秒）
    uint16_t        mb_offset;    // 被访问的起始 Modbus 寄存器
    mb_event_group_t type;        // 事件类型位
    uint8_t        *address;      // 对应到区域描述符里的存储地址
    size_t          size;         // 被访问的寄存器数
} mb_param_info_t;
```

### 4. 阻塞等待 + 取通知（来自 `serial_slave.c`）

```c
mb_param_info_t reg_info;

for (; holding_reg_params.holding_data0 < MB_CHAN_DATA_MAX_VAL;) {
    // 阻塞，直到主机访问匹配掩码的寄存器类型
    (void)mbc_slave_check_event(mbc_slave_handle, MB_READ_WRITE_MASK);
    ESP_ERROR_CHECK(mbc_slave_get_param_info(mbc_slave_handle, &reg_info, MB_PAR_INFO_GET_TOUT));

    const char *rw_str = (reg_info.type & MB_READ_MASK) ? "READ" : "WRITE";

    if (reg_info.type & (MB_EVENT_HOLDING_REG_WR | MB_EVENT_HOLDING_REG_RD)) {
        ESP_LOGI(TAG, "HOLDING %s (%" PRIu32 " us), ADDR:%u, SIZE:%u",
                 rw_str, reg_info.time_stamp,
                 (unsigned)reg_info.mb_offset, (unsigned)reg_info.size);

        // 主机刚写了 holding_data0，从站据此更新本地业务
        if (reg_info.address == (uint8_t *)&holding_reg_params.holding_data0) {
            (void)mbc_slave_lock(mbc_slave_handle);
            holding_reg_params.holding_data0 += MB_CHAN_DATA_OFFSET;
            (void)mbc_slave_unlock(mbc_slave_handle);
        }
    } else if (reg_info.type & MB_EVENT_INPUT_REG_RD) {
        ESP_LOGI(TAG, "INPUT READ (%" PRIu32 " us), ADDR:%u", reg_info.time_stamp, (unsigned)reg_info.mb_offset);
    } else if (reg_info.type & MB_EVENT_DISCRETE_RD) {
        ESP_LOGI(TAG, "DISCRETE READ ADDR:%u", (unsigned)reg_info.mb_offset);
    } else if (reg_info.type & (MB_EVENT_COILS_RD | MB_EVENT_COILS_WR)) {
        ESP_LOGI(TAG, "COILS %s ADDR:%u", rw_str, (unsigned)reg_info.mb_offset);
    }
}
```

### 5. TCP 从站变体（带停止命令检查）

来自 `tcp_slave.c`：在循环里额外用 `mb_console_event_check` 响应控制台命令，并把 `mbc_slave_get_param_info` 的返回值一并判断（超时不算错误）：

```c
mb_event_group_t bits = mbc_slave_check_event(slave_handle, MB_READ_WRITE_MASK);
esp_err_t err = mbc_slave_get_param_info(slave_handle, &reg_info, MB_PAR_INFO_GET_TOUT);
if ((err != ESP_ERR_TIMEOUT) && (reg_info.type & MB_READ_WRITE_MASK)) {
    // ……分类处理……
}
```

## 常见错误

| 错误 | 原因 | 解决方法 |
|---|---|---|
| `mbc_slave_check_event` 永远不返回 | 该类型区域未注册，主机访问触发异常而非事件 | 用 `mbc_slave_set_descriptor` 注册区域 |
| `mbc_slave_get_param_info` 返回 `ESP_ERR_TIMEOUT` | 在 `MB_PAR_INFO_GET_TOUT` 内没有新通知 | 区分超时与真错误；超时可继续循环 |
| 通知丢失 | 队列溢出（生产快于消费） | 调大 `CONFIG_FMB_CONTROLLER_NOTIFY_QUEUE_SIZE`；加快消费 |
| 主机写后从站读到旧值 | 读取未加锁，与协议栈写并发 | 用 `mbc_slave_lock`/`unlock` 包裹读改写 |
| `type` 位含义误判 | 把读/写位当成单一类型 | 读用 `MB_READ_MASK`，写用 `MB_WRITE_MASK` 分别判断 |

## 参考

- `espressif-repos/esp-modbus/examples/serial/mb_serial_slave/main/serial_slave.c`
- `espressif-repos/esp-modbus/examples/tcp/mb_tcp_slave/main/tcp_slave.c`
- `espressif-repos/esp-modbus/modbus/mb_controller/common/include/esp_modbus_slave.h`
- `espressif-repos/esp-modbus/docs/en/slave_api_overview.rst`（Table 4 `mb_param_info_t`）
- `recipes/serial_slave.md`、`recipes/tcp_slave.md`
