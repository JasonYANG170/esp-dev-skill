# 添加用户自定义 AT 指令

> **适用摘要**: 在不修改 esp-at 仓库源码的前提下，通过 `at_custom_cmd` 组件添加用户自定义 AT 指令，包括四种指令类型（Test/Query/Set/Execute）、参数解析、结果输出、可选参数、阻塞执行与接收端口原始数据。

## 触发意图

- "添加 AT 指令"
- "自定义 AT 命令"
- "add user-defined AT command"
- "AT+MYCMD"
- "esp_at_cmd_t"
- "解析 AT 指令参数"

## 前置条件

| 条件 | 要求 |
|---|---|
| 工程 | 已克隆 esp-at 并能编译（见 `recipes/build_and_flash.md`） |
| 组件 | 把 `examples/at_custom_cmd/` 复制到工程外作为自定义组件 |
| 环境变量 | `AT_CUSTOM_COMPONENTS` 指向组件目录 |

## 分步说明

### 指令命名与类型规则

- 命令名以 `+` 开头（如 `+TEST`，省略 `AT` 前缀）。
- 允许字符：字母 `A-Za-z`、数字 `0-9`、以及 `! % - . / : _`。
- 每条指令最多四种类型，不用的回调置 `NULL`：
  - Test Command：`AT+CMD=?`
  - Query Command：`AT+CMD?`
  - Set Command：`AT+CMD=<params>`
  - Execute Command：`AT+CMD`

### 步骤 1：定义指令（四种类型示例）

来自 `examples/at_custom_cmd/custom/at_custom_cmd.c`：

```c
#include <stdio.h>
#include <string.h>
#include <stdbool.h>
#include "esp_at.h"

// Test Command: AT+TEST=?
static uint8_t at_test_cmd_test(uint8_t *cmd_name)
{
    uint8_t buffer[64] = {0};
    snprintf((char *)buffer, 64, "test command: <AT%s=?> is executed\r\n", cmd_name);
    esp_at_port_write_data(buffer, strlen((char *)buffer));
    return ESP_AT_RESULT_CODE_OK;
}

// Query Command: AT+TEST?
static uint8_t at_query_cmd_test(uint8_t *cmd_name)
{
    uint8_t buffer[64] = {0};
    snprintf((char *)buffer, 64, "query command: <AT%s?> is executed\r\n", cmd_name);
    esp_at_port_write_data(buffer, strlen((char *)buffer));
    return ESP_AT_RESULT_CODE_OK;
}

// Set Command: AT+TEST=<digit>,"<str>"
static uint8_t at_setup_cmd_test(uint8_t para_num)
{
    uint8_t index = 0;
    int32_t digit = 0;
    if (esp_at_get_para_as_digit(index++, &digit) != ESP_AT_PARA_PARSE_RET_OK) {
        return ESP_AT_RESULT_CODE_ERROR;
    }
    uint8_t *str = NULL;
    if (esp_at_get_para_as_str(index++, &str) != ESP_AT_PARA_PARSE_RET_OK) {
        return ESP_AT_RESULT_CODE_ERROR;
    }
    uint8_t *buffer = (uint8_t *)malloc(512);
    if (!buffer) {
        return ESP_AT_RESULT_CODE_ERROR;
    }
    int len = snprintf((char *)buffer, 512, "setup command: <AT%s=%d,\"%s\"> is executed\r\n",
                       esp_at_get_current_cmd_name(), digit, str);
    esp_at_port_write_data(buffer, len);
    free(buffer);
    return ESP_AT_RESULT_CODE_OK;
}

// Execute Command: AT+TEST
static uint8_t at_exe_cmd_test(uint8_t *cmd_name)
{
    uint8_t buffer[64] = {0};
    snprintf((char *)buffer, 64, "execute command: <AT%s> is executed\r\n", cmd_name);
    esp_at_port_write_data(buffer, strlen((char *)buffer));
    return ESP_AT_RESULT_CODE_OK;
}

static const esp_at_cmd_t at_custom_cmd[] = {
    {"+TEST", at_test_cmd_test, at_query_cmd_test, at_setup_cmd_test, at_exe_cmd_test},
};
```

### 步骤 2：注册并用宏初始化

```c
bool esp_at_custom_cmd_register(void)
{
    return esp_at_custom_cmd_array_register(at_custom_cmd,
                                            sizeof(at_custom_cmd) / sizeof(esp_at_cmd_t));
}

// 用 ESP_AT_CMD_SET_INIT_FN 强制放入 .at_cmd_set_init_fn 段，
// 由 esp_at_init() 自动执行。第二参数为优先级，越大越晚执行。
ESP_AT_CMD_SET_INIT_FN(esp_at_custom_cmd_register, 1);
```

> 若在 `examples/at_custom_cmd` 目录内新增，避免把函数命名为 `esp_at_custom_cmd_register`（已被示例占用），改用 `esp_at_custom_cmd_register_foo`，对应改宏参数与链接选项。

### 步骤 3：CMakeLists.txt

```cmake
file(GLOB_RECURSE srcs *.c)
set(includes "include")
# 用到 lwip 等额外组件时追加
set(require_components at freertos nvs_flash)

idf_component_register(
    SRCS ${srcs}
    INCLUDE_DIRS ${includes}
    REQUIRES ${require_components})

idf_component_set_property(${COMPONENT_NAME} WHOLE_ARCHIVE TRUE)

# 链接选项确保运行时能找到注册函数（按实际函数名改）
target_link_libraries(${COMPONENT_LIB} INTERFACE "-u esp_at_custom_cmd_register")
```

### 步骤 4：设环境变量并编译

```bash
export AT_CUSTOM_COMPONENTS=/abs/path/to/at_custom_cmd
./build.py build
./build.py -p /dev/ttyUSB0 flash
```

### 步骤 5：验证

```
AT+TEST=?
test command: <AT+TEST=?> is executed
OK

AT+TEST=1,"espressif"
setup command: <AT+TEST=1,"espressif"> is executed
OK
```

## 进阶：复杂指令

### 可选参数（中间/末尾）

解析返回值有三态，必须显式判断 `_OMITTED`（空串 `""` 不算省略）：

```c
esp_at_para_parse_ret_t r = esp_at_get_para_as_digit(idx++, &v);
if (r == ESP_AT_PARA_PARSE_RET_FAIL) return ESP_AT_RESULT_CODE_ERROR;
if (r == ESP_AT_PARA_PARSE_RET_OMITTED) { /* 用默认值 */ }
```

判断末尾参数是否省略：比较 `num_index == para_num`。

### 阻塞指令（信号量同步）

```c
xSemaphoreHandle at_operation_sema = NULL;

uint8_t at_exe_cmd_test(uint8_t *cmd_name)
{
    // ... 输出 ...
    at_operation_sema = xSemaphoreCreateBinary();
    assert(at_operation_sema != NULL);
    // 阻塞等待其他任务 xSemaphoreGive 释放
    xSemaphoreTake(at_operation_sema, portMAX_DELAY);
    return ESP_AT_RESULT_CODE_OK;
}
```

### 接收 AT 端口原始数据（指定长度）

用 `esp_at_port_enter_specific` 注册接收回调，配合 `esp_at_port_read_data` 读数据：

```c
static xSemaphoreHandle at_sync_sema = NULL;

void wait_data_callback(void) { xSemaphoreGive(at_sync_sema); }

uint8_t at_setup_cmd_test(uint8_t para_num)
{
    int32_t specified_len = 0;
    if (esp_at_get_para_as_digit(0, &specified_len) != ESP_AT_PARA_PARSE_RET_OK) {
        return ESP_AT_RESULT_CODE_ERROR;
    }
    uint8_t *buf = malloc(specified_len);

    if (!at_sync_sema) {
        at_sync_sema = xSemaphoreCreateBinary();
        assert(at_sync_sema != NULL);
    }
    // 输出输入提示符
    esp_at_port_write_data((uint8_t *)">", strlen(">"));
    // 注册接收回调
    esp_at_port_enter_specific(wait_data_callback);

    int32_t received_len = 0;
    while (xSemaphoreTake(at_sync_sema, portMAX_DELAY)) {
        received_len += esp_at_port_read_data(buf + received_len, specified_len - received_len);
        if (specified_len == received_len) {
            esp_at_port_exit_specific();
            int32_t remain = esp_at_port_get_data_length();
            if (remain > 0) {
                esp_at_port_recv_data_notify(remain, portMAX_DELAY);
            }
            // 处理 buf ...
            break;
        }
    }
    free(buf);
    return ESP_AT_RESULT_CODE_OK;
}
```

不指定长度（类似网络透传）时，循环读直到收到 `+++` 退出。

### 输出 SEND OK

用 `esp_at_write_result` 单独输出 SEND OK（不改端口状态），再用返回值输出命令结果：

```c
uint8_t at_exe_cmd_test(uint8_t *cmd_name)
{
    // ... 发送数据到服务器/MCU ...
    esp_at_write_result(ESP_AT_RESULT_CODE_SEND_OK);   // 输出 SEND OK
    return ESP_AT_RESULT_CODE_OK;                        // 输出 OK
}
```

## 常见错误

| 错误 | 原因 | 解决方法 |
|---|---|---|
| 指令完全不响应 | 未加 `ESP_AT_CMD_SET_INIT_FN` | 加宏，并确保 `WHOLE_ARCHIVE TRUE` |
| 指令不响应（已加宏） | `AT_CUSTOM_COMPONENTS` 未设或路径错 | 导出正确绝对路径后重新 build |
| 指令不响应（链接丢弃） | 缺链接选项 `-u` | CMakeLists 加 `target_link_libraries(... INTERFACE "-u esp_at_custom_cmd_register")` |
| 编译报 `_RESULT_OK` 未定义 | 用了旧枚举名 | 改用 `ESP_AT_PARA_PARSE_RET_OK`（`_RET_`） |
| 可选参数被当错误 | 未判断 `_OMITTED` | 显式判断三态 |
| 指令后端口卡住 | 用 `write_result` 后未恢复端口 | 改用 `esp_at_dispatch_result(ESP_AT_RESULT_CODE_OK, NULL)` |
| 在 handler 内调 `esp_at_exe_cmd` | 死锁 | 通过独立任务+队列派发 |

## 参考

- 示例组件：`examples/at_custom_cmd/`（`custom/at_custom_cmd.c`、`include/at_custom_cmd.h`、`CMakeLists.txt`）
- 仓库文档：`docs/en/Compile_and_Develop/How_to_add_user-defined_AT_commands.rst`
- 头文件：`components/at/include/esp_at_core.h`、`esp_at_cmd_register.h`、`esp_at.h`
