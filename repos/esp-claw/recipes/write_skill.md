# 编写 ESP-Claw Skill

> **适用摘要**: 按 ESP-Claw Skill 规范创建 `skills/<skill_id>/SKILL.md`（JSON frontmatter + 正文 + 可选 scripts/references/assets），让 LLM 按需激活并获得工具与使用指南。

## 触发意图

- "写一个 Skill"
- "SKILL.md frontmatter 怎么写"
- "cap_groups / manage_mode"
- "{CUR_SKILL_DIR} 占位符"
- "让 Agent 学会某个能力"

## 前置条件

| 条件 | 要求 |
|---|---|
| 已读 | `.agents/spec/claw-skill-spec.md`、`docs/.../reference-project/skills.mdx` |
| 规范 | `name` 必须等于目录名；全局唯一；恰好一个 H1 |
| 参考 | `application/edge_agent/fatfs_image/storage/skills/light_switch/SKILL.md` |

## 分步说明

### 1. 目录结构

```
skills/<skill_id>/
├── SKILL.md            # 必需
├── references/guide.md # 可选
├── scripts/action.lua  # 可选（Skill 自带只读脚本）
└── assets/image.bin    # 可选
```
- 内置/组件 Skill 构建期同步进只读 SYSTEM 镜像 `/system/skills/<id>/`
- 运行时/用户 Skill 进可写 DATA 根 `<DATA>/skills/<id>/`（DATA = `/fatfs` 或 SD 卡挂载点）
- 冲突时 DATA Skill 优先

### 2. SKILL.md frontmatter（必须是合法 JSON 对象）

```md
---
{
  "name": "light_switch",
  "description": "Turn a board light on or off, set LED strip color or brightness, and control GPIO lights. Requires board_hardware_info skill.",
  "metadata": {
    "cap_groups": ["cap_lua"],
    "manage_mode": "readonly",
    "category": ["utility"],
    "peripherals": [],
    "tags": ["light", "led"]
  }
}
---
# Light Switch
```

字段规则：
- `name`：非空，**等于**目录名；稳定 id（小写字母/数字/下划线/连字符）
- `description`：描述用户意图与关键前提（影响匹配），别只写内部脚本名
- `metadata.cap_groups`：可选；能力组 id 数组。激活时这些组对 LLM 可见（工具 + 指南同一生命周期）
- `metadata.manage_mode`：`readonly` / `web` / `runtime`（Skills Lab 用 `web`，设备上等同 `readonly`）
- `metadata.category`：至少一个，须在允许列表内
- `metadata.peripherals`：0 个或多个，须在允许列表内
- `metadata.tags`：可选自由字符串数组，不得与 category/peripherals 重复

### 3. 正文写给 LLM（激活后整篇注入会话历史）

强 Skill 应含：场景框架（何时激活）、调用规则（参数约束/顺序/限频）、可直接复制的 JSON 示例、错误剧本、跨工具注意。

#### cap_* 型 Skill（JSON 工具调用）
````markdown
# My Feature

何时激活：用户要……

## Usage rules
- 调 `my_action` 时 `param` 必须非空
- 失败先校验 `param`，最多重试一次
- 连续调用不超过 3 次，间隔 ≥ 500ms

## Example calls
```json
{"param":"hello"}
```

## Error handling
- `Error: param is required` → 补上 `param` 再试
- `Error: invalid state` → 设备未就绪，让用户稍后再试
````

#### lua_module_* 型 Skill（LLM 生成 Lua）
给硬件摘要、init 故事、完整 API 表（参数/返回/约束）、失败模式。例：
````markdown
# myled
Controls a single LED on GPIO 2.
## Setup
无需显式 init，首次 `require` 时配置 GPIO 2。
## API
### `myled.set(on)` — boolean
### `myled.get()` → boolean
## Rules
- 仅控制 GPIO 2；50ms 内不要调用多次
````

### 4. 引用自带脚本/资源用 `{CUR_SKILL_DIR}`

```json
{"path":"{CUR_SKILL_DIR}/scripts/led_strip_switch.lua","args":{"enabled":true}}
```
- `{CUR_SKILL_DIR}` **只在正文展开**，frontmatter 不展开
- 展开后指向设备文件系统中该 Skill 目录（内置在 `/system/skills/<id>`，用户在 `<DATA>/skills/<id>`）
- 文件工具路径必须绝对、禁 `..`、禁 `../`

### 5. Skill 自带 Lua 脚本规则

放 `scripts/action.lua`：
- 只读、随 Skill 分发
- 用户动作应做成独立 Skill，而非 `test/` 条目
- 执行失败应让模型直接报错，**不要**改参数重试

### 6. 安装 / 注册 / 激活

- 组件 Skill：放 `components/<comp>/skills/<id>/SKILL.md`，构建期 `skill_builder` 同步进 SYSTEM 镜像
- 运行时 Skill：用 `register_skill` 工具（写 `<DATA>/skills/<id>/SKILL.md` 并 reload），或 Web 文件管理器，或 Skills Lab 下载
- 激活：LLM 调 `activate_skill {"skill_id":"..."}`（返回完整文档并打开 `cap_groups`）；Console 用 `cap call activate_skill '{"skill_id":"..."}'`

> `activate_skill` 把文档作为**工具返回值**注入会话历史（非系统提示），保持系统提示稳定以提高 LLM 缓存命中率。多 Skill 可在一轮并行 `activate_skill`。

## 常见错误

| 错误 | 原因 | 解决方法 |
|---|---|---|
| 构建失败 / 同步失败 | `name` 与目录名不一致；或 id 全局重复 | `name` 严格等于目录名；id 全局唯一 |
| frontmatter 解析失败 | 用了 YAML 风格 / JSON 语法错 | 必须是合法 JSON 对象，`---` 包裹 |
| 缺 H1 或多个 H1 | 规范要求恰好一个 H1 | 只留一个 `# Title` |
| 激活后工具不可见 | `cap_groups` 写错 group id | 用能力组 id（如 `cap_lua`、`cap_im_qq`） |
| Skill 自带脚本路径错 | 写死 `/fatfs/skills/...` | 用 `{CUR_SKILL_DIR}/scripts/...` |
| 激活了但「不在上下文」 | `--session` 与活动 session 不一致 | 激活按 `session_id` 隔离，对齐当前 session |

## 参考

- `.agents/spec/claw-skill-spec.md`（目录结构 / frontmatter / `{CUR_SKILL_DIR}` / 构建同步 / 命名冲突）
- `application/edge_agent/fatfs_image/storage/skills/light_switch/SKILL.md`（含 LED Strip / GPIO Light 完整 schema 与 Recommended Flow）
- `application/edge_agent/fatfs_image/storage/skills/scheduled_task/SKILL.md`
- `application/edge_agent/fatfs_image/storage/skills/plan_mode/SKILL.md`
- `docs/src/content/docs/en/reference-project/skills.mdx`
- `docs/src/content/docs/en/reference-cap/cap-skill.mdx`
