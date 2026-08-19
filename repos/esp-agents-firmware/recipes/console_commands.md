# 串口 Console 命令

> **适用摘要**: 使用 `agent_console` 组件注册与管理串口命令，复用默认命令（set-token/set-agent/set-wifi/cpu-dump/mem-dump/reboot/reset-to-factory 等）。

> Version: selected repo/component version; align with the user project and dependency manifest.
> Evidence: `repos/esp-agents-firmware/resources/`, source/examples in `repos/esp-agents-firmware/`, and this recipe path `repos/esp-agents-firmware/recipes/console_commands.md`.
> Validation: draft metadata added from repository routing; verify APIs, Kconfig symbols, and component dependencies against the selected version.

## 触发意图

- "串口命令"
- "注册 console command"
- "set-token / set-agent / set-wifi"
- "agent_console"
- "添加自定义命令"

## 前置条件

| 条件 | 要求 |
|---|---|
| 组件 | `components/agent_console`（`agent_console.h`）、`components/setup`（`setup/console.h`） |
| 参考源码 | `components/setup/src/setup_console.c`、`components/agent_console/src/agent_console.c` |
| 已 init | `app_main` 中 `agent_console_init()` 已执行 |

## 分步说明

### 1. 初始化 console

`app_main` 中（来自 `examples/voice_chat/main/app_main.c`）：

```c
#include <agent_console.h>

agent_console_init();   /* 初始化 console REPL */
```

`components/agent_console/include/agent_console.h` 提供：

```c
esp_err_t agent_console_init(void);
esp_err_t agent_console_register_default_commands(void);   /* cpu-dump/mem-dump/reboot/reset-to-factory 等 */
esp_err_t agent_console_register_command(const esp_console_cmd_t *cmd);
```

### 2. 内置 setup 命令

`components/setup/src/setup_console.c` 经 `setup_console_register_commands()`（`setup/console.h`）注册：

| 命令 | 用法 | 实现函数 |
|---|---|---|
| `set-token` | `set-token <refresh_token>` | `console_refresh_token_cmd_handler` → `agent_setup_set_refresh_token()` |
| `set-agent` | `set-agent <agent_id>` | `console_agent_id_cmd_handler` → `agent_setup_set_agent_id()` |

`components/setup/src/agent_setup.c` 经 `register_set_wifi_cli_handler()` 注册：

| 命令 | 用法 | 行为 |
|---|---|---|
| `set-wifi` | `set-wifi <ssid> <password>` | `esp_wifi_set_config` + `esp_wifi_start` + `esp_wifi_connect` |

> 这些命令都通过 `agent_console_register_command()` 挂到统一 console。

### 3. agent_console 默认命令

按 `components/agent_console/README.md`，`agent_console_register_default_commands()` 提供：`cpu-dump`、`mem-dump`、`reboot`、`reset-to-factory` 等。

### 4. 注册自定义命令

`esp_console_cmd_t` 是 ESP-IDF 标准结构。仿 `setup_console.c` 模式：

```c
#include <agent_console.h>
#include <esp_console.h>

static int my_cmd_handler(int argc, char **argv)
{
    if (argc != 2) {
        printf("Usage: my-cmd <arg>\n");
        return ESP_ERR_INVALID_ARG;
    }
    printf("got: %s\n", argv[1]);
    return ESP_OK;
}

esp_err_t my_console_register(void)
{
    const esp_console_cmd_t cmd = {
        .command = "my-cmd",
        .help = "Do something\nUsage: my-cmd <arg>",
        .func = my_cmd_handler,
    };
    return agent_console_register_command(&cmd);
}
```

在 `app_main` 的 `agent_console_init()` 之后调用 `my_console_register()`。

### 5. 调用顺序

```c
void app_main(void)
{
    esp_event_loop_create_default();
    nvs_flash_init();
    agent_console_init();                 // 1. 初始化 REPL
    /* setup 内部会注册 set-token/set-agent/set-wifi（由 app_agent_start → agent_setup_start 触发） */
    /* 自定义命令在这里注册：my_console_register(); */
    esp_board_manager_init();
    /* ... 其余初始化 ... */
    app_agent_start();                   // 触发 setup → 注册 wifi 命令 + 配网
}
```

## 常见错误

| 错误 | 原因 | 解决方法 |
|---|---|---|
| 命令不出现 | 未在 `agent_console_init` 之后注册 | 确认注册顺序 |
| `set-wifi` 找不到 | setup 未启动 | 确保 `app_agent_start()` 已执行 |
| 命令参数解析错 | handler 内未校验 argc | 仿 setup_console.c 校验 argc 并打印 usage |
| 中文/特殊字符乱码 | 终端编码 | 用 UTF-8 终端，115200 波特率 |

## 参考

- `components/agent_console/include/agent_console.h` — console API
- `components/agent_console/README.md` — 默认命令说明
- `components/setup/include/setup/console.h` — `setup_console_register_commands`
- `components/setup/src/setup_console.c` — `set-token`/`set-agent` 实现
- `components/setup/src/agent_setup.c` — `set-wifi` 实现
- `recipes/device_setup_provisioning.md` — 配网命令使用场景
