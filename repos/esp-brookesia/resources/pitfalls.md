# ESP-Brookesia 踩坑汇总

> SKILL.md 的 “Critical Pitfalls” 的展开版。每条给出错误现象、根因、正确写法（均来自仓库示例与文档）。

## A. 服务框架生命周期

### A1. 必须先 `ServiceManager::start()` 再 `bind()` 再调用

- 根因：服务在 bind 时才被启动；未 start 的 manager 无法 bind。
- 正解：`get_instance()` → `init()`（部分示例显式调用）→ `start()` → `bind()` → Helper 调用。
- 来源：`examples/service/wifi/main/main.cpp`

### A2. `binding` / `connection` 是 RAII，栈对象过早析构

- 现象：函数返回后服务停止、事件回调不再触发。
- 根因：`ServiceBinding`、`SignalConnection` 析构即停止服务/取消订阅。
- 正解：存入成员、`static`、堆，或 `std::vector<service::ServiceManager::ServiceBinding>` / `std::vector<service::EventRegistry::SignalConnection>`（chatbot 做法）。
- 来源：`examples/agent/chatbot/main/modules/ai_agents.cpp`

### A3. `is_available()` 为 false 仍编译通过

- 现象：Helper 代码能编译，但运行期 `is_available()` 返回 false。
- 根因：Helper 头基于 CRTP，即使未链接具体服务实现也可编译；运行期才检查。
- 正解：调用前先 `if (!Helper::is_available()) return;`；并确认 `idf_component.yml` 依赖了对应组件。
- 来源：`docs/en/service/usage.rst`

## B. 函数调用与参数

### B1. 参数类型必须匹配六类 schema

- 六类型：`String`(std::string) / `Number`(算术) / `Boolean`(bool) / `Object`(boost::json::object) / `Array`(boost::json::array) / `RawBuffer`(service::RawBuffer)。
- 易错：把枚举直接当参数（应 `BROOKESIA_DESCRIBE_TO_STR(Enum::Val)`）；把 struct 直接传（应 `BROOKESIA_DESCRIBE_TO_JSON(x).as_object()`）。

### B2. 返回值要显式模板参数

- 错：`call_function_sync(FunctionId::GetConnectedAps)`（取不到值）。
- 对：`call_function_sync<boost::json::array>(FunctionId::GetConnectedAps)`，再 `BROOKESIA_DESCRIBE_FROM_JSON` 反序列化。

### B3. 需要 Task Scheduler 的函数须在服务启动后调用

- 文档把函数标注 “Execution requirement: Required/Not required”。
- Required：须在服务 start 之后；sync/async 都走调度器。
- Not required：可在启动前调用；仅 sync；典型场景：含 `RawBuffer`、实现自身已线程安全、或必须在启动前运行（如 `FeedDecoderData`）。
- 正解：`FeedDecoderData` 不要传 `Timeout`。
- 来源：`docs/en/service/usage.rst`、`examples/service/audio`

### B4. async 串行但 sync 阻塞

- 同一服务连续 `call_function_async` 按提交顺序执行；`call_function_sync` 阻塞当前线程。勿在持有锁时同步调用，避免死锁。

## C. 事件与 EventMonitor

### C1. `subscribe_event` 必须在 `start()` 之后

- 否则订阅失败（`connected()` 为 false）。

### C2. 回调签名（item 数量/类型）必须匹配 schema

- 例：`ScanApInfosUpdated` 的回调应是 `(const std::string&, const boost::json::array&)`，而非两个 string。

### C3. EventMonitor 的 `wait_for` item 类型要对

- `GeneralEventHappened`：`(String, Boolean)`。传 `Number` 永远等不到。
- `DisplayBacklightBrightnessChanged`：`(double)`。
- 用 `get_last<T>()` 取最近值：`std::get<0>(last.value())`。

### C4. monitor 用完 `stop()` / `clear()`

- 复用前 `clear()`，否则历史事件干扰下一次 `wait_for`。

## D. HAL 与板级

### D1. 依赖 HAL 的服务必须先 `init_device`

- Audio/Device/Agent：在 `ServiceManager::start()` 前调 `hal::init_device(StorageDevice)` / `AudioDevice` / `DisplayDevice`，或 `hal::init_all_devices()`。
- 来源：`examples/service/audio`、`examples/service/device`

### D2. 多核 SPI LCD 锁核

- 现象：SPI LCD 总线 init/draw 跨核导致崩溃。
- 正解：`BROOKESIA_THREAD_CONFIG_GUARD({ .core_id = 1 })` 把 Display 初始��包起来。
- 来源：`examples/agent/chatbot/main/main.cpp`

### D3. 板级工程用 `gen-bmgr-config` 而非 `set-target`

- 板级示例含 `idf_ext.py`，会自动拉取 `esp_board_manager` 与 `brookesia_hal_boards`；用 `set-target` 会缺板配置。

### D4. VSCode 装的 ESP-IDF 可能构建失败

- 官方建议命令行安装 ESP-IDF（部分依赖 `esp_board_manager` 的示例对扩展环境不友好）。
- 来源：`docs/en/getting_started.rst`

## E. Agent 状态机

### E1. 启动顺序 Activate → Start

- 错：直接 Start。
- 对：先 `TriggerGeneralAction(Activate)`，再 `(Start)`；chatbot 在 Wi-Fi Connected 事件里 `start_agent()`。
- 状态机 Stable：Ready/Activated/Started/Slept；Transient：TimeSyncing/Activating/Starting/Sleeping/WakingUp/Stopping。

### E2. 异常 Stopped 自动重启

- 监听 `GeneralEventHappened` 的 `is_unexpected` Stopped；若 Wi-Fi 已连，用 `TaskScheduler::post_delayed` 延迟重新 Start。

### E3. Agent 依赖 SNTP 与 Wi-Fi

- `TimeSyncing` 状态需 SNTP；未连 Wi-Fi 不应启动 Agent（chatbot 在 Disconnected 时 `stop_agent()`）。

### E4. 唤醒词模型须烧入 `model` 分区

- AFE `WakeNetConfig.model_partition_label = "model"`；未烧模型则唤醒词无反应。
- 唤醒词默认 `"Hi,ESP"`，语言 `mn_language` 可设 `"cn"`。

## F. 内存与构建

### F1. 资源门槛

- 框架：Flash ≥ 8MB、PSRAM ≥ 4MB。
- Agent 示例（chatbot）：Flash ≥ 16MB、PSRAM ≥ 8MB。

### F2. PSRAM 但未开 XIP

- 现象：任务操作 Flash 导致崩溃。
- 正解：`player_task.stack_in_ext = true`（`CONFIG_SPIRAM_XIP_FROM_PSRAM` 时）。

### F3. 依赖名缺 `espressif/` 前缀

- `idf_component.yml` 必须写 `espressif/brookesia_service_wifi`，否则组件管理器找不到。

### F4. 自定义 struct/enum 未描述

- 缺 `BROOKESIA_DESCRIBE_STRUCT` / `BROOKESIA_DESCRIBE_ENUM` 则 `BROOKESIA_DESCRIBE_TO_JSON` / 序列化失败。
- 来源��`examples/service/nvs`

## G. MCP / LLM 工具

### G1. 按设备能力动态添加 MCP 工具

- 错：硬编码所有 Device 函数为工具。
- 对：先 `GetCapabilities`，再按 `AudioCodecPlayerIface::NAME` / `DisplayBacklightIface::NAME` / `StorageFsIface::NAME` / `PowerBatteryIface::NAME` 是否存在决定加入哪些 `DeviceHelper::FunctionId`。
- 来源：`examples/agent/chatbot/main/modules/ai_agents.cpp`

### G2. Emote 名字必须存在于资源集

- 例如 chatbot 中无 “cool” 表情，故替换为 “winking”；否则 `SetEmoji` 无效。
