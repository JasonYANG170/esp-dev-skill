# 自定义与覆盖功能码处理器

> **适用摘要**: 用 `mbc_set_handler` / `mbc_get_handler` / `mbc_delete_handler` / `mbc_get_handler_count` 注册新的厂商自定义功能码（如 0x41），或覆盖标准功能码（如 0x04 读输入寄存器）。覆盖主站和从站两种用法，并附 FC 0x41 回显示例。

> Version: selected repo/component version; align with the user project and dependency manifest.
> Validation: draft metadata added from repository routing; verify APIs, Kconfig symbols, and component dependencies against the selected version.

## 触发意图

- "自定义 Modbus 功能码"
- "厂商私有功能码 0x41"
- "覆盖读输入寄存器处理"
- "mbc_set_handler 用法"
- "Modbus 私有命令"

## 前置条件

| 条件 | 要求 |
|---|---|
| 参考示例 | 主站 `examples/serial/mb_serial_master/main/serial_master.c`；从站 `examples/serial/mb_serial_slave/main/serial_slave.c`（TCP 同名） |
| 头文件 | `mbcontroller.h`（提供 `mb_fn_handler_fp`、`mb_exception_t`） |
| 已创建对象 | `mbc_*_create_*` 成功；handler 注册应在 `start` 之前 |

## 分步说明

### 1. 处理器函数签名（来自 `mb_types.h`）

```c
typedef mb_exception_t (*mb_fn_handler_fp)(void *pinst, uint8_t *frame_ptr, uint16_t *len_buf);
```

`frame_ptr` 指向 **从功能码开始** 的 ADU 帧；`len_buf` 是入参也是出参（返回响应长度）。返回 `mb_exception_t`（`MB_EX_NONE` = 成功）。

### 2. 主站：注册 FC 0x41 厂商命令（来自 `serial_master.c`）

```c
#define MB_CUST_DATA_LEN 100
static char my_custom_data[MB_CUST_DATA_LEN] = {0};

mb_exception_t my_custom_handler(void *inst, uint8_t *frame_ptr, uint16_t *len)
{
    MB_RETURN_ON_FALSE((frame_ptr && len && *len && *len < (MB_CUST_DATA_LEN - 1)),
                       MB_EX_ILLEGAL_DATA_VALUE, TAG, "incorrect custom frame buffer");
    strncpy((char *)&my_custom_data[0], (char *)&frame_ptr[1], MB_CUST_DATA_LEN);
    return MB_EX_NONE;
}

// 在 master_init 中，create 之后、start 之前：
const uint8_t custom_command = 0x41;
esp_err_t err = mbc_delete_handler(master_handle, custom_command); // 先删（可能不存在，返回 INVALID_STATE 可忽略）
MB_RETURN_ON_FALSE((err == ESP_OK || err == ESP_ERR_INVALID_STATE), ESP_ERR_INVALID_STATE, TAG, "delete handler fail");
err = mbc_set_handler(master_handle, custom_command, my_custom_handler);
MB_RETURN_ON_FALSE((err == ESP_OK), ESP_ERR_INVALID_STATE, TAG, "set handler fail");
mb_fn_handler_fp handler = NULL;
err = mbc_get_handler(master_handle, custom_command, &handler);
MB_RETURN_ON_FALSE((err == ESP_OK && handler == my_custom_handler), ESP_ERR_INVALID_STATE, TAG, "verify handler fail");
```

### 3. 主站：发送自定义请求（`mbc_master_send_request`）

```c
char *pcustom_string = "Master";   // 要发送的字节
mb_param_request_t req = {
    .slave_addr = MB_DEVICE_ADDR1,
    .command = 0x41,
    .reg_start = 0,
    .reg_size = (strlen(pcustom_string) >> 1)   // 寄存器数（必须是偶数字节）
};
mbc_master_send_request(master_handle, &req, pcustom_string);
// 从站应答到达后，my_custom_handler 被调用，结果落到 my_custom_data[]
```

### 4. 从站：注册 FC 0x41 回声处理器（来自 `serial_slave.c`）

```c
#define MB_CUST_DATA_MAX_LEN 100

mb_exception_t my_custom_fc_handler(void *inst, uint8_t *frame_ptr, uint16_t *len)
{
    char *str_append = ":Slave";
    MB_RETURN_ON_FALSE((frame_ptr && len && *len < (MB_CUST_DATA_MAX_LEN - strlen(str_append))),
                       MB_EX_ILLEGAL_DATA_VALUE, TAG, "incorrect custom frame");
    frame_ptr[*len] = '\0';
    strcat((char *)&frame_ptr[1], str_append);   // 在原数据后追加 ":Slave"
    *len = (strlen(str_append) + *len);          // 更新响应长度
    return MB_EX_NONE;
}

// app_main 中，create 之后、start 之前：
const uint8_t custom_command = 0x41;
esp_err_t err = mbc_delete_handler(mbc_slave_handle, custom_command);
MB_RETURN_ON_FALSE((err == ESP_OK || err == ESP_ERR_INVALID_STATE), ;, TAG, "delete fail");
err = mbc_set_handler(mbc_slave_handle, custom_command, my_custom_fc_handler);
MB_RETURN_ON_FALSE((err == ESP_OK), ;, TAG, "set fail");
```

### 5. 覆盖标准功能码（保留原处理逻辑）

```c
// 覆盖 FC 0x04（读输入寄存器），先拿到标准处理器再在自定义里调用
mb_fn_handler_fp pstandard_handler = NULL;
mbc_get_handler(master_handle, 0x04, &pstandard_handler);

mb_exception_t my_fc04(void *pinst, uint8_t *frame_ptr, uint16_t *plen) {
    mb_exception_t exception = MB_EX_CRITICAL;
    // ……自定义预处理……
    if (pstandard_handler) {
        exception = pstandard_handler(pinst, frame_ptr, plen);
    }
    return exception;
}
mbc_set_handler(master_handle, 0x04, my_fc04);
```

### 6. 查询已注册处理器数量

```c
uint16_t count = 0;
mbc_get_handler_count(master_handle, &count);   // 上限 CONFIG_FMB_FUNC_HANDLERS_MAX（默认 16）
```

## 处理器约束

- 处理器在 **Modbus 控制器事件任务** 中执行，必须 **短小、非阻塞**。
- 从站处理器若耗时超过主站配置的 `CONFIG_FMB_MASTER_TIMEOUT_MS_RESPOND`，从站会丢弃挂起响应并记日志：`handling time [ms]: NNNN, exceeds slave response time in master.`。
- 处理器内不要做重度日志（`ESP_LOG_BUFFER_HEXDUMP` 大缓冲）、不要 `vTaskDelay`、不要长时间持有锁。

## 常见错误

| 错误 | 原因 | 解决方法 |
|---|---|---|
| `mbc_set_handler` 返回 `ESP_ERR_INVALID_STATE` | 表已满（超过 `CONFIG_FMB_FUNC_HANDLERS_MAX`） | 调大该 Kconfig，或先 `mbc_delete_handler` 释放槽位 |
| 自定义命令无应答 | 从站未注册对应功能码 | 从站侧也要 `mbc_set_handler` 注册相同 FC |
| 主站收到 `ESP_ERR_INVALID_RESPONSE` | 处理器返回了非 `MB_EX_NONE` 的异常 | 检查 `len` / 缓冲长度校验，必要时返回 `MB_EX_NONE` |
| 从站偶发不响应 | 处理器执行过久导致竞态 | 精简处理器；调大主站超时 |
| 覆盖后标准功能失效 | 没有调用原始 `pstandard_handler` | 在自定义处理器末尾调用拿到的标准处理器 |
| 发送数据被截断 | `reg_size` 不是偶数（必须整寄存器） | `req.reg_size = (byte_len >> 1)`，按偶数字节发送 |

## 参考

- `espressif-repos/esp-modbus/examples/serial/mb_serial_master/main/serial_master.c`
- `espressif-repos/esp-modbus/examples/serial/mb_serial_slave/main/serial_slave.c`
- `espressif-repos/esp-modbus/docs/en/master_api_overview.rst`（Master Customize Function Handlers）
- `espressif-repos/esp-modbus/docs/en/slave_api_overview.rst`（Slave Customize Function Handlers）
- `espressif-repos/esp-modbus/modbus/mb_controller/common/include/esp_modbus_common.h`
