# 服务控制台：CLI 调试、跨设备 RPC 与运行期性能剖析

> **适用摘要**: 使用 ESP-Brookesia 自带的 `examples/service/console` 串口交互 CLI，在运行期驱动任意服务/Agent（`svc_call`/`svc_subscribe` 等）、跨设备经 Wi-Fi 进行 RPC 远程调用与事件订阅（`svc_rpc_server`/`svc_rpc_call`/`svc_rpc_subscribe`），并用 `debug_mem`/`debug_thread`/`debug_time_report` 三类剖析器诊断内存、CPU 与耗时。本配方不写应用层 C++，而是教你编译烧录该控制台示例、用命令在线编排设备——这是框架对外宣称的 "Agent CLI" 与 "RPC-based remote communication" 能力的唯一官方入口。

## 触发意图

- "怎么在串口里直接调用服务函数"
- "svc_call 怎么用 / 参数怎么传"
- "跨设备调用服务 / 远程调试另一台板子"
- "svc_rpc_server / svc_rpc_call / svc_rpc_subscribe"
- "看设备内存（PSRAM/SRAM）占用 / 哪个线程吃 CPU / 函数耗时"
- "debug_mem / debug_thread / debug_time_report"
- "BROOKESIA_TIME_PROFILER_SCOPE 怎么打点"

## 前置条件

| 条件 | 要求 |
|---|---|
| 参考示例 | `examples/service/console`（板级工程，含 `components/cmd_service`、`components/cmd_debug`、`main/modules/profiler.hpp`） |
| 参考文档 | `examples/service/console/README.md`、`examples/service/console/docs/tutorial.md`、`docs/cmd_rpc.md`、`docs/cmd_debug.md`、`docs/en/service/manager/rpc.rst` |
| 硬件 | 任一受支持板（如 `esp_vocat_board_v1_2`、`esp_box_3`、`esp32_p4_function_ev`），Flash ≥ 8MB、PSRAM ≥ 4MB；RPC 两台设备须在同一 Wi-Fi |
| 构建 | 板级工程用 `idf.py gen-bmgr-config -b <board>`（不是 `set-target`），见 SKILL.md 踩坑 #12 |
| 网络（仅 RPC） | 两台设备均需先 `svc_call Wifi SetConnectAp ...` 连入同一 AP |

## 分步说明

### 1. 编译烧录控制台示例

控制台是一个**板级工程**（依赖 HAL 板配置），必须用 `gen-bmgr-config` 选板：

```bash
cd examples/service/console
idf.py gen-bmgr-config -b esp_vocat_board_v1_2
idf.py build
idf.py -p <PORT> flash monitor
```

启动后按回车，看到提示符 `esp32s3>` / `esp32p4>` / `esp32c5>` 即可输入命令。输入 `help` 列出所有已注册命令（含 `svc_*`、`svc_rpc_*`、`debug_*`）。

> 命令历史默认持久化到 Flash（`CONFIG_CONSOLE_STORE_HISTORY=y`，见 `main/Kconfig.projbuild`）；`CONFIG_ESP_CONSOLE_SECONDARY_NONE=y` 关闭副串口输出。

### 2. 服务 CLI：在线驱动任意服务/Agent

控制台核心是 `components/cmd_service/cmd_service.cpp` 注册的一组 `svc_*` 命令，它们在内部对 `ServiceManager` 做 `bind()` → `call_function_sync` / `subscribe_event`，等价于在代码里写 Helper 调用。命令名即服务名/函数名/事件名（首字母大写，如 `Wifi`、`TriggerGeneralAction`）。

| 命令 | 作用 | 示例 |
|---|---|---|
| `svc_list` | 列出所有已注册服务及状态 | `svc_list` |
| `svc_funcs <service>` | 列出某服务的全部函数及 schema | `svc_funcs Wifi` |
| `svc_events <service>` | 列出某服务的全部事件 | `svc_events Wifi` |
| `svc_call <service> <function> [JSON]` | 调用一个函数（JSON 参数无空格） | `svc_call Wifi SetConnectAp {"SSID":"ssid","Password":"pwd"}` |
| `svc_subscribe <service> <event>` | 订阅事件 | `svc_subscribe Wifi ScanApInfosUpdated` |
| `svc_unsubscribe <service> <event>` | 取消订阅 | `svc_unsubscribe Wifi ScanApInfosUpdated` |
| `svc_stop <service>` | 释放该服务的 binding（停止服务） | `svc_stop Wifi` |

`svc_call` 的 JSON 参数**不能含空格**——错误写法 `svc_call Wifi TriggerGeneralAction { "Action": "Start" }`，正确写法是 `svc_call Wifi TriggerGeneralAction {"Action":"Start"}`。无参函数传 `{}` 或省略。

完整启动 XiaoZhi Agent 的串口流程（摘自 `docs/tutorial.md`）：

```bash
# 1. 连 Wi-Fi（Agent 依赖网络）
svc_call Wifi SetConnectAp {"SSID":"ssid","Password":"password"}
svc_call Wifi TriggerGeneralAction {"Action":"Connect"}

# 2. 点亮背光 + 加载 Emote 资源
svc_call Device SetDisplayBacklightOnOff {"On":true}
svc_call Emote LoadAssetsSource {"Source":{"source":"anim_icon","type":"PartitionLabel","flag_enable_mmap":false}}

# 3. 选定 Agent 并激活（先订阅激活码事件）
svc_subscribe AgentXiaoZhi ActivationCodeReceived
svc_call AgentManager SetTargetAgent {"Name":"AgentXiaoZhi"}
svc_call AgentManager TriggerGeneralAction {"Action":"Activate"}

# 4. 激活成功后启动
svc_call AgentManager TriggerGeneralAction {"Action":"Start"}
```

> 注：`svc_subscribe AgentXiaoZhi ...` 可能先打印 `Service is not bindable: AgentXiaoZhi`，这是预期日志（Agent 未 Activate），按教程忽略即可。

运行期控制 Agent：

```bash
svc_call AgentManager InterruptSpeaking
svc_call AgentManager TriggerGeneralAction {"Action":"Sleep"}
svc_call AgentManager TriggerGeneralAction {"Action":"WakeUp"}
svc_call AgentManager GetGeneralState
```

> `svc_call` 内部固定超时 `DEFAULT_SVC_CALL_TIMEOUT_MS = 5000`（见 `cmd_service.cpp`）。`svc_call_periodic` / `svc_call_delayed` / `svc_call_cancel` / `svc_call_list` 提供周期/延迟调用与取消（在同一组件注册）。

### 3. 跨设备 RPC：远程调用函数与订阅事件

RPC 让一台设备经 Wi-Fi（TCP）调用另一台设备上的服务函数、订阅其事件，实现集中式多设备管理或远程调试。底层是 `brookesia_service_manager/rpc/` 的 `Server` / `Client` / `DataLinkServer` / `DataLinkClient`，由 `ServiceManager` 的 `start_rpc_server` / `connect_rpc_server_to_services` / `stop_rpc_server` 托管。

**拓扑：** 设备 A（被调方）启动 RPC Server 并把服务挂上去；设备 B（调用方）用 `svc_rpc_call` / `svc_rpc_subscribe` 指向 A 的 IP。

设备 A（RPC Server，IP 假设 `192.168.1.100`）：

```bash
# 1. 先连入 Wi-Fi 并记下本机 IP
svc_call Wifi SetConnectAp {"SSID":"ssid","Password":"password"}
svc_call Wifi TriggerGeneralAction {"Action":"Connect"}

# 2. 在默认端口 65500 启动 RPC Server
svc_rpc_server start
#   或自定义端口：svc_rpc_server start -p 9000

# 3. 把服务挂到 RPC Server（被远程访问的前提）
svc_rpc_server connect                 # 挂全部已运行服务
#   或只挂指定服务：svc_rpc_server connect -s NVS,Wifi

# 4. 停止 / 卸下服务
svc_rpc_server disconnect -s Wifi       # 卸下指定服务
svc_rpc_server stop                     # 停止 RPC Server
```

设备 B（RPC Client）：

```bash
# 1. 连入同一 Wi-Fi
svc_call Wifi SetConnectAp {"SSID":"ssid","Password":"password"}
svc_call Wifi TriggerGeneralAction {"Action":"Connect"}

# 2. 调用 A 上的函数（无参用 {} 或省略）
svc_rpc_call 192.168.1.100 Wifi TriggerScanStart
svc_rpc_call 192.168.1.100 NVS Set {"KeyValuePairs":{"key1":"value1"}}
#   自定义端口与超时（默认 port=65500, timeout=2000ms）
svc_rpc_call 192.168.1.100 NVS List -p 9000 -t 10000
#   本机回环也可：svc_rpc_call 127.0.0.1 NVS List

# 3. 订阅 A 上的事件（收到时在本机串口打印）
svc_rpc_subscribe 192.168.1.100 Wifi ScanApInfosUpdated
svc_rpc_subscribe 192.168.1.100 Wifi GeneralEventHappened -t 5000

# 4. 取消订阅
svc_rpc_unsubscribe 192.168.1.100 Wifi ScanApInfosUpdated
```

| RPC 命令 | 关键参数 | 默认值 |
|---|---|---|
| `svc_rpc_server <start\|stop\|connect\|disconnect>` | `-p <port>`、`-s <svc1,svc2>` | port=65500 |
| `svc_rpc_call <host> <service> <function> [JSON]` | `-p <port>`、`-t <ms>` | port=65500, timeout=2000ms |
| `svc_rpc_subscribe <host> <service> <event>` | `-p <port>`、`-t <ms>` | port=65500, timeout=2000ms |
| `svc_rpc_unsubscribe <host> <service> <event>` | `-p <port>`、`-t <ms>` | port=65500, timeout=2000ms |

> **连接复用：** 同一 `host:port` 的 RPC Client 会被缓存复用（`get_or_create_rpc_client`，见 `cmd_service.cpp`），多次 `svc_rpc_call`/`svc_rpc_subscribe` 共享一条连接以提升性能；断线会自动尝试重连。
>
> **服务端默认值（Kconfig）：** 监听端口 `CONFIG_BROOKESIA_SERVICE_MANAGER_RPC_SERVER_LISTEN_PORT=65500`、最大连接数 `..._RPC_SERVER_MAX_CONNECTIONS=2`、Client 默认调用超时 `..._RPC_CLIENT_CALL_FUNCTION_TIMEOUT_MS=2000`、全局最大 socket 数 `..._RPC_GLOBAL_MAX_SOCKETS`（取 `LWIP_MAX_SOCKETS` 与 `LWIP_MAX_ACTIVE_TCP` 的较小值）。

**程序化使用 RPC（不依赖 CLI）：** 应用代码也可直接用 `ServiceManager` 与 `rpc::Server`/`rpc::Client`（命令内部就是这么做的）：

```cpp
#include "brookesia/service_manager.hpp"
using namespace esp_brookesia::service;

auto &sm = ServiceManager::get_instance();
sm.start();

// 设备 A：启动 RPC Server 并挂上服务
rpc::Server::Config cfg;                 // listen_port=65500, max_connections=2（默认）
sm.start_rpc_server(cfg, 5000);          // 启动超时 5000ms
sm.connect_rpc_server_to_services({"Wifi", "NVS"});  // 空 vector = 全部
// ...
sm.disconnect_rpc_server_from_services({"Wifi"});
sm.stop_rpc_server();
bool up = sm.is_rpc_server_running();

// 设备 B：构造 Client 调用远程函数
auto client = std::make_shared<rpc::Client>();
client->init(sm.get_executor(), /*on_disconnect=*/[]{});
client->connect("192.168.1.100", 65500, 2000);
auto result = client->call_function_sync("Wifi", "TriggerScanStart",
                                         boost::json::object{}, 2000);
if (result.success && result.has_data()) { /* result.data.value() */ }

std::string sub_id = client->subscribe_event(
    "Wifi", "ScanApInfosUpdated",
    [](const std::string &event, const boost::json::value &data){ /* ... */ },
    2000);
client->unsubscribe_events("Wifi", {sub_id}, 2000);
```

`rpc::Client` 还提供 `call_function_async`（返回 `std::future<FunctionResult>`）、`is_connected()`、`disconnect()`、`deinit()`。完整签名见 `resources/api_reference.md` 与 `docs/en/service/manager/rpc.rst`。

### 4. 运行期性能剖析（debug_*）

控制台内置三类剖析器（`components/cmd_debug/cmd_debug.cpp` 注册），用于诊断 SKILL.md 反复强调的内存/CPU 门槛（Flash≥8MB、PSRAM≥4MB、多核 SPI LCD 锁核）。剖析器由 `main/modules/profiler.hpp` 的 `Profiler` 单例在 `app_main` 中初始化（`start_thread_profiler(false)` / `start_memory_profiler(false)`，关闭自动日志）。

**4a. 内存剖析器 `debug_mem`** —— 一次性快照，分 SRAM/PSRAM 两行：

```bash
debug_mem
```

输出（摘自 `docs/cmd_debug.md`）含 `Heap Type`（Internal (SRAM) / External (PSRAM)）、Total/Free/Largest (KB)、Used %，以及统计区 `Min Inter Free`、`Min Exter Free` 等历史最小值。底层是 `lib_utils::MemoryProfiler::take_snapshot()` + `print_snapshot()`。

**4b. 线程剖析器 `debug_thread`** —— 采样各任务 CPU%/优先级/栈/运行核：

```bash
debug_thread                              # 默认 -p core -s cpu -d 1000
debug_thread -p core -s cpu -d 2000       # 主排序 core，次排序 cpu，采样 2000ms
debug_thread -s stack                     # 只改次排序为 stack
```

| 参数 | 含义 | 可选值 | 默认 |
|---|---|---|---|
| `-p` | 主排序 | `none`、`core` | `core` |
| `-s` | 次排序 | `cpu`、`priority`、`stack`、`name` | `cpu` |
| `-d` | 采样时长 (ms) | 任意正整数 | `1000` |

命令会采样两次（间隔 `-d` ms）以计算 CPU%，对应 `ThreadProfiler::PrimarySortBy::{None,CoreId}` 与 `SecondarySortBy::{CpuPercent,Priority,StackUsage,Name}`。输出每行一个任务：`Name | CoreId | CPU% | Priority | HWM | Stack(Intr/Extr) | Run Time | State`。注意空闲任务（`IDLE0/1`）CPU% 高是正常的。

**4c. 时间剖析器 `debug_time_report` / `debug_time_clear`** —— 报告代码区段耗时树：

```bash
debug_time_report      # 打印 Performance Tree Report（calls/total/self/avg/min/max/%parent/%total）
debug_time_clear       # 清空历史数据
```

时间剖析需要在代码里显式打点（仅打点过的区段才会出现在 report 中）。三个宏（来自 `brookesia/lib_utils/time_profiler.hpp`）：

```cpp
// 作用域自动计时（构造 enter / 析构 leave）
BROOKESIA_TIME_PROFILER_SCOPE("my_function");

// 手动起止（必须成对，跨函数时用）
BROOKESIA_TIME_PROFILER_START_EVENT("encode");
// ... 待测代码 ...
BROOKESIA_TIME_PROFILER_END_EVENT("encode");
```

`report()` 输出形如 `|- app_main | calls=1 | total=922.31 | self=922.31 | %total=92.67%`（单位 ms），可定位热点。`TimeProfiler::get_instance().report()` / `.clear()` 是其程序化入口。

> **调试日志：** 需要看 RPC 数据链路细节时，在 menuconfig 开 `BROOKESIA_SERVICE_MANAGER_ENABLE_DEBUG_LOG=y`，再按需勾选子项 `..._RPC_DATA_LINK_ENABLE_DEBUG_LOG` / `..._RPC_SERVER_ENABLE_DEBUG_LOG` / `..._RPC_CLIENT_ENABLE_DEBUG_LOG`。注意开 debug 日志可能拖慢调用导致超时，此时调大 `CONFIG_BROOKESIA_SERVICE_MANAGER_DEFAULT_CALL_FUNCTION_TIMEOUT_MS`（默认 500）。

## 常见错误

| 错误 | 原因 | 解决方法 |
|---|---|---|
| `Command not recognized` 或 `help` 里没有 `svc_*` | 固件未烧对 / 串口连错 | 重新 `idf.py -p <PORT> flash monitor`，确认进的是 `examples/service/console` 工程 |
| `Service not found` | 服务未在 menuconfig 启用或未 `bind` | `svc_list` 看是否列出；未列出则确认对应服务组件已加入 `idf_component.yml` 并在 menuconfig 启用 |
| `svc_call` 报 JSON 解析错误 | 参数 JSON 里含空格 | 去掉空格：`{"Action":"Start"}` 而非 `{ "Action": "Start" }` |
| `svc_call` 返回错误但代码里正常 | 函数 schema 参数类型/顺序不符 | 先 `svc_funcs <service>` 查 schema，按 String/Number/Boolean/Object/Array/RawBuffer 六类型核对 |
| RPC `Connection failed` | 两台设备不在同一 Wi-Fi / 对方未 `svc_rpc_server start` | 两端连同一 AP；被调方先 `svc_rpc_server start` 再 `svc_rpc_server connect` |
| RPC `Service not found` | 被调方没把该服务挂到 Server | 被调方执行 `svc_rpc_server connect -s <service>` |
| RPC 调用超时 | 默认 2000ms 太短或开 debug 日志拖慢 | 加 `-t 10000`；或调大 `..._RPC_CLIENT_CALL_FUNCTION_TIMEOUT_MS` / `..._DEFAULT_CALL_FUNCTION_TIMEOUT_MS` |
| RPC Server 启动失败 | 端口被占或超出最大连接数 | 换 `-p <port>`；或调大 `..._RPC_SERVER_MAX_CONNECTIONS`（默认 2） |
| `debug_thread` CPU% 全 0 | 采样间隔太短或全在阻塞态 | 加大 `-d`（如 2000）；空闲任务高 CPU% 正常 |
| `debug_time_report` 为空 | 代码里没用 `BROOKESIA_TIME_PROFILER_*` 打点 | 在待测区段加 `BROOKESIA_TIME_PROFILER_SCOPE("name")` |
| 板级工程用 `set-target` 编译缺组件 | 控制台是板级工程 | 用 `idf.py gen-bmgr-config -b <board>`（见 SKILL.md 踩坑 #12） |

## 参考项目

- `examples/service/console` — 服务控制台示例（板级工程，含 `cmd_service`/`cmd_debug` 组件、`docs/tutorial.md`、`docs/cmd_rpc.md`、`docs/cmd_debug.md`）
- `examples/service/console/components/cmd_service/cmd_service.cpp` — 所有 `svc_*` / `svc_rpc_*` 命令的注册与实现（含 `get_or_create_rpc_client` 连接复用）
- `examples/service/console/components/cmd_debug/cmd_debug.cpp` — `debug_mem`/`debug_thread`/`debug_time_report`/`debug_time_clear` 命令实现
- `examples/service/console/main/modules/profiler.hpp` — `Profiler` 单例封装（启动 thread/memory profiler）
- `examples/service/console/main/modules/console.hpp` — `Console` 单例（`start()`）
- `examples/service/console/main/main.cpp` — 完整初始化序列（HAL→ServiceManager→TaskScheduler→Profiler→Console）
- `service/brookesia_service_manager/include/brookesia/service_manager/service/manager.hpp` — `ServiceManager::start_rpc_server` / `connect_rpc_server_to_services` / `is_rpc_server_running` 等
- `service/brookesia_service_manager/include/brookesia/service_manager/rpc/server.hpp` — `rpc::Server`（`Config{listen_port,max_connections}`）
- `service/brookesia_service_manager/include/brookesia/service_manager/rpc/client.hpp` — `rpc::Client`（`connect`/`call_function_sync`/`subscribe_event`/`unsubscribe_events`）
- `utils/brookesia_lib_utils/include/brookesia/lib_utils/{memory,thread,time}_profiler.hpp` — 三类剖析器
- `docs/en/service/manager/rpc.rst` — RPC API 参考（protocol / connection / data_link / server / client）
