# ESP-Brookesia 官方示例清单

> 路径相对仓库根 `espressif-repos/esp-brookesia/`。每条描述基于各示例 `README.md` / `main.cpp` 与 `docs/`。复制示例目录改造是创建工程的推荐方式。

## 应用示例（examples/）

| 示例路径 | 类型 | 一句话描述 |
|---|---|---|
| `examples/agent/chatbot` | 板级 / Agent | 完整 AI 语音助手：XiaoZhi(默认)/Coze/OpenAI 多 Agent 运行期切换、AFE 唤醒、Emote 表情、手势导航、MCP 硬件控制工具、Wi-Fi SoftAP 配网、NVS 持久化、运行期 profiling。要求 Flash≥16MB、PSRAM≥8MB |
| `examples/service/wifi` | 纯芯片 / Service | Wi-Fi 服务最完整范例：事件订阅、AP 扫描、STA 连接/断开、自动连接、SoftAP 配网、状态/历史查询、`ResetData`。`idf.py set-target esp32s3` |
| `examples/service/nvs` | 纯芯片 / Service | NVS 服务两套 API：通用 JSON `Set/Get/List/Erase` 与类型安全 `save_key_value`/`get_key_value`（含 struct/enum/容器），命名空间管理与错误处理 |
| `examples/service/audio` | 板级 / Service | Audio 服务全套：`SetPlaybackConfig`/编解码静态配置、单/多 URL 播放、`PlayControl`(暂停/恢复/停止)、PCM/OPUS/G711A 编解码回环、AFE(VAD+唤醒词)。需 `AudioCodecPlayer/Recorder` + `StorageFs` |
| `examples/service/device` | 板级 / Service | Device 服务：能力查询、板信息、显示背光亮度/开关、音频音量/静音、存储文件系统与容量、电池信息/状态/充电配置、事件监听、`ResetData`。需 HAL 接口 |
| `examples/service/console` | 板级 / 工具 | 服务控制台：串口 CLI 在线驱动任意服务/Agent（`svc_call`/`svc_subscribe`/`svc_funcs`/`svc_events`/`svc_list`/`svc_stop`），跨设备 Wi-Fi RPC（`svc_rpc_server`/`svc_rpc_call`/`svc_rpc_subscribe`/`svc_rpc_unsubscribe`，默认端口 65500），以及运行期剖析器（`debug_mem` 内/外堆、`debug_thread` 每核 CPU/栈、`debug_time_report`/`debug_time_clear` 配合 `BROOKESIA_TIME_PROFILER_SCOPE`）。含 XiaoZhi 配网/激活/对话完整串口流程。组件 `components/cmd_service`、`components/cmd_debug`、`main/modules/profiler.hpp`。文档：`docs/tutorial.md`、`docs/cmd_rpc.md`、`docs/cmd_debug.md`、`docs/en/service/manager/rpc.rst` |

## 示例内关键文件速查

| 示例 | 关键文件 |
|---|---|
| `examples/agent/chatbot` | `main/main.cpp`、`main/modules/ai_agents.cpp`、`main/modules/general_services.cpp`、`main/modules/wifi_provisioning.cpp`、`main/modules/profiler.cpp`、`main/modules/display/`、`main/Kconfig.projbuild`、`README.md` / `README_CN.md` |
| `examples/service/wifi` | `main/main.cpp`、`main/idf_component.yml`（含 `override_path` 依赖 nvs/wifi） |
| `examples/service/nvs` | `main/main.cpp`、`main/idf_component.yml` |
| `examples/service/audio` | `main/main.cpp`、`main/idf_component.yml` |
| `examples/service/device` | `main/main.cpp`、`main/idf_component.yml` |
| `examples/service/console` | `main/main.cpp`、`main/modules/console.cpp`、`main/modules/ai_agents.cpp`、`main/modules/emote.cpp`、`main/modules/profiler.cpp`、`components/cmd_debug/`、`components/cmd_service/`、`docs/tutorial.md`、`docs/cmd_debug.md`、`docs/cmd_rpc.md`、`main/Kconfig.projbuild` |

## 服务/Agent 组件 test_apps（额外可参考的小型用例）

各组件目录下含 `test_apps/`，可作为更小粒度的调用范例：

| 组件 | test_apps 路径 |
|---|---|
| `brookesia_lib_utils` | `utils/brookesia_lib_utils/test_apps/{check,describe_helpers,function_guard,log,memory_profiler,plugin,state_machine,task_scheduler,thread_config,thread_profiler,time_profiler}/` |
| `brookesia_hal_interface` | `hal/brookesia_hal_interface/test_apps/` |
| `brookesia_hal_adaptor` | `hal/brookesia_hal_adaptor/test_apps/` |
| `brookesia_service_audio` | `service/brookesia_service_audio/test_apps/` |
| `brookesia_service_custom` | `service/brookesia_service_custom/test_apps/` |
| `brookesia_service_device` | `service/brookesia_service_device/test_apps/` |
| `brookesia_service_manager` | `service/brookesia_service_manager/test_apps/` |
| `brookesia_service_nvs` | `service/brookesia_service_nvs/test_apps/` |
| `brookesia_service_sntp` | `service/brookesia_service_sntp/test_apps/` |
| `brookesia_service_wifi` | `service/brookesia_service_wifi/test_apps/` |

> test_apps 体量小、聚焦单一能力，适合作为某 API 的最小可运行参考。

## 选择建议

- **学服务框架通用范式** → `examples/service/wifi`（最完整、纯芯片、无需板）
- **学某具体服务** → 对应 `examples/service/<name>` + 组件 `test_apps`
- **学 AI Agent 完整产品形态** → `examples/agent/chatbot`
- **命令行在线调试服务/Agent** → `examples/service/console`
- **快速验证某底层能力** → 对应组件 `test_apps/<feature>`
