---
name: esp-brookesia-skill
description: >-
  AI Skill for ESP-Brookesia HMI/AIoT application development on ESP-IDF. Used when users need to
  create, modify, or debug ESP-Brookesia projects, including the service framework (Wi-Fi, Audio,
  NVS, SNTP, Device, Video, Custom), the AI Agent framework (XiaoZhi, Coze, OpenAI), the HAL
  adaptor/boards layer, and the expression (emote) module.
  Trigger words: "ESP-Brookesia", "Brookesia", "esp-brookesia", "brookesia_service_*", "brookesia_agent_*", "brookesia_hal_*", "brookesia_expression_emote", "ServiceManager", "service_helper", "agent_helper", "HMI", "AIoT", "XiaoZhi", "小智", "人机交互", "乐鑫", "Espressif", "MCP"
license: Apache-2.0
metadata:
  author: Community
  version: "1.1.0"
---

# esp-brookesia-skill

ESP-Brookesia 人机交互（HMI/AIoT）应用开发 AI Skill。本 Skill 完全基于 Espressif 官方仓库
`esp-brookesia` 的真实文档（`docs/`）与源码（`examples/`、各组件 `include/`、`Kconfig`、`idf_component.yml`）编写，
提供场景化配方（recipes）、API 速查（resources）、配置速查、踩坑清单与执行流程，帮助 AI 代理在 ESP-IDF 上正确使用
ESP-Brookesia 的服务框架、AI Agent 框架、HAL 板级适配层与表情（emote）模块。

ESP-Brookesia 是一套面向 AIoT 设备的人机交互开发框架，自 **v0.7** 起组件化：组件通过 ESP Component Registry 独立分发，
共享同一 `major.minor` 版本并依赖同一 ESP-IDF 版本。所有服务/Agent 均基于 **Manager + Helper（CRTP）** 架构，
应用层只通过 `ServiceManager` 单例与各类 Helper（如 `WifiHelper = service::helper::Wifi`）调用函数（function）与订阅事件（event）。

## Core Principles

1. **绝不臆造 API** — 任何函数名、结构体、枚举、宏、Kconfig 符号、文件路径必须可在 `resources/` 或仓库 `docs/`/源码中查到；查不到就直接省略，绝不编造。
2. **先 `ServiceManager::start()`，再 `bind()`** — 调用任何服务 API 之前必须 `service::ServiceManager::get_instance().start()`，再 `service_manager.bind(Helper::get_name().data())` 启动服务及其依赖。`binding` 存活期间服务运行，析构即停止（RAII）。
3. **Helper 优先于直接调用** — 应用代码通过 `service::helper::*`（或 `agent::helper::*`）的静态方法 `call_function_sync` / `call_function_async` / `subscribe_event` 访问服务能力，而不是直接操作底层组件。即使未链接具体服务实现，Helper 代码仍可编译（见 `is_available()` 运行期检查）。
4. **参数类型只有六种** — Function/Event schema 参数类型：`String`(std::string)、`Number`(算术类型)、`Boolean`(bool)、`Object`(boost::json::object，常由 struct 序列化)、`Array`(boost::json::array)、`RawBuffer`(service::RawBuffer)。调用时类型/数量/顺序必须与 schema 一致。
5. **CRTP 序列化用 `BROOKESIA_DESCRIBE_*` 宏** — 序列化 struct 到 JSON 用 `BROOKESIA_DESCRIBE_TO_JSON(x).as_object()/.as_array()`；枚举转字符串用 `BROOKESIA_DESCRIBE_TO_STR(Enum::Val)`；JSON 反序列化用 `BROOKESIA_DESCRIBE_FROM_JSON`，字符串转枚举用 `BROOKESIA_DESCRIBE_STR_TO_ENUM`。自定义 struct/enum 必须用 `BROOKESIA_DESCRIBE_STRUCT` / `BROOKESIA_DESCRIBE_ENUM` 描述。
6. **同步调用阻塞、异步调用立即返回** — `call_function_sync<RetT>(FunctionId::X, args..., [Timeout(ms)])` 阻塞至完成或超时；`call_function_async(FunctionId::X, args..., [handler])` 立即返回，结果在 handler 中异步回调。同一服务的连续 async 调用按提交顺序串行执行。
7. **事件订阅是 RAII 连接** — `subscribe_event` 返回的 `connection` 析构即取消订阅；需要长期保活请存入 `static`/堆/`std::vector<service::EventRegistry::SignalConnection>`。`subscribe_event` 必须在 `ServiceManager::start()` 之后调用。
8. **EventMonitor 提供阻塞等待** — `Helper::EventMonitor<Helper::EventId::X>` 在异步事件基础上叠加阻塞等待：`start()` 后用 `wait_for(items, timeout_ms)`（匹配指定 item）或 `wait_for_any(timeout_ms)`（任意一次），再 `get_last<T>()` 取最近事件载荷。
9. **Agent 是状态机** — `AgentManager` 用状态机管理生命周期：`TimeSyncing→Ready→Activating→Activated→Starting→Started→Sleeping→Slept→WakingUp→(回到 Started)→Stopping→Ready`。Stable 状态：Ready/Activated/Started/Slept。通过 `TriggerGeneralAction` 驱动（Init/Activate/Start/Stop/Sleep/WakeUp/TimeSync）。
10. **HAL 先于服务初始化** — 需要 HAL 设备的服务（Audio/Device/Agent 等）必须在 `ServiceManager::start()` 之前调用 `hal::init_device(...)` 或 `hal::init_all_devices()`。纯 Wi-Fi/NVS 等服务通常不需要 HAL。
11. **资源/内存门槛** — 框架要求 Flash ≥ 8MB、PSRAM ≥ 4MB；AI Agent 类示例（chatbot）要求 Flash ≥ 16MB、PSRAM ≥ 8MB。SPI LCD 的总线初始化与数据传输必须在同一 CPU 核完成（多核时用 `BROOKESIA_THREAD_CONFIG_GUARD` 锁核）。
12. **日志与检查宏来自 lib_utils** — `#define BROOKESIA_LOG_TAG "X"` 后用 `BROOKESIA_LOGI/E/W/D`（boost::format 风格 `%1%`）；错误检查用 `BROOKESIA_CHECK_FALSE_EXIT/RETURN`、`BROOKESIA_CHECK_NULL_RETURN`、`BROOKESIA_CHECK_OUT_RANGE_RETURN`、`BROOKESIA_CHECK_EXCEPTION_EXIT`。

## When to Use

**Applicable:**
- 创建新的 ESP-Brookesia / ESP-IDF HMI 应用项目
- 使用服务框架调用 Wi-Fi / NVS / SNTP / Audio / Device / Video / Custom 服务
- 构建 AI 语音助手（XiaoZhi / Coze / OpenAI Agent）并管理其生命周期
- 适配或新增开发板（HAL boards YAML 配置）
- 使用 Emote 表情模块做拟人化视觉反馈
- 通过 MCP/Function Calling 让 LLM 调用设备能力
- 排查服务绑定、事件订阅、状态机、RAII 连接等典型问题

**Not applicable:**
- 与 ESP-Brookesia 无关的纯 ESP-IDF 驱动/外设问题（应参考 ESP-IDF 编程指南）
- 非 ESP32 系列 MCU 平台（STM32、CH57x 等）
- PCB 硬件设计与原理图绘制
- 旧版（v0.6 及以下、非组件化）ESP-Brookesia 的 LVGL screen 方案（本 Skill 聚焦 v0.7 组件化框架）

---

## Scenario Quick Reference (Recipes)

当用户意图命中下列场景时，**先读对应 recipe** — 它包含完整调用链、分步说明、常见错误与代码示例。

### 项目与构建

| recipe | scenario |
|---|---|
| `recipes/project_setup.md` | 新建 ESP-Brookesia 项目：依赖声明、板级/芯片选择、构建烧录流程 |
| `recipes/service_framework_basics.md` | 服务框架通用范式：ServiceManager 启动、bind、同步/异步调用、事件订阅、EventMonitor |

### 通用服务

| recipe | scenario |
|---|---|
| `recipes/wifi_service.md` | Wi-Fi 服务：扫描、连接/断开、SoftAP 配网、状态查询、事件订阅 |
| `recipes/nvs_service.md` | NVS 服务：类型安全 `save_key_value`/`get_key_value` 与通用 JSON `Set/Get/List/Erase` |
| `recipes/audio_service.md` | Audio 服务：播放 URL/多 URL、播放控制（暂停/恢复/停止）、编解码回环、AFE 唤醒/VAD |
| `recipes/device_service.md` | Device 服务：能力查询、背光/音量控制、存储/电池查询、事件监听 |
| `recipes/sntp_service.md` | SNTP 服务：NTP 服务器、时区、同步状态查询 |
| `recipes/custom_service.md` | Custom 服务：运行期动态注册 function/event |
| `recipes/service_console_rpc_debug.md` | 服务控制台：串口 CLI（`svc_call`/`svc_subscribe`）、跨设备 RPC（`svc_rpc_server`/`svc_rpc_call`）、内存/线程/时间剖析器（`debug_mem`/`debug_thread`/`debug_time_report`） |

### AI Agent 与表��

| recipe | scenario |
|---|---|
| `recipes/agent_chatbot.md` | 完整 AI 语音助手：AgentManager 绑定、多 Agent 初始化、MCP 工具、状态机驱动 |
| `recipes/expression_emote.md` | Emote 表情模块：加载资源、SetEmoji、事件消息、二维码、动画插入 |

### HAL 与板级

| recipe | scenario |
|---|---|
| `recipes/hal_boards.md` | HAL 板级适配：`init_all_devices`、接口查询、新增自定义板 |

---

## Component & Board Support

### 组件分层（来自 `docs/en/index.rst`）

| 层 | 组件（真实组件名，发布于 ESP Registry） |
|---|---|
| Utils | `brookesia_lib_utils`、`brookesia_mcp_utils` |
| HAL | `brookesia_hal_interface`、`brookesia_hal_adaptor`、`brookesia_hal_boards` |
| Service 框架 | `brookesia_service_manager`、`brookesia_service_helper` |
| 通用服务 | `brookesia_service_wifi`、`brookesia_service_nvs`、`brookesia_service_sntp`、`brookesia_service_audio`、`brookesia_service_device`、`brookesia_service_video`、`brookesia_service_custom` |
| AI Agent | `brookesia_agent_manager`、`brookesia_agent_helper`、`brookesia_agent_coze`、`brookesia_agent_openai`、`brookesia_agent_xiaozhi` |
| 表情 | `brookesia_expression_emote` |

### 受支持开发板（来自 `docs/en/hal/boards/espressif.rst`，Espressif 板）

| 板名 (`idf.py gen-bmgr-config -b <board>`) | 芯片 | Flash | PSRAM |
|---|---|---|---|
| `esp32_s3_korvo2_v3` | ESP32-S3 | 16MB | 8MB |
| `esp_box_3` | ESP32-S3 | 16MB | 16MB |
| `esp_vocat_board_v1_0` | ESP32-S3 | 32MB | 16MB |
| `esp_vocat_board_v1_2` | ESP32-S3 | 32MB | 16MB |
| `esp32_p4_function_ev` | ESP32-P4 | 16MB | 32MB |
| `esp32_p4x_function_ev` | ESP32-P4 | 16MB | 32MB |
| `esp32_s31_korvo1` | ESP32-S31 | 16MB | 16MB |
| `esp_sensair_shuttle` | ESP32-C5 | 16MB | 8MB |

> 另有 Waveshare 板（`esp32_s3_touch_amoled_1_75c` / `esp32_s3_touch_amoled_1_8` / `esp32_s3_touch_amoled_2_16`）与 rymcu 板（`rymcu_bigsmart`），位于 `hal/brookesia_hal_boards/boards/<vendor>/`。各板支持的 HAL 接口（AudioCodecPlayer/Recorder、DisplayPanel/Touch/Backlight、StorageFs 等）见 `docs/en/hal/boards/espressif.rst`。

### 版本与依赖（来自 `docs/en/getting_started.rst`）

| ESP-Brookesia | ESP-IDF | 状态 |
|---|---|---|
| master (v0.7) | >= v5.5 | 组件化，活跃开发 |
| release/v0.6 | >= v5.3, <= 5.5 | 预览系统框架，已停止维护 |

---

## Service Framework State & Call Model

### 服务调用统一范式（来自 `docs/en/service/usage.rst`）

```
[dependencies in idf_component.yml]
  → #include "brookesia/service_manager.hpp" + helper headers
  → ServiceManager::get_instance().start()
  → Helper::is_available()        // 运行期可用性检查
  → service_manager.bind(Helper::get_name().data())   // RAII binding
  → Helper::call_function_sync/async(FunctionId::X, args...)   // 调用函数
  → Helper::subscribe_event(EventId::Y, handler)              // RAII 订阅事件
  → Helper::EventMonitor<EventId::Y>{}.wait_for(items, ms)    // 阻塞等待
```

### 关键返回类型

```cpp
// call_function_sync 返回 expected<RetT, Error>
auto result = WifiHelper::call_function_sync<RetT>(WifiHelper::FunctionId::X, args..., service::helper::Timeout(100));
if (!result) { BROOKESIA_LOGE("Failed: %1%", result.error()); }
else { auto &v = result.value(); }

// call_function_async 返回 expected<void, Error>，结果走 handler
auto on_done = [](service::FunctionResult &&result) {
    if (!result.success) { BROOKESIA_LOGE("%1%", result.error_message); }
    else { /* auto &v = result.get_data<RetT>(); */ }
};
Helper::call_function_async(Helper::FunctionId::X, args..., on_done);
```

---

## Critical Pitfalls (Must Read)

### 1. 必须先 start()，再 bind()，最后才能调用 Helper

```cpp
// ❌ WRONG — 未 start ServiceManager 直接 bind/调用
auto binding = service_manager.bind(WifiHelper::get_name().data());
WifiHelper::call_function_sync(WifiHelper::FunctionId::TriggerScanStart);

// ✅ CORRECT
auto &service_manager = service::ServiceManager::get_instance();
service_manager.init();                      // 部分示例显式调用
service_manager.start();
auto binding = service_manager.bind(WifiHelper::get_name().data());
if (!binding.is_valid()) { /* 启动失败 */ }
// 之后才可调用 Helper
WifiHelper::call_function_sync(WifiHelper::FunctionId::TriggerScanStart);
```

### 2. binding/connection 是 RAII，栈上对象析构会立即停止服务/取消订阅

```cpp
// ❌ WRONG — binding 在 if 块结束时析构，服务随即停止
{
    auto binding = service_manager.bind(AudioHelper::get_name().data());
    AudioHelper::call_function_sync(AudioHelper::FunctionId::PlayUrl, "1.mp3");
}   // ← binding 析构，Audio 服务停止，后续调用失败

// ✅ CORRECT — 把 binding 存到足够长的生命周期（成员/容器/全局）
// 方式 A：存到 vector（chatbot 示例做法）
std::vector<service::ServiceManager::ServiceBinding> service_bindings_;
service_bindings_.push_back(std::move(binding));

// 方式 B：在 app_main 作用域内保持到函数末尾
int app_main_body() {
    auto binding = service_manager.bind(...);   // 活到函数返回
    ...
}
```

### 3. 同步调用要带 Timeout，且参数顺序/类型必须匹配 schema

```cpp
// ❌ WRONG — 参数顺序反了；缺 Timeout 导致用默认超时
WifiHelper::call_function_sync(WifiHelper::FunctionId::SetConnectAp, "password", "ssid");

// ✅ CORRECT — schema 顺序 SSID(String) -> Password(String)，可选 service::helper::Timeout(ms)
auto r = WifiHelper::call_function_sync(
    WifiHelper::FunctionId::SetConnectAp, "ssid1", "password1", service::helper::Timeout(100));
if (!r) { BROOKESIA_LOGE("Failed: %1%", r.error()); }
```

### 4. 返回 Array/Object 要用正确模板参数并反序列化

```cpp
// ❌ WRONG — 用默认 void 模板取不到返回值
WifiHelper::call_function_sync(WifiHelper::FunctionId::GetConnectedAps);

// ✅ CORRECT — 明确 boost::json::array，再用 BROOKESIA_DESCRIBE_FROM_JSON 反序列化
auto res = WifiHelper::call_function_sync<boost::json::array>(WifiHelper::FunctionId::GetConnectedAps);
if (res) {
    std::vector<WifiHelper::ConnectApInfo> infos;
    BROOKESIA_DESCRIBE_FROM_JSON(res.value(), infos);
}
```

### 5. 枚举/struct 必须用 BROOKESIA_DESCRIBE 描述，并通过宏转字符串/JSON

```cpp
// ❌ WRONG — 直接把 enum 当参数传入；schema 要 String
WifiHelper::call_function_sync(WifiHelper::FunctionId::TriggerGeneralAction, WifiHelper::GeneralAction::Start);

// ✅ CORRECT — 枚举转字符串后再传
WifiHelper::call_function_sync(
    WifiHelper::FunctionId::TriggerGeneralAction,
    BROOKESIA_DESCRIBE_TO_STR(WifiHelper::GeneralAction::Start));

// 自定义 struct 必须先描述才能序列化
struct Point { int x; int y; };
BROOKESIA_DESCRIBE_STRUCT(Point, (), (x, y))   // 缺这一行则无法 BROOKESIA_DESCRIBE_TO_JSON
```

### 6. 需要 HAL 设备的服务必须先初始化 HAL

```cpp
// ❌ WRONG — Audio/Device 服务依赖 HAL 设备，未初始化直接 bind 会失败
service_manager.start();
auto binding = service_manager.bind(AudioHelper::get_name().data());

// ✅ CORRECT — 先 init_device，再 start ServiceManager（Audio 示例做法）
#include "brookesia/hal_interface.hpp"
#include "brookesia/hal_adaptor.hpp"
hal::init_device(hal::StorageDevice::DEVICE_NAME);
hal::init_device(hal::AudioDevice::DEVICE_NAME);
service_manager.start();
auto binding = service_manager.bind(AudioHelper::get_name().data());
// 或一次性初始化全部：hal::init_all_devices();
```

### 7. EventMonitor 用完要 stop()/clear()，且 item 类型/顺序须匹配事件 schema

```cpp
// ❌ WRONG — wait_for 的 item 用错类型（事件是 String+Boolean，却传 Number）
WifiGeneralEventMonitor m; m.start();
m.wait_for(std::vector<service::EventItem>{ 123 }, 5000);   // 类型不匹配，永远等不到

// ✅ CORRECT — GeneralEventHappened schema: Event(String), IsUnexpected(Boolean)
WifiHelper::EventMonitor<WifiHelper::EventId::GeneralEventHappened> m;
m.start();
bool got = m.wait_for(std::vector<service::EventItem>{
    BROOKESIA_DESCRIBE_TO_STR(WifiHelper::GeneralEvent::Started), false
}, WIFI_WAIT_START_TIMEOUT_MS);
m.stop();
m.clear();
```

### 8. subscribe_event 必须在 start() 之后，且回调签名要匹配 item 列表

```cpp
// ❌ WRONG — 在 ServiceManager::start() 之前订阅；回调参数数量与 schema 不符
auto conn = WifiHelper::subscribe_event(
    WifiHelper::EventId::ScanApInfosUpdated,
    [](const std::string &, const std::string &) { /* 错：应是 array */ });
service_manager.start();

// ✅ CORRECT — 先 start，回调签名匹配 ScanApInfosUpdated 的 item: ApInfos(Array)
service_manager.start();
auto conn = WifiHelper::subscribe_event(
    WifiHelper::EventId::ScanApInfosUpdated,
    [](const std::string & event_name, const boost::json::array & ap_infos) { /* ... */ });
```

### 9. Agent 启动顺序：Activate → Start，停止用 Stop/Deinit；状态由状态机保证

```cpp
// ❌ WRONG — 直接 Start 而 未 Activate；或在未连 Wi-Fi 时启动 Agent
AgentHelper::call_function_sync(AgentHelper::FunctionId::TriggerGeneralAction,
                                BROOKESIA_DESCRIBE_TO_STR(AgentHelper::GeneralAction::Start));

// ✅ CORRECT — 先 Activate 再 Start；chatbot 在 Wi-Fi Connected 后才 start_agent()
AgentHelper::call_function_async(AgentHelper::FunctionId::TriggerGeneralAction,
    BROOKESIA_DESCRIBE_TO_STR(AgentHelper::GeneralAction::Activate), on_activate);
AgentHelper::call_function_async(AgentHelper::FunctionId::TriggerGeneralAction,
    BROOKESIA_DESCRIBE_TO_STR(AgentHelper::GeneralAction::Start), on_start);
```

### 10. 多核 SPI LCD 必须把总线初始化锁在同一核

```cpp
// ❌ WRONG — SPI LCD 的 init 与 draw 在不同核上执行，可能崩溃
hal::init_device(hal::DisplayDevice::DEVICE_NAME);

// ✅ CORRECT — 用 BROOKESIA_THREAD_CONFIG_GUARD 锁核（chatbot 示例做法）
#if CONFIG_SOC_CPU_CORES_NUM > 1
{
    BROOKESIA_THREAD_CONFIG_GUARD({ .core_id = 1 });
    std::thread([&]() { hal::init_device(hal::DisplayDevice::DEVICE_NAME); }).join();
}
#endif
hal::init_all_devices();
```

### 11. idf_component.yml 依赖名要写全 espressif/ 前缀

```yaml
# ❌ WRONG — 缺前缀，组件管理器找不到
dependencies:
  brookesia_service_wifi: "*"

# ✅ CORRECT
dependencies:
  espressif/brookesia_service_wifi: "*"
  espressif/brookesia_service_nvs: "*"   # 可选
```

### 12. 板级工程用 gen-bmgr-config 选板，而不是 set-target

```bash
# ❌ WRONG — 依赖板级外设（Audio/LCD）的示例用 set-target 会缺少板配置
idf.py set-target esp32s3

# ✅ CORRECT — 板级工程（如 examples/service/console）用 gen-bmgr-config
idf.py gen-bmgr-config -b esp_vocat_board_v1_2
idf.py build
# 纯芯片工程（如 examples/service/wifi）才用 set-target
idf.py set-target esp32s3
```

---

## Execution Workflow

| Step | Name | Description |
|------|------|-------------|
| 1 | Plan | 明确目标：用哪个服务/Agent？需要哪些 HAL 设备？目标芯片或板？ |
| 2 | Recipe | 在 `recipes/` 找到最贴近的场景，照其调用链实现 |
| 3 | Query | recipe 未覆盖的 API，查 `resources/api_reference.md`；配置项查 `resources/config_reference.md` |
| 4 | Validate | 核对：依赖名（`espressif/...`）、include 头、`start()`→`bind()`→调用顺序、参数 schema、RAII 生命周期 |
| 5 | Confirm | 向用户给出实现计划：依赖、includes、init 顺序、调用与事件订阅、HAL 依赖 |
| 6 | Execute | **新项目：** 复制最接近的 `examples/` 目录改造（见下）。**已有项目：** 原地编辑 |
| 7 | Check | 检查 ServiceManager 启动顺序、binding/connection 生命周期、HAL 设备已初始化、多核锁核 |
| 8 | Build | `idf.py gen-bmgr-config -b <board>`（板级）或 `idf.py set-target <chip>`（纯芯片）→ `idf.py build` → `idf.py -p <PORT> flash` |
| 9 | Monitor | `idf.py -p <PORT> monitor`；通过日志宏 `BROOKESIA_LOGI`（TAG 见 `#define BROOKESIA_LOG_TAG`）观察状态机与事件 |

### Step 6 Detail ��� 示例选择策略

依据用户需求选最贴近的官方示例作为起点（路径相对仓库根 `examples/`）：

- Wi-Fi 服务 → `examples/service/wifi`（扫描/连接/SoftAP 配网，纯芯片工程，最完整的服务范式参考）
- NVS 服务 → `examples/service/nvs`（类型安全 + 通用 JSON 两套用法）
- Audio 服务 → `examples/service/audio`（播放/控制/编解码/AFE，板级工程）
- Device 服务 → `examples/service/device`（能力查询/背光/音量/存储/电池）
- Service 控制台（命令行调试） → `examples/service/console`（含 `svc_call`/`svc_subscribe` 命令，可在线驱动服务/Agent）
- 完整 AI 语音助手 → `examples/agent/chatbot`（XiaoZhi/Coze/OpenAI 多 Agent、MCP 工具、Emote、Wi-Fi 配网、UI）

复制整个示例目录（含 `main/idf_component.yml`）到目标工程，再按需改 `idf_component.yml`、menuconfig、main 逻辑。**不要原地修改仓库示例。**

---

## Failure Strategies

| Situation | Action |
|---|---|
| API/符号在 `resources/` 与仓库查不到 | 立即停止，告知用户该符号不存在，不要臆造 |
| `binding.is_valid()` 为 false | 检查：是否先 `ServiceManager::start()`；HAL 设备是否已 init；依赖组件是否加入 `idf_component.yml` |
| `is_available()` 为 false | 该服务组件未链接进构建；确认 `idf_component.yml` 依赖了对应 `espressif/brookesia_service_*` |
| 同步调用超时/失败 | 检查参数类型与顺序（schema 六类型）；服务是否已 `bind`；是否需要 Task Scheduler（标 Required 的函数须在服务启动后调用） |
| 事件收不到 | `subscribe_event` 是否在 `start()` 之后；回调签名 item 数量/类型是否匹配 schema；connection 是否过早析构 |
| Agent 不响应 | Wi-Fi 是否已连接（Agent 依赖网络）；menuconfig 是否启用并填了 API Key/Bot ID；模型分区是否烧入唤醒词模型 |
| 板级工程编译缺组件 | 用 `idf.py gen-bmgr-config -b <board>`（工程内含 `idf_ext.py` 会自动拉取 `esp_board_manager` 与 `brookesia_hal_boards`） |
| SPI LCD 偶发崩溃 | 多核 SoC 上用 `BROOKESIA_THREAD_CONFIG_GUARD` 把总线 init/draw 锁同一核 |

## References

- 场景配方 → `recipes/` 目录
- 服务/Agent/HAL API 速查 → `resources/api_reference.md`
- Kconfig 配置项速查 → `resources/config_reference.md`
- 踩坑汇总 → `resources/pitfalls.md`
- 真实示例清单 → `resources/example_list.md`
- 官方在线文档（英文）：https://docs.espressif.com/projects/esp-brookesia/en
- 官方在线文档（中文）：https://docs.espressif.com/projects/esp-brookesia/zh_CN
- ESP-IDF 编程指南：https://docs.espressif.com/projects/esp-idf/en
- 组件仓库：https://components.espressif.com/
