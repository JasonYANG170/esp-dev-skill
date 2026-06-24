# Changelog

本 Skill 遵循 ESP-Brookesia 仓库的真实文档与源码。所有变更仅记录 Skill 自身的迭代。

## [1.1.0] - 2026-06-22

补齐服务控制台（最大的有文档无配方特性）的配方与 API 速查。

### 新增

- **recipes/service_console_rpc_debug.md** — 服务控制台配方，覆盖三块此前无配方的能力：
  1. **服务 CLI**：`svc_list`/`svc_funcs`/`svc_events`/`svc_call`/`svc_subscribe`/`svc_unsubscribe`/`svc_stop`（含 XiaoZhi 配网/激活/对话完整串口流程、`svc_call` JSON 无空格约束、5000ms 固定超时、`svc_call_periodic`/`svc_call_delayed`/`svc_call_cancel`/`svc_call_list`）。
  2. **跨设备 RPC**：`svc_rpc_server start/stop/connect/disconnect`、`svc_rpc_call`、`svc_rpc_subscribe`、`svc_rpc_unsubscribe`（默认端口 65500、超时 2000ms、`host:port` 连接复用），含被调方/调用方双端拓扑与程序化用法（`ServiceManager::start_rpc_server`/`connect_rpc_server_to_services`/`rpc::Server`/`rpc::Client`）。
  3. **运行期剖析器**：`debug_mem`（内/外堆快照）、`debug_thread`（`-p/-s/-d` 排序与采样）、`debug_time_report`/`debug_time_clear`（配合 `BROOKESIA_TIME_PROFILER_SCOPE`/`_START_EVENT`/`_END_EVENT` 打点）。
- **resources/api_reference.md**：新增 "Service Console 命令"、"RPC（service::rpc）"、"运行期剖析器（lib_utils）" 三节，含 `rpc::Server`/`rpc::Client`/`MemoryProfiler`/`ThreadProfiler`/`TimeProfiler` 的真实签名与 RPC 相关 Kconfig 默认值。
- **SKILL.md**：Scenario Quick Reference 的"通用服务"表新增 `service_console_rpc_debug.md` 行。
- **resources/example_list.md**：`examples/service/console` 描述扩充为覆盖 svc_* CLI / RPC / debug_* 剖析器并标注默认端口与配套文档。

### 依据（grounding）

- `examples/service/console/README.md`、`docs/tutorial.md`、`docs/cmd_rpc.md`、`docs/cmd_debug.md`
- `examples/service/console/components/cmd_service/cmd_service.cpp`（所有 `svc_*`/`svc_rpc_*` 命令注册与 `get_or_create_rpc_client`）
- `examples/service/console/components/cmd_debug/cmd_debug.cpp`（`debug_*` 命令）
- `examples/service/console/main/modules/profiler.hpp`、`main/modules/console.hpp`、`main/main.cpp`
- `service/brookesia_service_manager/include/brookesia/service_manager/service/manager.hpp`（`start_rpc_server` 等）
- `service/brookesia_service_manager/include/brookesia/service_manager/rpc/server.hpp`、`.../rpc/client.hpp`
- `service/brookesia_service_manager/Kconfig`（RPC 端口/连接/超时默认）
- `utils/brookesia_lib_utils/include/brookesia/lib_utils/{memory,thread,time}_profiler.hpp`
- `docs/en/service/manager/rpc.rst`

## [1.0.0] - 2026-06-18

首个正式版本，基于 ESP-Brookesia master/v0.7（ESP-IDF >= v5.5）。

### 新增

- **SKILL.md**：核心原则（12 条）、When to Use、10 个配方索引、组件分层与板支持表（8 块 Espressif 板）、版本依赖表、服务调用统一范式与返回类型、12 条关键踩坑（含错误/正确代码对照）、执行流程表与示例选择策略、失败策略表、参考链接。
- **AGENTS.md**：项目上下文（C++/ESP/ESP-IDF）、文件命名、include 模式、命名空间与 Helper 别名、标准项目结构、标准 `app_main` 范式、构建工作流（`gen-bmgr-config` / `set-target` / `idf_ext.py`）、CRTP Helper 扩展约定、codegen 检查清单、Do Not Modify 说明。
- **recipes/**（10 个配方）：
  - `project_setup.md` — 新建项目、依赖声明、芯片/板选择、最小 app_main
  - `service_framework_basics.md` — ServiceManager 启动、bind、同步/异步调用、事件订阅、EventMonitor
  - `wifi_service.md` — 扫描、连接/断开、SoftAP 配网、状态查询、事件订阅
  - `nvs_service.md` — 类型安全 API 与通用 JSON API、命名空间管理、错误处理
  - `audio_service.md` — 配置、单/多 URL 播放、PlayControl、编解码回环、AFE
  - `device_service.md` — 能力查询、板信息、背光/音量/存储/电池、事件监听、ResetData
  - `sntp_service.md` — NTP 服务器、时区、与 Agent TimeSyncing 联动
  - `custom_service.md` — 运行期动态注册 function/event
  - `agent_chatbot.md` — 完整 AI 语音助手：AgentManager、多 Agent、MCP 工具、状态机、Emote 联动
  - `expression_emote.md` — 资源加载、SetEmoji、事件消息、二维码、动画插入
  - `hal_boards.md` — 选板、板配置目录结构、设备初始化、接口查询、新增自定义板
- **resources/**（4 个速查）：
  - `api_reference.md` — Base CRTP、序列化宏、日志/检查宏、ServiceManager、Wi-Fi/NVS/Audio/Device/Agent/Emote Helper、HAL 接口、TaskScheduler 的真实签名与枚举
  - `config_reference.md` — 各组件 Kconfig 代表性符号、组件 Kconfig 文件清单、示例级 Kconfig、板级默认
  - `pitfalls.md` — 服务框架/调用参数/事件/HAL/Agent/内存构建/MCP 七大类踩坑展开
  - `example_list.md` — 6 个应用示例 + 各组件 test_apps 清单与选择建议
- **README.md** / **CHANGELOG.md**。

### 依据（grounding）

全部 API、枚举、宏、Kconfig 符号、文件路径与代码片段来自：
`espressif-repos/esp-brookesia/docs/en/**`、`examples/**/main.cpp`、各组件 `include/` 与 `Kconfig`、`idf_component.yml`。
