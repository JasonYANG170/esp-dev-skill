# 用 Lua 驱动硬件并发布事件

> **适用摘要**: 在设备上写 Lua 脚本控制 GPIO / 显示 / LED strip，用 `storage` 安全读写文件，用 `event_publisher` 把结果发回 IM/路由器，用 `capability.call` 在脚本里调 `cap_*` 工具。

> Version: selected repo/component version; align with the user project and dependency manifest.
> Evidence: `repos/esp-claw/resources/`, source/examples in `repos/esp-claw/`, and this recipe path `repos/esp-claw/recipes/automation_lua.md`.
> Validation: draft metadata added from repository routing; verify APIs, Kconfig symbols, and component dependencies against the selected version.

## 触发意图

- "写个 Lua 脚本点灯"
- "Lua 控制 GPIO / display"
- "脚本里发消息回 IM"
- "capability.call 调能力"
- "arg_schema 解析参数"

## 前置条件

| 条件 | 要求 |
|---|---|
| cap_lua | `APP_CLAW_CAP_LUA` 启用；受管根默认 `/fatfs/scripts` |
| 模块 | 用到的 `lua_driver_*` / `lua_module_*` 对应 Kconfig 启用 |
| 参考 | `application/edge_agent/fatfs_image/storage/skills/light_switch/scripts/*.lua` |

## 分步说明

### 1. 写脚本（用 cap_files 的 write_file，或 Web 文件管理器）

```json
// LLM 工具调用：把脚本写到受管根
{"path":"scripts/blink.lua","content":"..."}
```
> 用 `write_file` 写到 `scripts/blink.lua`，再用 `lua_run_script {"path":"blink.lua"}` 跑（去掉前导 `scripts/`）。Skill 自带脚本用 `{CUR_SKILL_DIR}/scripts/...`。

### 2. GPIO 点灯（最小例）

```lua
local gpio  = require("gpio")
local delay = require("delay")

gpio.set_direction(2, "output")
for i = 1, 10 do
  gpio.set_level(2, i % 2)
  delay.delay_ms(500)   -- 轮询/动画循环里务必让出 CPU
end
gpio.set_level(2, 0)
```
- `gpio.set_direction(pin, mode)`：mode 为 `"input"` / `"output"` / `"input_output"` / `"output_od"` / `"input_output_od"` / `"disable"`
- `gpio.set_level(pin, level)`：非 0 即高
- `gpio.get_level(pin)` → 整数

### 3. LED strip（WS2812 等）—— 来自 light_switch 真实脚本

```lua
local led_strip = require("led_strip")
local strip = led_strip.new(38, 1)   -- io=38, 1 颗像素；失败返回 nil, err
if not strip then error("led_strip.new failed") end
strip:set_pixel(0, 255, 0, 0)        -- index, r, g, b
strip:refresh()
-- 关闭：strip:clear(); strip:refresh()
strip:close()
```
> `light_switch` Skill 的 `led_strip_switch.lua` 用 `arg_schema` 解析 `io/led_count/enabled/brightness/color`，按亮度缩放 RGB 后逐像素上色；含 `xpcall` + `cleanup()` 释放资源——是写硬件脚本的范本。

### 4. 参数归一化（arg_schema，来自 lua_module_system）

```lua
local arg_schema = require("arg_schema")
local SCHEMA = {
  io        = arg_schema.int({ default = 38, min = 0 }),
  enabled   = arg_schema.bool({ default = true }),
  brightness= arg_schema.int({ default = 255, min = 0, max = 255 }),
}
local ctx = arg_schema.parse(args, SCHEMA)   -- args 是调用方传入的全局
```
> Agent 在 IM 会话里跑脚本时，若工具调用未显式给 `args.channel/chat_id/session_id`，runtime 会自动合并当前会话上下文进 `args`，便于直接回复。

### 5. 安全文件读写（storage，禁止硬编码 /fatfs）

```lua
local storage = require("storage")
local root = storage.get_root_dir()
local dir  = storage.join_path(root, "demo")
storage.mkdir(dir)
storage.write_file(storage.join_path(dir, "test.txt"), "hello")
local text = storage.read_file(storage.join_path(dir, "test.txt"))
-- storage.exists / storage.stat / storage.listdir / storage.remove / storage.rename / storage.get_free_space
```

### 6. 把结果发回 IM / 路由器（event_publisher）

```lua
local ep = require("event_publisher")
-- 最简形式（回调里优先用）：runtime 设 source_cap="lua_script"，并从 args 回填 channel/chat_id
ep.publish_message("按钮被按下！")
-- 需要额外字段时用完整 table（source_cap 与 text 必填）：
ep.publish_message({
  source_cap = "lua_script",
  channel    = args.channel,
  chat_id    = args.chat_id,
  text       = "done",
})
```
> **只能点号调用** `ep.publish_message(...)`；不要用冒号 `ep:publish_message(...)`（会把 self 当首参）。table 形式不能写残缺（缺 `source_cap` 会报错）。

### 7. 在脚本里调 cap_*（capability）

```lua
local capability = require("capability")
local json = require("json")

local ok, out, err = capability.call("scheduler_add", {
  schedule_json = json.encode({ id="hi", kind="interval", interval_ms=60000,
                                 event_type="schedule", event_key="hi" }),
}, { source_cap = "lua_script", max_output_bytes = 8192 })
if not ok then error(tostring(err)) end
```
> `scheduled_task` Skill 的 `add_scheduled_task.lua` 就是范例：`add_router_rule` + `scheduler_add`，失败时 best-effort 回滚 router rule。

### 8. 同步 vs 异步运行

- `lua_run_script`：同步，可选 `timeout_ms`，返回脚本输出字符串——适合快速/状态读取
- `lua_run_script_async`：异步，立即返回 `job_id`，支持 `name` / `exclusive`（互斥组，如 `"display"`）/ `replace:true`（抢占同名/同组任务）——适合动画/等传感器等长任务
  - 状态机：`queued → running → done/failed/timeout/stopped`
  - 轮询：`lua_get_async_job {"name":"..."}`；停止：`lua_stop_async_job {"name":"..."}` / `lua_stop_all_async_jobs {"exclusive":"display"}`

## 常见错误

| 错误 | 原因 | 解决方法 |
|---|---|---|
| `require("xxx")` 失败 | 模块未注册或 Kconfig 关闭 | 在 `app_lua_modules.c` 注册（先于 `register_group`）；启用对应 `APP_CLAW_LUA_*` |
| 脚本超时 / 触发看门狗 | 死循环不让 CPU | 轮询循环里 `delay.delay_ms(...)` 协作式 yield |
| 写文件路径错 | 硬编码 `/fatfs` | 用 `storage.get_root_dir()` + `storage.join_path(...)` |
| `ep:publish_message` 报错 | 用了冒号语法 | 改点号 `ep.publish_message(...)` |
| event publish 缺字段 | table 形式漏 `source_cap` | 用字符串形式，或补全 `source_cap`/`text` |
| `capability.call` 返回 err | 工具入参格式不对 | `scheduler_add` 等的 `_json` 字段是转义字符串；用 `json.encode` |
| `lua_run_script` 路径找不到 | 用了带前导 `scripts/` 的相对路径 | 写时 `scripts/x.lua`，跑时 `{"path":"x.lua"}`；Skill 脚本用 `{CUR_SKILL_DIR}/scripts/...` |

## 参考

- `application/edge_agent/fatfs_image/storage/skills/light_switch/scripts/led_strip_switch.lua`
- `application/edge_agent/fatfs_image/storage/skills/light_switch/scripts/gpio_light_switch.lua`
- `application/edge_agent/fatfs_image/storage/skills/scheduled_task/scripts/add_scheduled_task.lua`
- `components/lua_modules/lua_driver_gpio/README.md`、`lua_module_storage/README.md`、`lua_module_delay/README.md`、`lua_module_json/README.md`、`lua_module_event_publisher/README.md`
- `docs/src/content/docs/en/reference-cap/cap-lua.mdx`、`lua-modules.mdx`
