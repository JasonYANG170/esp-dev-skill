# 注册与实现本地工具（Local Tool）

> **适用摘要**: 通过 `esp_agent_register_local_tool` 注册设备端本地工具，实现 `esp_agent_tool_handler_t` 回调，并同步 Agent 配置。

## 触发意图

- "添加本地工具"
- "register local tool"
- "自定义工具 handler"
- "让 Agent 调用设备功能"
- "工具 result 分配"

## 前置条件

| 条件 | 要求 |
|---|---|
| 参考源码 | `examples/voice_chat/main/app_tools.c`、`examples/common/app_common/src/app_common_tools.c` |
| Agent 已 init | `app_agent_init` 已执行（拿到内部 agent_handle） |
| Agent 配置文件 | `examples/<ex>/agent_config.json` 可编辑 |

## 分步说明

### 1. 理解工具模型

工具是设备端可被云端 Agent 调用的本地函数。流程：
1. **固件侧**：`app_agent_register_tool(name, handler, user_data)`（封装 `esp_agent_register_local_tool`）
2. **配置侧**：在 `agent_config.json` 的 `toolConfiguration.tools[]` 声明同名 `{ "type":"local", "name":"<name>", "inputSchema":{...} }`
3. 运行时 Agent 决定调用 → 框架解析参数 → 调 handler → 把 `*result` 字符串回传服务端

参数仅支持三种类型（来自 `esp_agent_tools.h`）：

| `esp_agent_tool_param_type_t` | JSON Schema 类型 | 取值 |
|---|---|---|
| `ESP_AGENT_PARAM_TYPE_NUMBER` | number | `params[i].value.i`（double） |
| `ESP_AGENT_PARAM_TYPE_STRING` | string | `params[i].value.s`（const char*） |
| `ESP_AGENT_PARAM_TYPE_BOOL` | boolean | `params[i].value.b` |

### 2. 实现 handler（来自 app_tools.c 的真实模式）

```c
#include <string.h>
#include <esp_check.h>
#include <esp_log.h>
#include "app_tools.h"
#include "app_agent.h"

static const char *TAG = "my_tools";

static esp_err_t my_action_handler(esp_agent_handle_t handle, const char *tool_name,
                                   esp_agent_tool_param_t params[], size_t num_params,
                                   void *user_data, char **result)
{
    const char *name = NULL;
    int repeat = 1;
    for (size_t i = 0; i < num_params; i++) {
        if (strcmp(params[i].name, "name") == 0 &&
            params[i].type == ESP_AGENT_PARAM_TYPE_STRING) {
            name = params[i].value.s;
        } else if (strcmp(params[i].name, "repeat") == 0 &&
                   params[i].type == ESP_AGENT_PARAM_TYPE_NUMBER) {
            repeat = (int)params[i].value.i;
        }
    }
    if (!name) {
        *result = strdup("Error: 'name' parameter missing.");
        return ESP_ERR_INVALID_ARG;
    }

    /* 业务逻辑 ... */

    /* ★ result 必须堆分配；框架发完响应后内部 free ★ */
    char *msg = malloc(256);
    if (msg) {
        snprintf(msg, 256, "Done: %s x%d", name, repeat);
    }
    *result = msg ? msg : strdup("ok");
    return ESP_OK;
}

esp_err_t app_tools_register(void)
{
    app_agent_register_tool("my_action", my_action_handler, NULL);
    return ESP_OK;
}
```

### 3. 在 agent_config.json 声明同名工具

`examples/voice_chat/agent_config.json` 片段（仿照内置工具结构）：

```json
{
  "type": "local",
  "name": "my_action",
  "description": "Describe what this tool does so the LLM knows when to call it.",
  "inputSchema": {
    "json": {
      "type": "object",
      "properties": {
        "name":    { "type": "string", "description": "Target name." },
        "repeat":  { "type": "number", "description": "Repeat count." }
      },
      "required": ["name"]
    }
  }
}
```

> `agent_config.json` 不入固件，用于在 ESP Private Agents Dashboard 创建/重建 Agent。改后需重新下发 agent_id 到设备。

### 4. 注销工具

```c
app_agent_tool_unregister("my_action");   /* 封装 esp_agent_unregister_local_tool */
```

### 5. user_data 透传

`app_agent_register_tool(name, handler, user_data)` 的 `user_data` 会原样传给 handler 第 5 参数，用于携带上下文。

### 6. 真实参考：set_emotion（voice_chat）

`examples/voice_chat/main/app_tools.c` 的 `app_tools_set_emotion_handler`：取 `emotion_name`(STRING) → `app_display_is_emotion_valid()` 校验 → `app_display_set_emotion()` → `*result = strdup(...)`。注册时工具名 `"set_emotion"`，对应 `agent_config.json` 同名声明。

## 常见错误

| 错误 | 原因 | 解决方法 |
|---|---|---|
| 工具永远不被调用 | `agent_config.json` 没声明同名 tool | 同步加 tool 声明并重建 Agent |
| 调用后设备崩溃 | `*result` 用了栈/字面量 | 改用 `strdup`/`malloc` |
| 参数取不到 | 类型判断错或 name 不符 | 严格按 `ESP_AGENT_PARAM_TYPE_*` + 完全匹配 name |
| handler 收到未知参数 | LLM 传了额外字段 | 遍历时只处理认识的 name，其余忽略 |
| 重建 Agent 后工具仍不调 | agent_id 没更新到设备 | 串口 `set-agent` 或 App 扫码重下发 |

## 参考

- `components/agent/include/esp_agent_tools.h` — 工具 API 与参数类型
- `examples/common/app_common/include/app_agent.h` — `app_agent_register_tool`
- `examples/voice_chat/main/app_tools.c` — set_emotion handler 范例
- `examples/common/app_common/src/app_common_tools.c` — set_reminder/get_local_time/set_volume 范例
- `examples/voice_chat/agent_config.json` — 工具配置 JSON 结构范例
- `docs/example_customisation.md` — 自定义工具官方说明
- `recipes/builtin_tools.md` — 内置工具用法
