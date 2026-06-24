# AGENTS.md — Supplementary Agent Guide

> 核心规则、recipe 索引、陷阱、执行流程都在 `SKILL.md`。
> 本文件仅补充 `SKILL.md` 未涉及的约定与工具链指引，**不重复内容**。

## Project Context

**Language**: C (ESP-IDF) · **Target**: ESP32-S3 系列开发板（ESP-BOX-3 / ESP-VoCat v1.2 / M5Stack CoreS3 / +H2 Gateway） · **Toolchain**: ESP-IDF v5.5.2+（须在 `release/5.5` 分支）+ idf.py

**云后端**: ESP Private Agents 平台（默认 `api.agents.espressif.com`，可经 `CONFIG_ESP_AGENT_API_ENDPOINT` 改为自建 AWS 部署）；配网/设备管理经 ESP RainMaker。

## Code Generation Conventions

### 文件命名 / 目录

- 组件源码：`components/<name>/{src,include,idf_component.yml,Kconfig*}`
- 示例：`examples/<name>/main/{app_main.c, app_tools.c, app_tools.h, CMakeLists.txt, idf_component.yml}`
- 公共应用封装：`examples/common/app_common/{src,include}`（`app_agent` / `app_audio` / `app_device` / `app_display` / `app_common_tools` / `app_touch_press` / `app_capacitive_touch`）
- 板级定义：`examples/common/boards/<board>/{board_defs.h, sdkconfig.defaults, README.md}`
- Agent 工具配置（**不入固件，用于 Dashboard 建模**）：`examples/<name>/agent_config.json`

### Include Pattern

```c
/* 标准示例 app_main.c 的包含顺序（来自仓库 voice_chat/main/app_main.c） */
#include <esp_log.h>
#include <nvs_flash.h>
#include <esp_check.h>

#include <agent_console.h>          /* components/agent_console */

#include <agent_setup.h>            /* components/setup */
#include <setup/rainmaker.h>
#include <setup/console.h>

#include <esp_board_manager.h>      /* ESP GMF board manager */

#include <app_agent.h>              /* examples/common/app_common */
#include <app_device.h>
#include <app_display.h>
#include <board_defs.h>             /* 当前选中板的宏 */

#include <app_audio.h>
#include "app_tools.h"              /* 示例自有工具注册 */
```

底层 Agent API（当直接用裸 API 而非 app 封装时）：

```c
#include <esp_agent.h>              /* 聚合头：core + events + tools + messages */
```

### 标准示例工程结构

```
MyAgentProject/                    # 由 examples/voice_chat 或 matter_controller 拷贝而来
├── main/
│   ├── app_main.c                 # app_main() 初始化与启动
│   ├── app_tools.c                # 本地工具 handler 实现 + app_tools_register()
│   ├── app_tools.h                # esp_err_t app_tools_register(void);
│   ├── CMakeLists.txt             # idf_component_register(SRC_DIRS "." INCLUDE_DIRS ".")
│   └── idf_component.yml          # idf: version '>=5.5' + 依赖（agent/audio/setup/boards/app_common）
├── agent_config.json              # Dashboard Agent 建模用，不入固件
├── CMakeLists.txt                 # 顶层（cmake_minimum_required + project）
└── sdkconfig.defaults             # 可选：默认 menuconfig 覆盖
```

matter_controller 示例额外有 `main/matter/`（`app_controller.h` 等）与 `components/matter_controller/`（`rmaker_controller_service/`、`controller_rest_apis/`、`op_creds_issuer/`）。

### Canonical app_main 模式（来自仓库源码）

```c
void app_main(void)
{
    ESP_RETURN_VOID_ON_ERROR(esp_event_loop_create_default(), TAG, "Failed to create event loop");
    ESP_RETURN_VOID_ON_ERROR(nvs_flash_init(), TAG, "Failed to initialize NVS");

    ESP_RETURN_VOID_ON_ERROR(agent_console_init(), TAG, "Failed to initialize console");
    ESP_RETURN_VOID_ON_ERROR(esp_board_manager_init(), TAG, "Failed to initialize board manager");
    ESP_RETURN_VOID_ON_ERROR(app_display_init(), TAG, "Failed to initialize display");
    ESP_RETURN_VOID_ON_ERROR(app_audio_init(), TAG, "Failed to initialize audio pipeline");

    app_device_config_t device_config = {
        .set_text_cb = app_text_message_callback,
        .system_state_changed_cb = app_state_changed_callback,
        .priv_data = NULL,
    };
    ESP_RETURN_VOID_ON_ERROR(app_device_init(&device_config), TAG, "Failed to initialize device");

    app_agent_config_t agent_config = { .event_handler = app_event_handler };
    ESP_RETURN_VOID_ON_ERROR(app_agent_init(&agent_config), TAG, "Failed to initialize agent");

    ESP_RETURN_VOID_ON_ERROR(app_tools_register(), TAG, "Failed to register tools");
    ESP_RETURN_VOID_ON_ERROR(app_agent_start(), TAG, "Failed to start agent");

    /* 仅 matter_controller 示例追加： */
    /* ESP_RETURN_VOID_ON_ERROR(matter_controller_start_task(), TAG, "Matter controller start"); */
}
```

事件转发约定：自定义 `app_event_handler` **必须**调用 `app_agent_default_event_handler(arg, event_base, event_id, event_data)`，否则默认状态机/音频联动失效。

### 本地工具 handler 模板（来自 examples/*/main/app_tools.c）

```c
static esp_err_t my_tool_handler(esp_agent_handle_t handle, const char *tool_name,
                                 esp_agent_tool_param_t params[], size_t num_params,
                                 void *user_data, char **result)
{
    /* 1. 遍历 params，按 name + type 取值（仅 NUMBER/STRING/BOOL） */
    /* 2. 业务逻辑 */
    /* 3. *result 必须堆分配（strdup/malloc），框架发完会 free */
    *result = strdup("ok");
    return ESP_OK;
}

esp_err_t app_tools_register(void)
{
    app_agent_register_tool("my_tool", my_tool_handler, NULL);
    return ESP_OK;
}
```

### 日志约定

```c
static const char *TAG = "main";   /* 每个文件一个 TAG */
ESP_LOGI(TAG, "Agent Started");
ESP_RETURN_VOID_ON_ERROR(fn(), TAG, "Failed to ...");
```

## Build Workflow

1. 切到 ESP-IDF `release/5.5`（`idf.py --version` 确认 ≥ 5.5.2）
2. 在示例目录：`idf.py select-board --board <board>`（board ∈ `esp_vocat_board_v1_2` / `esp_box_3` / `m5stack_cores3` / `m5stack_cores3_h2_gateway`）
3. `idf.py build flash monitor`
4. 首次配网：ESP RainMaker Home App 扫码，或串口 `set-wifi`/`set-token`/`set-agent`
5. 调试：串口日志（默认波特率 115200），关注 `app_agent` / `setup` / `esp_agent` TAG

### 组件依赖（idf_component.yml，来自仓库）

- `components/agent` 依赖 `espressif/esp_websocket_client: ^1.6.0`（IDF ≥ 5.0）
- `components/audio` 依赖 `espressif/esp_codec_dev ^1.5`（public）、`espressif/esp-sr ^2.1.5`、`espressif/gmf_ai_audio ^0.7.2`、`espressif/gmf_audio ^0.7.1`、`espressif/esp_audio_simple_player ^0.9`、`espressif/gmf_io ^0.7`
- `examples/common/app_common` 依赖 `espressif2022/esp_emote_expression: 0.0.*` + 本地 `agent`/`audio`/`setup`/`boards` override + `espressif/network_provisioning: *`

## Codegen Checklist

- [ ] `app_main` 初始化顺序与仓库一致（event loop → NVS → console → board_manager → display → audio → device → agent → tools → agent_start）
- [ ] 自定义 `app_event_handler` 已转发给 `app_agent_default_event_handler`
- [ ] 每个本地工具：固件 `app_agent_register_tool(name, ...)` **且** `agent_config.json` 同名声明 `inputSchema`
- [ ] 工具 `*result` 一律 `strdup`/`malloc`（无栈/字面量）
- [ ] 工具参数仅判 `ESP_AGENT_PARAM_TYPE_NUMBER/STRING/BOOL`
- [ ] 选板：`idf.py select-board --board <board>` 且 `board_defs.h` 已含所需宏
- [ ] 自定义部署：`CONFIG_ESP_AGENT_API_ENDPOINT` 已改
- [ ] 音频 Kconfig：`CONFIG_AUDIOUPLOAD/DOWNLOAD_SAMPLE_RATE`、`FRAME_DURATION_MS` 与云端 Agent 一致
- [ ] Matter 示例：`CONFIG_ESP_CLIENT_ONLY_MATTER_CONTROLLER=y` 且互斥项（server/commissioner/commissioning driver/CHIPOBLE）已关；`matter_controller_start_task()` 在 `app_agent_start()` 之后调用
- [ ] 无屏板：已按 `docs/example_customisation.md` 注释 `app_display_*`
- [ ] 默认 agent_id：`CONFIG_AGENT_SETUP_DEFAULT_AGENT_ID` 已设（或运行时 `set-agent`）

## Do Not Modify

- `components/` 下官方组件源码（`agent` / `agent_console` / `audio` / `setup`）— 通过 `idf_component.yml` override 引用，不要就地改
- `examples/common/app_common/` 公共封装 — 新行为通过回调/工具扩展，不直接改公共源
- `examples/common/boards/<existing>/` 现有板定义 — 新板请新建子目录
- `agent_config.json` 是 Dashboard 建模文件，不入固件；改动后需在 ESP Private Agents Dashboard 重建 Agent 并重新下发 agent_id
- `SKILL.md` frontmatter（Skill 元数据）
