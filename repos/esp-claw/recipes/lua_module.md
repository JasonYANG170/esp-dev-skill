# 新增 Lua 模块（C 绑定 + 文档 + Skill）

> **适用摘要**: 创建一个 `lua_module_*` / `lua_driver_*` 组件，把硬件或服务以 Lua API 暴露给设备脚本，并在 app 里注册（必须在 `cap_lua_register_group` 之前）。

## 触发意图

- "加一个 Lua 模块"
- "lua_module_* 怎么写"
- "让 Lua 脚本能调我的硬件"
- "luaopen_xxx / cap_lua_register_module"
- "lua_CFunction 绑定"

## 前置条件

| 条件 | 要求 |
|---|---|
| 已读 | `docs/.../reference-cap/lua-modules.mdx`、`.agents/spec/lua-module-spec.md` |
| 注册点 | `components/common/app_claw/app_lua_modules.c`（必须在 `cap_lua_register_group` 前） |
| 参考 | `components/lua_modules/lua_module_gpio/`（最小完整样例） |

## 分步说明

### 1. 组件目录（命名必须 `lua_module_xx` / `lua_driver_xx`）

```
components/lua_modules/lua_module_myled/
├── CMakeLists.txt
├── README.md                 # 必需，agent 面向 API 文档（构建期同步进 docs）
├── test/myled_smoke.lua      # 可选，model-readable 参考脚本
├── lib/lib_myled.md + .lua   # 可选，可复用 Lua 库（每个 .lua 必须有同名 .md）
└── src/
    ├── lua_module_myled.c
    └── lua_module_myled.h
```

```cmake
# CMakeLists.txt
idf_component_register(
    SRCS "src/lua_module_myled.c"
    INCLUDE_DIRS "src"
    REQUIRES cap_lua driver
)
```
> 纯 Lua 模块可用空 `idf_component_register()`。

### 2. 实现 Lua C 函数（`int fn(lua_State *L)`）

```c
// src/lua_module_myled.c
#include "lua.h"
#include "lauxlib.h"
#include "cap_lua.h"
#include "driver/gpio.h"

#define MYLED_GPIO GPIO_NUM_2

static int myled_set(lua_State *L)
{
    int on = lua_toboolean(L, 1);
    gpio_set_level(MYLED_GPIO, on ? 1 : 0);
    return 0;  // 无返回值
}

static int myled_get(lua_State *L)
{
    lua_pushboolean(L, gpio_get_level(MYLED_GPIO));
    return 1;  // 一个返回值
}

int luaopen_myled(lua_State *L)
{
    // 板级 GPIO 初始化通常集中在 board_manager；这里仅示例
    gpio_config_t cfg = { .pin_bit_mask = (1ULL << MYLED_GPIO), .mode = GPIO_MODE_OUTPUT };
    gpio_config(&cfg);

    lua_newtable(L);
    lua_pushcfunction(L, myled_set);  lua_setfield(L, -2, "set");
    lua_pushcfunction(L, myled_get);  lua_setfield(L, -2, "get");
    return 1;  // 返回 module table
}

esp_err_t lua_module_myled_register(void)
{
    return cap_lua_register_module("myled", luaopen_myled);
}
```

参数校验用 `luaL_checkinteger` / `luaL_checkstring`；底层失败用 `luaL_error(L, "...")` 让脚本以明确原因失败。

### 3. 头文件

```c
// src/lua_module_myled.h
#pragma once
#include "esp_err.h"
int lua_module_myled_register(void);
```

### 4. 写 README.md（API 文档，会同步进 docs）

```markdown
# Lua myled
Controls a single LED on GPIO 2.
## How to call
- `local myled = require("myled")`
- `myled.set(on)` — boolean
- `myled.get()` — returns boolean
## Example
local myled = require("myled")
local delay = require("delay")
myled.set(true); delay.delay_ms(500); myled.set(false)
## Rules
- 只控制 GPIO 2；不要在 50ms 内调用多次
```

### 5. test/ 与 lib/（可选）

`test/myled_smoke.lua`：自包含、含清理与资源释放的硬件验证脚本。
`lib/foo.lua`：实现库（非可运行程序），必须配 `lib/foo.md` 文档。

### 6. 在 app 注册（顺序！）

```c
// components/common/app_claw/app_lua_modules.c
#include "lua_module_myled.h"
void app_lua_modules_init(void)
{
    lua_module_display_register();
    lua_driver_gpio_register();
    lua_module_myled_register();        // ← 你的模块
    // …其它模块…
    cap_lua_register_group("/fatfs/scripts");  // 此后锁定，不能再注册
}
```

### 7. Kconfig 与依赖

在 `components/common/app_claw/Kconfig` 加（若希望可裁剪）：
```kconfig
config APP_CLAW_LUA_MODULE_MYLED
    bool "Enable myled Lua module"
    default y
    help
        Enable the myled Lua module ...
```
硬件相关模块用 `default y if SOC_XXX_SUPPORTED` / `depends on` 守护，别把板级假设塞进通用模块。

### 8. 配套 Skill（推荐）

`skills/lua_module_myled/SKILL.md`：frontmatter `cap_groups:["cap_lua"]`，正文给硬件摘要、init 故事、完整 API 表、用法规则、失败模式（详见 `recipes/write_skill.md`）。`lua_module_*` Skill 挂在 `cap_lua`，**不**绑定独立 group。

### 9. 构建、在脚本里用

```bash
idf.py build && idf.py flash monitor
# Console 把脚本写到 /fatfs/scripts，再：
cap call lua_run_script '{"path":"myled_smoke.lua"}'
```

## 常见错误

| 错误 | 原因 | 解决方法 |
|---|---|---|
| `require("myled")` 失败 / 模块不存在 | 未注册，或在 `register_group` 之后注册 | 在 `app_lua_modules.c` 里、`cap_lua_register_group` 之前调 `lua_module_myled_register()` |
| 模块编不进固件 | `APP_CLAW_LUA_MODULE_*` 默认 n | 在 menuconfig 启用，或设对 `default y if SOC_...` |
| 脚本报 `invalid gpio mode` | 字符串枚举不匹配 | 用 `luaL_error` 给明确原因；mode 用 `"input"/"output"/...` |
| `README.md` 没进 docs | 组件名不是 `lua_module_*` | `README.md` 同步只对 `lua_module_*` 生效 |
| `lib/foo.lua` 同步失败 | 缺同名 `lib/foo.md` | 每个被同步的 `lib/*.lua` 必须有同名 `.md` |

## 参考

- `components/lua_modules/lua_module_gpio/`（最小 C 绑定参考：`src/lua_module_gpio.c`、`README.md`）
- `components/lua_modules/lua_driver_gpio/README.md`、`lua_module_storage/README.md`、`lua_module_delay/README.md`、`lua_module_json/README.md`
- `components/common/app_claw/app_lua_modules.c`
- `docs/src/content/docs/en/reference-cap/lua-modules.mdx`
- `.agents/spec/lua-module-spec.md`
