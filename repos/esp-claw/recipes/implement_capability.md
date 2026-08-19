# 新增一个 Capability 能力组

> **适用摘要**: 从零写一个 `cap_*` 能力组件——定义 descriptor / group、实现 `execute`、在 app 注册、可选附带 Skill，最终成为 LLM/Console/自动化可调用的工具。

> Version: selected repo/component version; align with the user project and dependency manifest.
> Evidence: `repos/esp-claw/resources/`, source/examples in `repos/esp-claw/`, and this recipe path `repos/esp-claw/recipes/implement_capability.md`.
> Validation: draft metadata added from repository routing; verify APIs, Kconfig symbols, and component dependencies against the selected version.

## 触发意图

- "加一个能力"
- "implement a capability"
- "claw_cap_descriptor_t 怎么写"
- "让 LLM 能调用我的功能"
- "cap_my_feature_register_group"

## 前置条件

| 条件 | ��求 |
|---|---|
| 已读 | `docs/.../reference-cap/implement-capability.mdx`、`reference-core/claw-cap.mdx` |
| 头文件 | `components/claw_modules/claw_cap/include/claw_cap.h` |
| 注册点 | `components/common/app_claw/app_capabilities.c` |

## 分步说明

### 1. 脚手架

```
components/claw_capabilities/cap_my_feature/
├── CMakeLists.txt
├── include/cap_my_feature.h
├── src/cap_my_feature.c
└── skills/cap_my_feature/SKILL.md   # 可选但推荐
```

```cmake
# CMakeLists.txt
idf_component_register(
    SRCS "src/cap_my_feature.c"
    INCLUDE_DIRS "include"
    REQUIRES claw_cap cJSON
)
```

### 2. 公共头（只暴露注册）

```c
// include/cap_my_feature.h
#pragma once
#include "esp_err.h"
#ifdef __cplusplus
extern "C" {
#endif
esp_err_t cap_my_feature_register_group(void);
#ifdef __cplusplus
}
#endif
```

### 3. 实现 execute（JSON 入、文本出）

```c
// src/cap_my_feature.c
#include "cap_my_feature.h"
#include "claw_cap.h"
#include "cJSON.h"
#include <string.h>
#include <stdio.h>

static esp_err_t my_feature_execute(const char *input_json,
                                    const claw_cap_call_context_t *ctx,
                                    char *output, size_t output_size)
{
    (void)ctx;
    cJSON *root = cJSON_Parse(input_json ? input_json : "{}");
    if (!root) {
        snprintf(output, output_size, "Error: invalid JSON");
        return ESP_ERR_INVALID_ARG;
    }

    cJSON *param = cJSON_GetObjectItem(root, "param");
    if (!cJSON_IsString(param)) {
        cJSON_Delete(root);
        snprintf(output, output_size, "Error: param is required");
        return ESP_ERR_INVALID_ARG;
    }

    // 真实工作放这里 …
    snprintf(output, output_size, "Done: %s", param->valuestring);

    cJSON_Delete(root);
    return ESP_OK;
}
```

> 规则：成功返回 `ESP_OK`；人/模型可读文本写进 `output`（通常 4–8KB，别溢出）；错误前缀 `"Error: "`；返回前释放临时分配；大载荷分块/落盘/返回路径让上层再读。

### 4. descriptor + group

```c
static const claw_cap_descriptor_t s_my_descriptors[] = {
    {
        .id = "my_action",
        .name = "my_action",
        .family = "custom",
        .description = "Perform my custom action with the given param.",
        .kind = CLAW_CAP_KIND_CALLABLE,
        .cap_flags = CLAW_CAP_FLAG_CALLABLE_BY_LLM,
        .input_schema_json =
            "{\"type\":\"object\","
            "\"properties\":{\"param\":{\"type\":\"string\"}},"
            "\"required\":[\"param\"]}",
        .execute = my_feature_execute,
    },
};

static const claw_cap_group_t s_my_group = {
    .group_id = "cap_my_feature",
    .descriptors = s_my_descriptors,
    .descriptor_count = sizeof(s_my_descriptors) / sizeof(s_my_descriptors[0]),
};
```

可选生命周期钩子（自有后台 task/定时器时）：在 descriptor 上设 `.init` / `.start` / `.stop`。

### 5. 注册（幂等）

```c
esp_err_t cap_my_feature_register_group(void)
{
    if (claw_cap_group_exists(s_my_group.group_id)) return ESP_OK;
    return claw_cap_register_group(&s_my_group);
}
```

### 6. 在 app 注册并设置可见性

```c
// components/common/app_claw/app_capabilities.c
#include "cap_my_feature.h"
// 在 app_capabilities_init() 里：
cap_my_feature_register_group();
// 若需默认对 LLM 可见（否则要走 Skill 激活）：
// claw_cap_set_llm_visible_groups(...) 里加 "cap_my_feature"
```

若作为事件源（IM 网关 / 传感器），用 `CLAW_CAP_KIND_EVENT_SOURCE` 并在后台任务发布：
```c
claw_event_router_publish_message("my_gateway", "my_channel",
                                  chat_id, text, sender_id, message_id);
// 或自定义事件：
claw_event_t event = {0};
strlcpy(event.source_cap, "my_gateway", sizeof(event.source_cap));
strlcpy(event.event_type, "my_custom_event", sizeof(event.event_type));
claw_event_router_publish(&event);
```

### 7. 配套 Skill（可选但推荐）

`skills/cap_my_feature/SKILL.md`：JSON frontmatter 声明 `"cap_groups":["cap_my_feature"]` 与 `"manage_mode":"readonly"`，正文写场景/调用规则/JSON 示例/错误剧本（详见 `recipes/write_skill.md`）。构建期 `skill_builder` 把它同步进 SYSTEM 镜像 `/system/skills/cap_my_feature/`。

### 8. 构建、验证

```bash
idf.py build && idf.py flash monitor
# Console:
cap list                         # 看到 cap_my_feature
cap call my_action '{"param":"hi"}'
```

## 常见错误

| 错误 | 原因 | 解决方法 |
|---|---|---|
| `cap call` 返回 `Error: invalid JSON` | 入参 JSON 不合法 / 引号转义错 | 用单引号包 JSON：`cap call my_action '{"param":"hi"}'` |
| 工具对 LLM 不可见 | 未加入 `llm_visible_groups`，也没 Skill 激活 | 二选一：默认可见加进 allow-list，或写 Skill 用 `activate_skill` 打开 |
| `execute` 输出被截断 | `output_size` 通常 4–8KB | 大载荷分块、落盘、返回路径让上层 `read_file` |
| 后台 task 不退出 | `stop` 钩子没实现/没等 task 退出 | 在 `.stop` 里置停止标志并 join task |
| 重复注册 | 多处调 `register_group` | 用 `claw_cap_group_exists()` 幂等保护 |

## 参考

- `components/claw_modules/claw_cap/include/claw_cap.h`
- `components/claw_capabilities/cap_system/src/cap_system.c`（纯查询型参考）
- `components/claw_capabilities/cap_files/src/cap_files.c`（文件 IO 型参考）
- `components/claw_capabilities/cap_im_platform/src/cap_im_tg.c`（事件源型参考）
- `docs/src/content/docs/en/reference-cap/implement-capability.mdx`
- `docs/src/content/docs/en/reference-core/claw-cap.mdx`
