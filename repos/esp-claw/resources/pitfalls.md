# ESP-Claw 汇总陷阱（Must Read）

> 全部来自真实仓库文档（`docs/`、`.agents/gotchas.md`、各 cap 文档）与源码。SKILL.md 已列 15 条核心陷阱，本文件补充并细化。

## A. 启动与 Core

1. **`claw_core` 条件启动**：`llm_api_key` / `llm_model` / `llm_backend_type` 任一为空 → `app_claw_start` 跳过 `claw_core_init/start` 并 WARN。`ask`、默认 Agent 路由、图像检查不可用；但事件路由、自动化、本地能力、Console REPL 仍可用。配齐后**必须重启**。
2. **`system_prompt` 必填非空**：`claw_core_init` 校验 `system_prompt` 非空，否则失败。
3. **context provider 顺序固定**（缓存命中优化）：profile → long_term(full/_lightweight) → session_history → skills_list → tools。Skill 文档**不**走单独 provider，而是经 `activate_skill` 工具返回值注入会话历史，保持系统提示稳定。
4. **agent_stage 默认不发**：`CLAW_CORE_STAGE_VERBOSITY` 默认 Simple，路由层收不到 `agent_stage`；要 IM 进度推送需选 Verbose + 配 `agent_stage_im_notify` 规则。

## B. 路径与文件系统

5. **禁硬编码 `/fatfs`**：可写根在 SD 卡场景是 SD 挂载点。C 用 `claw_paths_join(CLAW_PATH_DATA, ...)`，Lua 用 `storage.get_root_dir()` + `storage.join_path(...)`。
6. **Skill 自带文件用 `{CUR_SKILL_DIR}`**：仅 SKILL.md **正文**展开（frontmatter 不展开）；文件工具要绝对路径、禁 `..`、禁反斜杠、禁 `../`。
7. **SYSTEM 只读、DATA 可写**：板子 `fatfs_image/` overlay 进 **SYSTEM**（只读），**不**进 DATA；隐藏目录不被采纳。`.recovery` 种子仅在 DATA 缺失时回拷。
8. **`cap_files` 读上限 32KB**（`CAP_FILES_MAX_FILE_SIZE`）；超出截断。大文件别整读进 LLM 上下文。
9. **`edit_file` 仅替换第一处**（`strstr` + 单次回写）；要批量替换读出→处理→`write_file` 整体覆盖。
10. **`write_file` 自动建父目录**；`copy_file`/`move_file` 源/目的都必须在受管根下，禁同源同目的。

## C. 能力与 Lua

11. **Lua 注册锁**：`cap_lua_register_module()` 仅在 `cap_lua_group_init`（即 `cap_lua_register_group`）前有效，之后返回 `ESP_ERR_INVALID_STATE`。所有 `lua_module_*_register()` 必须在 app init 里、`register_group` 之前。
12. **能力/Lua 选择是配置驱动**：启用组、LLM 可见组、启用 Lua 模块来自 app 配置。空选择通常表示「用默认/启用可用模块」；未知 token 被忽略并告警。新增能力/模块要同步更新注册表与 Kconfig 默认。
13. **模块按 SoC 默认关闭**：`lua_driver_pcnt/rmt/touch` 等默认 `y if SOC_..._SUPPORTED`，无对应硬件的芯片编不进。
14. **`lua_module_*` Skill 挂 `cap_lua`**：不绑定独立 group；`lua_driver_*` README 是 agent 面向文档（简洁、能力导向、运行时准确），不是用户营销或完整开发手册。
15. **`execute` 输出有限**：`output` 通常 4–8KB；大载荷分块/落盘/返回路径让上层 `read_file`；返回前释放临时分配。
16. **事件源防自环**：`claw_event_router` 自身发出的（`source_cap=claw_event_router`）事件会被跳过；仍要避免规则间互相 `emit_event` 造成循环。

## D. Skill

17. **frontmatter 必须合法 JSON**（非 YAML）；`name` 必须等于目录名；全局唯一；恰好一个 H1。
18. **激活按 `session_id` 隔离**：`activate_skill` 读 `ctx->session_id`；Console `skill --session` 要与活动 session 对齐，否则「激活了但不在上下文」。重启后激活态从 FATFS 恢复。
19. **`metadata.cap_groups` 是 group id**：如 `cap_lua`、`cap_im_qq`；激活时把这些组加入该 session 的 LLM allow-list。
20. **`description` 影响匹配**：要写用户意图与关键前提（如「Requires board_hardware_info skill」），别只写内部脚本名。
21. **构建同步规则**：每个 `skills/<skill_id>/SKILL.md` 必须存在；skill id 全局唯一；两个组件产出相同输出路径则构建失败；旧 manifest 记录的已删文件会被清理。

## E. 自动化与调度

22. **`run_agent` 异步**：响应以 `out_message` 事件回路由器；要发到 IM 必须配一条匹配 `source_cap=claw_core, event_type=out_message` 的 `send_message` 规则。`channel`/`chat_id` 来自触发时传入的 `target_channel`/`target_chat_id`。
23. **动作参数放 `input`**：`run_script`/`send_message`/`emit_event` 的参数放 `actions[*].input`，不放 action 顶层。
24. **`text_match_rule:"prefix"`**：路由左裁剪、要求命令后有词边界，剩余参数为 `{{match.remainder}}`；默认是精确匹配。
25. **调度与执行分离**：`cap_scheduler` 只发 `schedule` 事件；行为靠 router rule。只加 schedule 不加 rule → 什么都不发生。
26. **cron/once 需 SNTP**：系统时间早于 2024-01-01 不触发；首次同步成功 `cap_scheduler_handle_time_sync` 重基准。`interval` 不依赖外部时间，上电即计时。
27. **`scheduler_add` 入参是字符串**：`schedule_json` 是转义 JSON 串，不是嵌套对象。
28. **手编 `router_rules.json`/`schedules.json` 后**：必须 `auto reload` / `scheduler --reload` 才生效。

## F. Lua 脚本

29. **`event_publisher` 仅点号调用**：`ep.publish_message(...)`；冒号 `ep:publish_message(...)` 会把 self 当首参。table 形式必填 `source_cap` 与 `text`；回调里优先用字符串形式。
30. **长循环让出 CPU**：runtime 有指令钩子+墙钟超时保护，但轮询/动画循环里仍建议 `delay.delay_ms(...)` 协作式 yield，避免任务看门狗。
31. **`lua_run_script` 路径规则**：写时 `scripts/x.lua`，跑时去前导 `{"path":"x.lua"}`；内置相对路径（如 `builtin/test/hello.lua`）不得含 `..`；Skill 脚本用 `{CUR_SKILL_DIR}/scripts/...`。
32. **async 互斥与抢占**：`lua_run_script_async` 的 `exclusive`（如 `"display"`）做单槽互斥；`replace:true` 抢占同名/同组任务。`display` 模块有所有权仲裁，`display.init` 获前台所有权，`deinit`/脚本退出释放。

## G. 配置与运行时

33. **NVS 覆盖 menuconfig**：运行时值 = NVS 有则用 NVS，否则编译默认。改 menuconfig 默认值不一定生效；恢复出厂清 NVS 键。
34. **`app_behavior` / AP 回落**：STA 连续失败 `APP_WIFI_MAX_RETRY`（默认 5）次后回落纯 AP 等待重新配网。
35. **微信扫码登录**：`CONFIG_APP_CLAW_CAP_IM_WECHAT` 时 Web 暴露 `cap_im_wechat_qr_login_*` 状态接口；未扫码则不在线。
36. **安全**：配置页与 NVS 含 secret token；不要公开分享导出配置或 NVS dump。

## 参考来源

- `docs/src/content/docs/en/reference-project/{boot-and-runtime,dataflow-and-automation,configuration,console-usage,lua,memory,skills}.mdx`
- `docs/src/content/docs/en/reference-cap/{cap-lua,lua-modules,cap-files,cap-scheduler,cap-skill,cap-system,cap-im-platform,cap-mcp}.mdx`
- `docs/src/content/docs/en/reference-core/{claw-core,claw-cap,claw-event-router,claw-memory}.mdx`
- `.agents/{design,gotchas}.md`、`.agents/spec/claw-skill-spec.md`
- `application/edge_agent/fatfs_image/system/.recovery/router_rules/router_rules.json`
- `application/edge_agent/fatfs_image/storage/skills/{light_switch,scheduled_task}/`
