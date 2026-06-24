# 设备作为 MCP 服务端（mcp_server_point 精简应用）

> **适用摘要**: 用仓库内置的 `application/mcp_server_point` 把 ESP32 变成一台 **MCP 服务端**，对外（桌面 IDE / 其它 ESP-Claw 设备）通过 SSE + mDNS 暴露 `lua.run_script` / `lua.run_script_async` / async 作业管理等工具，是唯一不依赖完整 `edge_agent` 栈的 ESP-Claw 运行形态——无 `claw_core`、无 `event_router`、无 IM、无 CLI、无记忆，仅 `cap_lua` + `cap_mcp_server`。

## 触发意图

- "把设备当 MCP 服务端 / 工具提供方"
- "不用完整 Agent，只让外部调 Lua"
- "mcp_server_point 怎么构建"
- "cap_mcp_server 怎么暴露工具 / mDNS"
- "多台 ESP-Claw 互调能力（cap_mcp_client + cap_mcp_server）"

## 前置条件

| 条件 | 要求 |
|---|---|
| 应用目录 | `application/mcp_server_point/`（独立应用，非 `edge_agent`） |
| Kconfig 默认 | `CONFIG_APP_CLAW_CAP_LUA=y`、`CONFIG_APP_CLAW_CAP_MCP_SERVER=y`；`CORE / EVENT_ROUTER / MEMORY / FILES / SCHEDULER / IM / SKILL_MGR / SESSION_MGR / ROUTER_MGR / SYSTEM / WEB_SEARCH / LLM_INSPECT / MCP_CLIENT` 全部 `n`；`CONFIG_APP_CLAW_ENABLE_CLI=n` |
| 启用限制 | `app_config.c` 把 `app_claw_config_t.enabled_cap_groups` 写死为 `"cap_lua"`，`cap_mcp_server` 不走 cap 分组，而是在 `main.c` 里手动 `init/start` |
| 网络要求 | STA 必须连上 Wi-Fi（mDNS + SSE 需要）；无 Web 配置门户，SSID/密码从 menuconfig 或 NVS 取 |
| 参考文档 | `docs/src/content/docs/en/reference-cap/cap-mcp.mdx`、`reference-project/boot-and-runtime.mdx` |
| 参考示例 | `application/mcp_server_point/`（含 `main/main.c`、`components/mcp_server_point_tools/cap_mcp_lua.c`） |

## 分步说明

### 1. 应用结构与 edge_agent 的差异

mcp_server_point 是仓库内的「精简第二应用」：没有事件路由器、没有 agent core、没有 IM、没有 CLI，只剩 `cap_lua` + `cap_mcp_server`。能力组在 `components/app_config/app_config.c` 中硬编码：

```c
// application/mcp_server_point/components/app_config/app_config.c
#define APP_MCP_SERVER_POINT_ENABLED_CAP_GROUPS "cap_lua"
// ...
strlcpy(out->enabled_cap_groups, APP_MCP_SERVER_POINT_ENABLED_CAP_GROUPS,
        sizeof(out->enabled_cap_groups));
```

`cap_mcp_server` **不在** `enabled_cap_groups` 里——它由 `main.c` 在 `app_claw_start` 之后手动起。

### 2. 启动顺序（main.c 真实流程）

```c
// application/mcp_server_point/main/main.c::app_main 关键节选
ESP_ERROR_CHECK(init_nvs());
ESP_ERROR_CHECK(app_config_init());
ESP_ERROR_CHECK(app_config_load(s_config));
app_config_to_claw(s_config, s_claw_config);
ESP_ERROR_CHECK(init_fatfs());                       // /fatfs on partition "storage"
ESP_ERROR_CHECK(wifi_manager_init());
wifi_manager_start(&(wifi_manager_config_t){
    .sta_ssid     = s_config->wifi_ssid,
    .sta_password = s_config->wifi_password,
});

ESP_ERROR_CHECK(claw_paths_set(CLAW_PATH_DATA,   "/fatfs"));
ESP_ERROR_CHECK(claw_paths_set(CLAW_PATH_SYSTEM, "/fatfs"));  // 本应用两路径同根
ESP_ERROR_CHECK(app_claw_start(s_claw_config));   // 仅装 cap_lua

// cap_mcp_server 三段式：init -> 注册工具 -> start
ESP_ERROR_CHECK(cap_mcp_server_init());
ESP_ERROR_CHECK(cap_mcp_lua_tools_init());        // 把 6 个 lua.* MCP 工具注册进 server
ESP_ERROR_CHECK(cap_mcp_server_start());          // 起 HTTP SSE + mDNS 广播
```

> 顺序约束（来自 `cap_mcp_lua.h`）：`cap_mcp_lua_tools_init()` 必须在 `cap_mcp_server_init()` 之后、`cap_mcp_server_start()` 之前调用；且此时 `cap_lua` 能力组必须已经由 `app_claw_start` 注册并启动。

### 3. cap_mcp_server 的对��契约

`cap_mcp_server` 是 `CLAW_CAP_KIND_HYBRID` 能力，**无 LLM 可调工具**（工具数 = 0），作用是把「设备自己注册的工具」对外暴露成 MCP 工具，供局域网内的 MCP 客户端发现并调用：

- 传输：MCP SSE（Server-Sent Events）
- 发现：mDNS 广播（`CONFIG_MDNS_*` 在 `sdkconfig.defaults` 已启用 PSRAM 分配）
- 生命周期：`init → add_tool → start → stop → deinit`，全部幂等

工具定义类型（`cap_mcp_server.h`，真实结构体）：

```c
typedef struct {
    const char *name;                 // MCP 工具名，如 "lua.run_script"
    const char *description;          // 给 MCP 客户端看的说明
    esp_mcp_value_t (*callback)(const esp_mcp_property_list_t *properties);
    const char *property_names[6];    // 接受的字符串属性名（最多 6）
    size_t property_count;
} cap_mcp_server_tool_def_t;

esp_err_t cap_mcp_server_init(void);
esp_err_t cap_mcp_server_add_tool(const cap_mcp_server_tool_def_t *tool_defs, uint16_t tool_count);
esp_err_t cap_mcp_server_start(void);
esp_err_t cap_mcp_server_stop(void);
esp_err_t cap_mcp_server_deinit(void);
```

### 4. mcp_server_point 自带的 6 个 Lua MCP 工具

`components/mcp_server_point_tools/cap_mcp_lua.c` 注册的工具（真实 `s_tool_defs[]`）：

| MCP 工具名 | 作用 | properties |
|---|---|---|
| `lua.run_script` | 同步跑 Lua，返回输出字符串 | `path` `args` `timeout_ms` |
| `lua.run_script_async` | 异步跑 Lua，返回 job id | `path` `args` `timeout_ms` `name` `exclusive` `replace` |
| `lua.list_async_jobs` | 按 status 列出异步作业 | `status` |
| `lua.get_async_job` | 按 job_id 或 name 取状态/摘要 | `job_id` `name` |
| `lua.stop_async_job` | 协作式停一个作业（默认等待 2000 ms） | `job_id` `name` `wait_ms` |
| `lua.stop_all_async_jobs` | 停全部（可按 exclusive 过滤） | `exclusive` `wait_ms` |

`path` 既可是 `scripts/` 下的相对路径（会被拼成 `/fatfs/scripts/<path>`），也可是绝对 `.lua` 路径；包含 `..` 或不以 `.lua` 结尾一律拒绝（`cap_mcp_server_lua_resolve_run_path` 沙箱校验）。脚本必须**事先**通过 `fatfs_image/scripts/` 打包进 `storage` 分区。

### 5. 构建与烧录（注意是 `gen-bmgr-config`，不是 `bmgr`）

```bash
cd application/mcp_server_point
. $IDF_PATH/export.sh

# 选板：本应用用 gen-bmgr-config（与 edge_agent 的 idf.py bmgr 不同）
idf.py gen-bmgr-config -c ./boards -b espressif/esp_Ditto

idf.py build
idf.py -p PORT flash monitor
```

- 固件产物：`build/mcp_server_point.bin`
- `storage.bin`（FAT 镜像）由 `fatfs_create_spiflash_image` 在 `FLASH_IN_PROJECT` 启用时随固件一并烧入；`scripts/` 与 `scripts/builtin/`（构建期从 `lua_module_builder` 同步）在镜像内
- 默认分区表 `partitions_16MB.csv`，由 `tools/cmake/flash_partition_defaults.cmake` 按板子 flash 大小自动选

> `components/gen_bmgr_codes/CMakeLists.txt` 指向**具体**板子目录；换板后必须重跑 `gen-bmgr-config` **并**对齐该组件里的路径，否则构建/硬件不匹配。

### 6. 让外部 MCP 客户端发现并调用

设备起来后，在同一局域网：
- MCP 客户端用 mDNS 发现 `esp-claw` 主机（`CONFIG_LWIP_LOCAL_HOSTNAME="esp-claw"`）
- 调 `lua.run_script`：`{"path":"hello.lua","args":{}}`
- 调 `lua.run_script_async`：`{"path":"blink.lua","name":"blink","exclusive":"display"}` → 拿到 job id，之后用 `lua.get_async_job` 轮询、`lua.stop_async_job` 取消

### 7. （可选）多设备分布式 Agent 网络

当 `cap_mcp_client`（`CONFIG_APP_CLAW_CAP_MCP_CLIENT`）与 `cap_mcp_server` 同时启用时（注意 mcp_server_point 默认关了 client，需自行打开），多台 ESP-Claw 可通过 MCP 互相发现、互调对方能力，形成分布式 Agent 网络。client 侧工具：`mcp_list_tools` / `mcp_call_tool` / `mcp_discover`。

## 常见错误

| 错误 | 原因 | 解决方法 |
|---|---|---|
| 启动后外部 MCP 客户端发现不到设备 | STA 没连上 Wi-Fi（无 Web 门户可兜底） | menuconfig 里设对 SSID/密码，或写入 NVS；`wifi_manager_wait_connected` 失败会落 AP 但不会触发 mDNS |
| `cap_mcp_lua_tools_init` 报错 | 在 `cap_mcp_server_init` 之前或 `cap_mcp_server_start` 之后调用 | 严格按 `init → tools_init → start` 顺序，且 `app_claw_start` 已装好 `cap_lua` |
| `lua.run_script` 返回 invalid path | path 含 `..` 或非 `.lua` 后缀，或脚本不存在 | path 用相对名（自动拼 `/fatfs/scripts/`）或绝对 `/fatfs/.../*.lua`；脚本要预先打进 `fatfs_image/scripts/` |
| 改 `APP_CLAW_CAP_MEMORY=n` 还想配 memory mode | 记忆模式只在 `CAP_MEMORY=y` 时有意义 | 不设 `APP_CLAW_MEMORY_MODE_*`；本应用不跑记忆 |
| 换板后编译报板子目录错 | `gen_bmgr_codes` 组件路径没跟着改 | 换板后重跑 `gen-bmgr-config` 并对齐 `components/gen_bmgr_codes` 的路径 |
| 误把 `cap_mcp_server` 写进 `enabled_cap_groups` | `app_config.c` 把分组写死成 `cap_lua` | `cap_mcp_server` 走 `main.c` 手动起，不进 cap 分组 |
| 期望 MCP server 有 LLM 工具 | `cap_mcp_server` 工具数 = 0，是 HYBRID | 它只对外暴露设备工具；要调 LLM 用 `cap_mcp_client` 或跑 `edge_agent` |

## 参考项目

- `application/mcp_server_point/README.md` — 应用结构与启动流程
- `application/mcp_server_point/main/main.c` — `app_main` 真实装配（NVS→FATFS→Wi-Fi→`app_claw_start`→`cap_mcp_server` 三段式）
- `application/mcp_server_point/components/app_config/app_config.c` — `enabled_cap_groups` 写死 `cap_lua`
- `application/mcp_server_point/components/mcp_server_point_tools/cap_mcp_lua.c` — 6 个 `lua.*` MCP 工具的回调实现与 `s_tool_defs[]`
- `application/mcp_server_point/components/mcp_server_point_tools/cap_mcp_lua.h` — `cap_mcp_lua_tools_init()` 契约
- `application/mcp_server_point/sdkconfig.defaults` — lean Kconfig 默认（关掉 CORE/EVENT_ROUTER/MEMORY/IM…）
- `components/claw_capabilities/cap_mcp_server/include/cap_mcp_server.h` — `cap_mcp_server_tool_def_t` / `init/add_tool/start/stop/deinit`
- `components/claw_capabilities/cap_mcp_client/` — client 侧（`mcp_list_tools`/`mcp_call_tool`/`mcp_discover`）
- `docs/src/content/docs/en/reference-cap/cap-mcp.mdx` — 官方 cap_mcp 文档（SSE/mDNS/HYBRID）
- `docs/src/content/docs/en/reference-project/boot-and-runtime.mdx` — edge_agent 启动流程对照
