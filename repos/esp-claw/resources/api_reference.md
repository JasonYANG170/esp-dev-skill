# ESP-Claw API 速查

> 全部函数名 / 工具 id / 结构体 / 模块名均来自 `espressif/esp-claw` 真实源码与文档。未在仓库中找到的 API 一律不列。

## 1. 核心 C 结构体与 API（`components/claw_modules/`）

### claw_cap — 能力注册与分发（`claw_modules/claw_cap/include/claw_cap.h`）

```c
typedef enum {
    CLAW_CAP_KIND_CALLABLE,      // 可被 LLM/Console/自动化调用
    CLAW_CAP_KIND_EVENT_SOURCE,  // 事件源（IM 网关/传感器）
    CLAW_CAP_KIND_HYBRID,        // 既发事件又响应（如 cap_mcp_server）
} claw_cap_kind_t;

#define CLAW_CAP_FLAG_CALLABLE_BY_LLM   // 标记对 LLM 可见

typedef struct {
    const char *id;                 // 全局唯一工具 id
    const char *name;
    const char *family;             // 逻辑族标签（"im"/"files"/"system"...）
    const char *description;        // 模型可见摘要
    claw_cap_kind_t kind;
    uint32_t cap_flags;
    const char *input_schema_json;  // JSON Schema
    esp_err_t (*init)(void);
    esp_err_t (*start)(void);
    esp_err_t (*stop)(void);
    esp_err_t (*execute)(const char *input_json,
                         const claw_cap_call_context_t *ctx,
                         char *output, size_t output_size);
} claw_cap_descriptor_t;

typedef struct {
    const char *group_id;
    const char *plugin_name;        // 可选
    const char *version;            // 可选
    void *plugin_ctx;               // 可选
    const claw_cap_descriptor_t *descriptors;
    size_t descriptor_count;
    esp_err_t (*group_init)(void);
    esp_err_t (*group_start)(void);
    esp_err_t (*group_stop)(void);
} claw_cap_group_t;

// 调用上下文（execute 第二参）
typedef struct {
    const char *session_id;
    const char *chat_id;
    const char *source_channel;
    const char *caller;
    // ...其它路由字段
} claw_cap_call_context_t;
```

常用 API：
```c
esp_err_t claw_cap_init(void);
esp_err_t claw_cap_register_group(const claw_cap_group_t *group);
bool      claw_cap_group_exists(const char *group_id);
esp_err_t claw_cap_set_llm_visible_groups(const char *groups_csv); // 启动期播种
esp_err_t claw_cap_set_session_llm_visible_groups(...);            // 按 session 调整
esp_err_t claw_cap_start_all(void);
esp_err_t claw_cap_call_from_core(const char *cap_id, const char *input_json,
                                  const claw_cap_call_context_t *ctx,
                                  char *output, size_t output_size);
```

### claw_event_router — 事件调度（`claw_modules/claw_event_router/`）

```c
typedef struct {
    char source_cap[...];
    char source_channel[...];
    char event_type[...];
    char event_key[...];
    char chat_id[...];
    char sender[...];
    char *text;            // 或 JSON payload
    char content_type[...];
    char session_policy[...];
    int64_t timestamp_ms;
    // ...
} claw_event_t;

esp_err_t claw_event_router_init(...);
esp_err_t claw_event_router_start(void);
esp_err_t claw_event_router_publish(claw_event_t *event);
esp_err_t claw_event_router_publish_message(const char *source_cap,
                                            const char *source_channel,
                                            const char *chat_id,
                                            const char *text,
                                            const char *sender_id,
                                            const char *message_id);
esp_err_t claw_event_router_register_outbound_binding(const char *channel, ...); // qq/feishu/...
esp_err_t claw_event_router_cancel_event(int event_id);
esp_err_t claw_event_router_purge_queue(const char *event_type_filter,
                                        const char *source_cap_filter,
                                        int *out_cancelled);
```

### claw_core — Agent 运行时（`claw_modules/claw_core/include/claw_core.h`）

```c
typedef struct {
    char *session_id;
    char *user_text;
    char *source_channel, *source_cap, *chat_id, *message_id;
    uint32_t flags;   // CLAW_CORE_REQUEST_FLAG_PUBLISH_OUT_MESSAGE
                      // CLAW_CORE_REQUEST_FLAG_SKIP_RESPONSE_QUEUE
                      // CLAW_CORE_REQUEST_FLAG_USER_INTERRUPT
    // ...
} claw_core_request_t;

typedef struct {
    char *text;            // 助手最终文本
    char *error_message;
} claw_core_response_t;

typedef enum {
    CLAW_CORE_CONTEXT_KIND_SYSTEM_PROMPT,
    CLAW_CORE_CONTEXT_KIND_MESSAGES,
    CLAW_CORE_CONTEXT_KIND_TOOLS,
} claw_core_context_kind_t;

// claw_core_config_t 关键字段：
//   backend_type / model / base_url / auth_type / max_tokens_field
//   timeout_ms / max_tokens / system_prompt（必填非空）
//   supports_tools / supports_vision / image_remote_url_only / image_max_bytes
//   request_gate / task_stack_size / task_priority / task_core
//   request_queue_len / response_queue_len / max_context_providers

esp_err_t claw_core_init(const claw_core_config_t *config);
esp_err_t claw_core_start(void);
esp_err_t claw_core_submit(claw_core_request_t *req);
esp_err_t claw_core_receive(claw_core_response_t *resp);     // / _receive_for
esp_err_t claw_core_cancel_request(uint32_t request_id);     // 0 = 任意当前请求
esp_err_t claw_core_add_completion_observer(...);            // claw_core_completion_summary_t
```
> context provider（edge_agent 注册顺序）：`claw_memory_profile_provider` → `claw_memory_long_term_provider`(full) / `_lightweight_provider` → `claw_memory_session_history_provider` → `claw_skill_skills_list_provider` → `claw_cap_tools_provider`。

### claw_memory / claw_skill / claw_paths / cap_lua（C 助手）

```c
// claw_paths
esp_err_t claw_paths_set(claw_path_kind_t kind, const char *base_path); // CLAW_PATH_DATA / CLAW_PATH_SYSTEM
char     *claw_paths_join(claw_path_kind_t kind, ...);

// cap_lua（固件内同步/异步跑脚本，注册模块）
esp_err_t cap_lua_run_script(const char *path, const char *args_json,
                             uint32_t timeout_ms, char *output, size_t output_size);
esp_err_t cap_lua_run_script_async(const char *path, const char *args_json,
                                   uint32_t timeout_ms, const char *name,
                                   const char *exclusive, bool replace,
                                   char *output, size_t output_size);
esp_err_t cap_lua_stop_job(const char *name_or_job_id, uint32_t timeout_ms, char *output, size_t output_size);
esp_err_t cap_lua_stop_all_jobs(const char *exclusive, uint32_t timeout_ms, char *output, size_t output_size);
esp_err_t cap_lua_register_module(const char *name, lua_CFunction luaopen);  // 必须在 register_group 前
esp_err_t cap_lua_register_group(const char *base_dir);   // 之后锁定注册
esp_err_t cap_lua_set_base_dir(const char *dir);

// cap_scheduler_init 见 recipes/scheduled_task.md
```

### claw_memory — 记忆 C API（`claw_modules/claw_memory/include/claw_memory.h`）

```c
typedef struct {
    const char *session_root_dir;          // edge_agent 默认 /fatfs/sessions
    const char *memory_root_dir;           // 默认 /fatfs/memory
    size_t      max_message_chars;         // 默认 4096
    uint32_t    max_tool_iterations;       // 与 claw_core 对齐
    claw_memory_llm_config_t llm;          // 抽取用 LLM 配置
    bool        enable_async_extract_stage_note;  // full 默认 true
} claw_memory_config_t;

typedef struct {
    char     id[40];         // memory_id（store 时由系统生成）
    char     source[16];     // manual / auto_llm
    char     content[256];   // 归一化事实陈述
    uint16_t summary_ids[3]; // 内部 summary 标签 id
    uint8_t  summary_id_count;
    char     tags[96];       // 逗号分隔，建议 1-3 个稳定主题
    char     keywords[128];  // 关键词索引
    uint32_t created_at;
    uint32_t updated_at;
    uint16_t access_count;
    uint8_t  deleted;
} claw_memory_item_t;

typedef struct {
    const char *const *summary_labels;   // 必须来自注入目录的精确值
    size_t summary_label_count;
    size_t limit;
} claw_memory_query_t;

esp_err_t claw_memory_init(const claw_memory_config_t *config);
esp_err_t claw_memory_register_group(void);                 // 注册 claw_memory 能力组（full 模式）
esp_err_t claw_memory_store(const claw_memory_item_t *item);
esp_err_t claw_memory_store_with_result(claw_memory_item_t *item, bool *out_changed);
esp_err_t claw_memory_recall(const claw_memory_query_t *query, char **out_json);  // *out_json 需 free
esp_err_t claw_memory_update(const claw_memory_item_t *item);
esp_err_t claw_memory_update_with_result(claw_memory_item_t *item, bool *out_changed);
esp_err_t claw_memory_forget(const char *memory_id);
esp_err_t claw_memory_forget_with_result(const char *memory_id, claw_memory_item_t *out_item, bool *out_changed);
esp_err_t claw_memory_list(char **out_json);                // *out_json 需 free

// Context Providers（在 claw_core_init 后注册）
extern const claw_core_context_provider_t claw_memory_profile_provider;              // 注入 user/soul/identity.md
extern const claw_core_context_provider_t claw_memory_long_term_provider;            // full：注入 summary-tag 目录
extern const claw_core_context_provider_t claw_memory_long_term_lightweight_provider; // lightweight：注入 MEMORY.md 文本
extern const claw_core_context_provider_t claw_memory_session_history_provider;      // 注入当前 session 历史
```
> 工具层 schema 见 `s_memory_descriptors[]`（`claw_memory_cap.c`）：`memory_store` 必填 `content`（可选 `memory_id`/`tags`/`keywords`）；`memory_recall` 必填 `summary_labels:string[]`（精确白名单值）；`memory_update`/`memory_forget` 必填 `memory_id`。

### cap_mcp_server — MCP 服务端 C API（`cap_mcp_server/include/cap_mcp_server.h`）

```c
// HYBRID 能力，工具数=0，对外暴露设备工具；SSE + mDNS
typedef struct {
    const char *name;                 // MCP 工具名，如 "lua.run_script"
    const char *description;
    esp_mcp_value_t (*callback)(const esp_mcp_property_list_t *properties);
    const char *property_names[6];    // 字符串属性名（最多 6）
    size_t      property_count;
} cap_mcp_server_tool_def_t;

esp_err_t cap_mcp_server_init(void);   // 幂等；create MCP engine
esp_err_t cap_mcp_server_add_tool(const cap_mcp_server_tool_def_t *tool_defs, uint16_t tool_count); // init 后、start 前
esp_err_t cap_mcp_server_start(void);  // 起 HTTP SSE + mDNS 广播；幂等
esp_err_t cap_mcp_server_stop(void);
esp_err_t cap_mcp_server_deinit(void);
```
> `mcp_server_point` 自带的 6 个工具（`cap_mcp_lua.c`）：`lua.run_script` / `lua.run_script_async` / `lua.list_async_jobs` / `lua.get_async_job` / `lua.stop_async_job` / `lua.stop_all_async_jobs`，直接代理 `cap_lua_*`。

### cap_im_platform — IM 平台 C 配置 API（`cap_im_platform/include/cap_im_*.h`）

```c
// 每平台凭据 setter；在 group start 前调用
esp_err_t cap_im_tg_set_token(const char *bot_token);
esp_err_t cap_im_feishu_set_credentials(const char *app_id, const char *app_secret);
esp_err_t cap_im_qq_set_credentials(const char *app_id, const char *app_secret);
void      cap_im_qq_set_msg_type(int msg_type);                  // QQ 消息类型
esp_err_t cap_im_wechat_set_client_config(const cap_im_wechat_client_config_t *config);  // 微信走扫码登录

typedef struct {
    const char *storage_root_dir;          // 默认 /fatfs/inbox
    size_t      max_inbound_file_bytes;    // 默认 2 * 1024 * 1024
    bool        enable_inbound_attachments;
} cap_im_tg_attachment_config_t;   // feishu/qq/wechat 同名前缀同结构

esp_err_t cap_im_tg_set_attachment_config(const cap_im_tg_attachment_config_t *config);
esp_err_t cap_im_feishu_set_attachment_config(const cap_im_feishu_attachment_config_t *config);
esp_err_t cap_im_qq_set_attachment_config(const cap_im_qq_attachment_config_t *config);
esp_err_t cap_im_wechat_set_attachment_config(const cap_im_wechat_attachment_config_t *config);
```
> Telegram 后端：`CAP_IM_TG_DEDUP_CACHE_SIZE=64` 的 FNV-1a 64 位哈希环去重；20 s 长轮询；附件异步队列 + `getFile` 流式下载；文本自动分块；文件 multipart 流式上传（用 `stat()` 算 `Content-Length`，`esp_http_client_open` 流式）。`chat_id` 缺省时 feishu/qq/tg 的 send 工具回退到 `ctx->chat_id`；wechat 不回退。

## 2. 能力工具 id（LLM/Console 可调，真实）

| 能力组 | 工具 id | 备注 |
|---|---|---|
| `cap_files` | `read_file` / `write_file` / `edit_file` / `delete_file` / `copy_file` / `move_file` / `list_dir` | 受管根默认 `/fatfs`；禁 `..`；读上限 32KB；`edit_file` 仅替换首处；`write_file` 自动建父目录 |
| `cap_lua` | `lua_run_script` / `lua_run_script_async` / `lua_list_async_jobs` / `lua_get_async_job` / `lua_stop_async_job` / `lua_stop_all_async_jobs` | 受管根默认 `/fatfs/scripts`；async 支持 `name`/`exclusive`/`replace` |
| `cap_scheduler` | `scheduler_list` / `scheduler_get` / `scheduler_add` / `scheduler_update` / `scheduler_remove` / `scheduler_enable` / `scheduler_disable` / `scheduler_pause` / `scheduler_resume` / `scheduler_trigger_now` / `scheduler_reload` | `*_json` 入参为转义字符串 |
| `cap_router_mgr` | `list_router_rules` / `get_router_rule` / `add_router_rule` / `update_router_rule` / `delete_router_rule` / `reload_router_rules` | |
| `cap_skill` | `list_skill`（不默认对 LLM 可见）/ `register_skill` / `unregister_skill` / `activate_skill` | `activate_skill` 返回 `<skill_content>` 并打开 `cap_groups` |
| `cap_system` | `get_system_info` / `get_current_time` / `restart_device` | `sections` 可选：`chip/uptime/memory/cpu/wifi/ip` |
| `cap_mcp_client` | `mcp_list_tools` / `mcp_call_tool` / `mcp_discover` | 默认不对 LLM 可见 |
| `cap_mcp_server` | （无 LLM 工具，HYBRID，对外暴露设备能力为 MCP 工具） | mDNS + SSE |
| `cap_im_tg` | `tg_gateway` / `tg_send_message` / `tg_send_image` / `tg_send_file` | 长文本分块；文件 multipart 流式上传 |
| `cap_im_qq` | `qq_gateway` / `qq_send_message` / `qq_send_image` / `qq_send_file` | 目标 `c2c:<openid>` / `group:<group_openid>` |
| `cap_im_feishu` | `feishu_gateway` / `feishu_send_message` / `feishu_send_image` / `feishu_send_file` | 目标 `chat_id` 或 `ou_` 开头 `open_id` |
| `cap_im_wechat` | `wechat_gateway` / `wechat_send_message` / `wechat_send_image` | 不支持 `send_file` |
| `cap_im_local` | `local_gateway` / `local_send_message` | Web 内置 IM |
| `cap_http_request` | （受 `search_http_allowlist` 守护的出站 HTTP） | |
| `cap_web_search` | （Brave / Tavily 等） | key 非空才启用 |
| `cap_llm_inspect` | （嵌套 LLM 推理 / 图像检查） | `claw_core_llm_infer_*` |
| `claw_memory`（full） | `memory_store` / `memory_recall` / `memory_list` / `memory_update` / `memory_forget` | 仅 full 模式 |

## 3. Lua 模块 API（`require("name")`，真实签名）

### gpio（`lua_driver_gpio`）
```lua
gpio.set_direction(pin, mode)  -- mode: "input"|"output"|"input_output"|"output_od"|"input_output_od"|"disable"
gpio.set_level(pin, level)     -- 非0即高
gpio.get_level(pin) -> int
```

### delay（`lua_module_delay`）
```lua
delay.delay_ms(ms)  -- ms 整数；负值钳 0
delay.delay_us(us)  -- 忙等，0..1000000；更长用 delay_ms
```

### storage（`lua_module_storage`）
```lua
storage.get_root_dir() -> string           -- 可写根（/fatfs 或 SD 挂载点）
storage.join_path(...) -> string           -- 拼接，去重分隔符；首段决定绝对/相对
storage.exists(path) -> bool
storage.stat(path) -> info|nil,err
storage.mkdir(path)
storage.write_file(path, content)
storage.read_file(path) -> string
storage.listdir(path)
storage.remove(path)
storage.rename(old, new)
storage.get_free_space() -> {total, free, used}
```

### json（`lua_module_json`）
```lua
json.encode(value) -> string   -- 数组(连续正整数键) vs 对象；不支持值/深嵌套报错
json.decode(text) -> value     -- null -> nil
```

### event_publisher（`lua_module_event_publisher`）
```lua
ep.publish_message(text_or_table)
-- 字符串形式：runtime 设 source_cap="lua_script"，从 args 回填 channel/chat_id
-- table 形式必填 source_cap 与 text；channel/chat_id 缺则从 args 回填
ep.publish_trigger(table)
ep.publish(table)
-- 仅点号调用；勿冒号
```

### capability（`lua_module_call_capability`）
```lua
local ok, out, err = capability.call(cap_id, payload_table, opts)
-- opts: { source_cap=..., max_output_bytes=... }
```

### arg_schema（`lua_module_system` 纯 Lua）
```lua
local arg_schema = require("arg_schema")
local schema = {
  io   = arg_schema.int({default=38, min=0}),
  flag = arg_schema.bool({default=true}),
  obj  = arg_schema.object({default={...}, fields={...}}),
}
local ctx = arg_schema.parse(args, schema)
```

### led_strip（`lua_module_led_strip`）—— 来自 light_switch 真实用法
```lua
local strip, err = led_strip.new(io, led_count)  -- 失败返回 nil, err
strip:set_pixel(index, r, g, b)
strip:refresh()
strip:clear()
strip:close()
```

> 其它模块（`display` / `lcd` / `lvgl` / `audio` / `camera` / `imu` / `i2c` / `adc` / `mcpwm` / `pcnt` / `uart` / `touch` / `button` / `knob` / `ir` / `http_server` / `thread` / `system` / `image` / `dht` / `environmental_sensor` / `magnetometer` / `lib_fuel_gauge` / `sci` / `lcd_touch` / `ble` / `ble_hid` / `motion_detect`）的 API 见各组件 `README.md` 与 `lib/*.md`（构建期同步进 `/system/scripts/docs` 与 `builtin_lua_modules` Skill）。

## 4. Console 命令（真实）

```sh
help
session [<id>]            # 切换/查看；session new 建新会话
ask <text>                # 多轮；ask_once <text> 单轮不写历史
cap list | groups | enable|disable|unload <group> | load <group>   # cap load qq 动态注册
cap call <id> '<json>'    # 例：cap call list_dir '{"keyword":"scripts"}'
auto reload|rules|rule <id>|add_rule '<json>'|update_rule '<json>'|delete_rule <id>
auto emit_message <source_cap> <channel> <chat_id> <text...>
auto emit_trigger <source_cap> <event_type> <event_key> '<payload_json>'
auto last
event_router --rules|--rule <id>|--add-rule-json '<json>'|--update-rule-json '<json>'|--delete-rule <id>|--reload
scheduler --list|--reload|--add --json '<json>'|--enable|--disable|--pause|--resume|--trigger <id>
time / web_search / mcp_client / mcp_server / llm_inspect   # 其它已注册命令
# skill 命令默认未注册；用 cap call activate_skill '{"skill_id":"..."}' 替代
```

## 参考来源

- `components/claw_modules/claw_cap/include/claw_cap.h`
- `components/claw_modules/claw_core/include/claw_core.h`
- `components/claw_modules/claw_event_router/include/claw_event.h`
- `components/claw_capabilities/cap_files/include/cap_files.h`、`cap_lua/include/cap_lua.h`、`cap_scheduler/include/cap_scheduler.h`
- `components/lua_modules/lua_driver_gpio/README.md`、`lua_module_storage/README.md`、`lua_module_delay/README.md`、`lua_module_json/README.md`、`lua_module_event_publisher/README.md`
- `docs/src/content/docs/en/reference-cap/*.mdx`、`reference-core/*.mdx`
- `application/edge_agent/fatfs_image/storage/skills/light_switch/scripts/led_strip_switch.lua`
