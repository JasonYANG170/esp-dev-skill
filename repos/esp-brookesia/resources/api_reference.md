# ESP-Brookesia API 速查

> 所有签名、枚举、宏、路径均来自仓库 `docs/en/` 与 `examples/`、各组件 `include/`。未在仓库中查到的 API 一律未列出。命名空间为 `esp_brookesia`（service/agent/lib_utils/hal 子空间）。

## 通用调用与事件（service::helper::Base，CRTP 基类）

头：`brookesia/service_helper/base.hpp`（来自 `docs/en/service/helper/base.rst`）

```cpp
namespace esp_brookesia::service::helper {

template <typename Derived>
class Base {
public:
    // 可用性：对应服务组件是否链接进构建
    static bool is_available();

    // 同步调用：阻塞至完成或超时。RetT 省略表示 void
    template <typename RetT = void, typename... Args>
    static auto call_function_sync(typename Derived::FunctionId id, Args&&... args)
        -> /* expected<RetT, Error> */;

    // 异步调用：立即返回，结果走 handler（可选）
    template <typename... Args>
    static auto call_function_async(typename Derived::FunctionId id, Args&&... args)
        -> /* expected<void, Error> */;

    // 订阅事件：返回 RAII SignalConnection，析构取消订阅
    template <typename Handler>
    static auto subscribe_event(typename Derived::EventId id, Handler && handler)
        -> /* SignalConnection */;

    // schema 查询
    static auto get_function_schema(...) /* -> const FunctionSchema* */;
    static auto get_event_schema(...)    /* -> const EventSchema* */;
};

// 超时包装
struct Timeout { explicit Timeout(uint32_t ms); };

} // namespace
```

### 调用模板

```cpp
// 同步（带返回）
auto r = Helper::call_function_sync<RetT>(Helper::FunctionId::X, arg1, arg2, service::helper::Timeout(100));
if (!r) { BROOKESIA_LOGE("%1%", r.error()); }
else { auto &v = r.value(); }

// 异步（handler 可选）
auto on_done = [](service::FunctionResult &&result) {
    if (!result.success) { BROOKESIA_LOGE("%1%", result.error_message); }
    else { auto &v = result.get_data<RetT>(); }
};
Helper::call_function_async(Helper::FunctionId::X, arg1, on_done);
```

### EventMonitor（阻塞等待）

```cpp
Helper::EventMonitor<Helper::EventId::X> monitor;
monitor.start();
bool any = monitor.wait_for_any(timeout_ms);
bool got = monitor.wait_for(std::vector<service::EventItem>{ items... }, timeout_ms);
auto last = monitor.get_last<T>();   // std::optional<std::tuple<T>>；用 std::get<0>(last.value())
monitor.stop();
monitor.clear();
```

## 序列化宏（lib_utils / describe_helpers）

头：`brookesia/lib_utils.hpp`（来自 `docs/en/utils/lib_utils/describe_helpers.rst`）

```cpp
// struct 描述：(flags, fields)  —— fields 为字段名元组
BROOKESIA_DESCRIBE_STRUCT(StructName, (), (field1, field2, ...))

// enum 描述
BROOKESIA_DESCRIBE_ENUM(EnumName, Val1, Val2, ...)

// 转换
BROOKESIA_DESCRIBE_TO_JSON(x)               // -> boost::json::value（再 .as_object()/.as_array()）
BROOKESIA_DESCRIBE_FROM_JSON(json_value, x) // -> bool（成功否）
BROOKESIA_DESCRIBE_TO_STR(Enum::Val)        // -> const char*
BROOKESIA_DESCRIBE_STR_TO_ENUM(str, enum_var)
BROOKESIA_DESCRIBE_ENUM_TO_STR(enum_val)
BROOKESIA_DESCRIBE_TO_STR_WITH_FMT(x, FMT)  // FMT: BROOKESIA_DESCRIBE_FORMAT_JSON / VERBOSE
```

## 日志与检查宏（lib_utils）

```cpp
#define BROOKESIA_LOG_TAG "X"
BROOKESIA_LOGI(fmt, ...)   // boost::format 风格 %1% %2%
BROOKESIA_LOGE / BROOKESIA_LOGW / BROOKESIA_LOGD

BROOKESIA_CHECK_FALSE_EXIT(expr, fmt, ...)      // expr 为假则打日志并 return（无返回值函数）
BROOKESIA_CHECK_FALSE_RETURN(expr, ret, fmt, ...) // expr 为假则 return ret
BROOKESIA_CHECK_NULL_RETURN(ptr, ret, fmt, ...)
BROOKESIA_CHECK_OUT_RANGE_RETURN(val, lo, hi, ret, fmt, ...)
BROOKESIA_CHECK_EXCEPTION_EXIT(stmt, fmt, ...)   // 捕获异常并退出

BROOKESIA_LOG_TRACE_GUARD() / BROOKESIA_LOG_TRACE_GUARD_WITH_THIS()

// 多核线程锁核
BROOKESIA_THREAD_CONFIG_GUARD({ .core_id = N });
```

## ServiceManager

头：`brookesia/service_manager.hpp`（来自 `docs/en/service/manager/index.rst`，单例）

```cpp
namespace esp_brookesia::service {

class ServiceManager {
public:
    static ServiceManager & get_instance();
    bool init();
    bool start();
    void stop();

    // 绑定服务（按 Helper::get_name()），返回 RAII ServiceBinding
    ServiceBinding bind(const char * service_name);
};

class ServiceManager::ServiceBinding {
public:
    bool is_valid() const;
    // 析构 -> 服务停止
};

// RAII 清理守卫
namespace lib_utils { class FunctionGuard { FunctionGuard(std::function<void()>); }; }

// 事件订阅返回的连接（RAII）
namespace esp_brookesia::service::EventRegistry { class SignalConnection {
    bool connected() const; void disconnect(); }; }

// 运行期结果（async handler 入参）
struct FunctionResult {
    bool success;
    std::string error_message;
    template <typename T> T & get_data();
};
using EventItem = /* 事件项值，按 schema 顺序放入 vector */;

} // namespace
```

## Wi-Fi Helper（service::helper::Wifi）

头：`brookesia/service_helper/wifi.hpp`（来自 `examples/service/wifi`）

```cpp
enum class FunctionId {
    TriggerGeneralAction,    // 参数 Action:String(GeneralAction 枚举)
    GetGeneralState,         // 返回 String(GeneralState)
    SetConnectAp,            // SSID:String, Password:String
    GetConnectAp,            // 返回 Object(ConnectApInfo)
    GetConnectedAps,         // 返回 Array(ConnectApInfo[])
    SetScanParams,           // Param:Object(ScanParams)
    TriggerScanStart,
    TriggerScanStop,
    SetSoftApParams,         // Param:Object(SoftApParams)
    GetSoftApParams,         // 返回 Object(SoftApParams)
    TriggerSoftApProvisionStart,  // 可选 Timeout
    TriggerSoftApProvisionStop,   // 可选 Timeout
    ResetData,
};

enum class EventId {
    GeneralActionTriggered,  // Action:String
    GeneralEventHappened,    // Event:String, IsUnexpected:Boolean
    ScanStateChanged,        // IsRunning:Boolean
    ScanApInfosUpdated,      // ApInfos:Array(ScanApInfo[])
    SoftApEventHappened,     // Event:String(SoftApEvent)
};

enum class GeneralAction { Init, Start, Stop, Deinit, Connect, Disconnect, TimeSync };
enum class GeneralEvent  { Started, Stopped, Connected, Disconnected, /* ... */ };
enum class GeneralState  { /* ... */ Started, Connected, Connecting, Max };
enum class SoftApEvent   { Started, Stopped };

struct ConnectApInfo { std::string ssid; /* ... */ bool operator==(...) const; };
struct ScanApInfo    { /* ssid/rssi/... */ };
struct ScanParams    { int ap_count; int interval_ms; int timeout_ms; };
struct SoftApParams  { std::string ssid; std::string password; int channel; /* ... */ };
```

## NVS Helper（service::helper::NVS）

头：`brookesia/service_helper/nvs.hpp`（来自 `examples/service/nvs`）

```cpp
using KeyValueMap = std::map<std::string, std::variant<bool, int32_t, std::string /* ... */>>;

enum class FunctionId { Set, Get, List, Erase /*, ... */ };
// Set:   namespace:String, KV:Object(KeyValueMap 序列化)
// Get:   namespace:String, keys:Array(string[]) -> Object
// List:  namespace:String -> Array(EntryInfo[])
// Erase: namespace:String, keys:Array(string[])

struct EntryInfo { std::string key; /* EntryType type */ };

// 类型安全 API（推荐）
template <typename T>
auto save_key_value(const std::string & nspace, const std::string & key, const T & value,
                    uint32_t timeout_ms = default) -> /* expected<void,Error> */;
template <typename T>
auto get_key_value(const std::string & nspace, const std::string & key,
                   uint32_t timeout_ms = default) -> /* expected<T,Error> */;
auto erase_keys(const std::string & nspace,
                const std::vector<std::string> & keys = {}) -> /* expected<void,Error> */;
```

## Audio Helper��service::helper::Audio）

头：`brookesia/service_helper/audio.hpp`（来自 `examples/service/audio`）

```cpp
enum class FunctionId {
    SetPlaybackConfig,        // Param:Object(PlaybackConfig)
    SetEncoderStaticConfig,   // Param:Object(EncoderStaticConfig)
    SetDecoderStaticConfig,   // Param:Object(DecoderStaticConfig)
    PlayUrl,                  // Url:String [, Config:Object(PlayUrlConfig)]
    PlayUrls,                 // Urls:Array(string[]) [, Config:Object(PlayUrlConfig)]
    PlayControl,              // Action:String(PlayControlAction)
    StartEncoder,             // Config:Object(EncoderDynamicConfig) [, Timeout]
    StopEncoder,              // [Timeout]
    StartDecoder,             // Config:Object(DecoderDynamicConfig) [, Timeout]
    StopDecoder,              // [Timeout]
    FeedDecoderData,          // RawBuffer  （不需要 task scheduler，无 Timeout）
    SetAFE_Config,            // Param:Object(AFE_Config)
    GetAFE_WakeWords,         // 返回 Array(string[])
};

enum class EventId {
    PlayStateChanged,      // PlayState:String
    EncoderDataReady,      // Data:RawBuffer
    AFE_EventHappened,     // Event:String(AFE_Event)
};

enum class PlayControlAction { Pause, Resume, Stop };
enum class CodecFormat { PCM, OPUS, G711A /* ... */ };

struct CodecGeneralConfig { int channels; int sample_bits; int sample_rate; int frame_duration; };
struct PlayUrlConfig      { bool interrupt=false; int delay_ms=0; int loop_count=1; int loop_interval_ms=0; int timeout_ms=0; };
struct PlaybackConfig     { struct { int core_id; int priority; int stack_size; bool stack_in_ext; } player_task; };
struct EncoderDynamicConfig  { CodecFormat type; CodecGeneralConfig general; /* extra (Opus...) */ };
struct EncoderExtraConfigOpus{ bool enable_vbr; int bitrate; };
struct DecoderDynamicConfig  { CodecFormat type; CodecGeneralConfig general; };
struct EncoderStaticConfig {};
struct DecoderStaticConfig {};
struct AFE_Config        { AFE_VAD_Config vad; AFE_WakeNetConfig wakenet; };
struct AFE_VAD_Config    {};
struct AFE_WakeNetConfig { const char* model_partition_label; const char* mn_language; int start_timeout_ms; int end_timeout_ms; };

struct RawBuffer { uint8_t * data_ptr; size_t data_size; };  // service::RawBuffer
```

## Device Helper（service::helper::Device）

头：`brookesia/service_helper/device.hpp`（来自 `examples/service/device`）

```cpp
using Capabilities = std::map<std::string, std::vector<std::string>>;  // interface_name -> instances

enum class FunctionId {
    GetCapabilities,                 // -> Object
    GetBoardInfo,                    // -> Object(BoardInfo)
    ResetData,
    // 显示背光
    GetDisplayBacklightBrightness,   // -> double(Number)
    SetDisplayBacklightBrightness,   // Brightness:Number(double)
    GetDisplayBacklightOnOff,        // -> bool(Boolean)
    SetDisplayBacklightOnOff,        // On:Boolean
    // 音频播放器
    GetAudioPlayerVolume,            // -> double(Number)
    SetAudioPlayerVolume,            // Volume:Number(double)
    GetAudioPlayerMute,              // -> bool(Boolean)
    SetAudioPlayerMute,              // Enable:Boolean
    // 存储
    GetStorageFileSystems,           // -> Array
    GetStorageFileSystemCapacity,    // mount_point:String -> Object
    // 电池
    GetPowerBatteryInfo,             // -> Object(PowerBatteryInfo)
    GetPowerBatteryState,            // -> Object(PowerBatteryState)
    GetPowerBatteryChargeConfig,     // -> Object(PowerBatteryChargeConfig)
    SetPowerBatteryChargingEnabled,  // Enable:Boolean
};

enum class EventId {
    DisplayBacklightBrightnessChanged,  // Brightness:Number(double)
    DisplayBacklightOnOffChanged,       // On:Boolean
    AudioPlayerVolumeChanged,           // Volume:Number(double)
    AudioPlayerMuteChanged,             // Enable:Boolean
    PowerBatteryStateChanged,           // State:Object(PowerBatteryState)
    PowerBatteryChargeConfigChanged,    // Config:Object
};

struct BoardInfo {};
struct PowerBatteryInfo { bool has_ability(hal::PowerBatteryIface::Ability) const; };
struct PowerBatteryState { std::optional<int> percentage; hal::PowerBatteryIface::ChargeState charge_state; /* ... */ };
struct PowerBatteryChargeConfig { bool enabled; /* ... */ };
```

## Agent Helpers（agent::helper::*）

### Manager（agent::helper::Manager）

头：`brookesia/agent_helper/manager.hpp`（来自 `docs/en/agent/manager/index.rst`）

```cpp
enum class FunctionId {
    SetAgentInfo,            // Name:String(agent helper name), Info:Object
    SetTargetAgent,          // Name:String
    TriggerGeneralAction,    // Action:String(GeneralAction)
    InterruptSpeaking,
    GetGeneralState,         // -> String(GeneralState)
    /* Sleep/WakeUp/Activate/Start/Stop 走 TriggerGeneralAction */
};

enum class EventId {
    GeneralActionTriggered,  // Action:String
    GeneralEventHappened,    // Event:String, IsUnexpected:Boolean
    SuspendStatusChanged,    // IsSuspended:Boolean
    SpeakingStatusChanged,   // IsSpeaking:Boolean
    ListeningStatusChanged,  // IsListening:Boolean
    AgentSpeakingTextGot,    // Text:String
    UserSpeakingTextGot,     // Text:String
    EmoteGot,                // Emote:String
};

enum class GeneralAction { Activate, Start, Stop, Sleep, WakeUp, TimeSync /*, Init */ };
enum class GeneralEvent  { TimeSynced, Activated, Started, Stopped, Awake /* ... */ };
// Stable 状态: Ready / Activated / Started / Slept
// Transient: TimeSyncing / Activating / Starting / Sleeping / WakingUp / Stopping
```

### XiaoZhi（agent::helper::XiaoZhi）

```cpp
enum class FunctionId {
    AddMCP_ToolsWithServiceFunction,  // ServiceName:String, Functions:Array(FunctionId[]) -> Array
    /* ... */
};
enum class EventId { ActivationCodeReceived /*, CozeEventHappened-style ... */ };
```

### Coze（agent::helper::Coze）

```cpp
struct Info {
    struct Authorization { std::string session_name, device_id, custom_consumer, app_id, user_id, public_key, private_key; } authorization;
    std::vector<struct { std::string name, bot_id, voice_id, description; }> robots;
};
enum class EventId { CozeEventHappened };   // Event:String(CozeEvent)
enum class CozeEvent { InsufficientCreditsBalance /*, ... */ };
```

### OpenAI（agent::helper::Openai）

```cpp
struct Info { std::string model; std::string api_key; std::string voice; };
```

## Emote Helper（service::helper::ExpressionEmote）

头：`brookesia/service_helper/expression/emote.hpp`（来自 `docs/en/expression/emote.rst`、chatbot 示例）

```cpp
enum class FunctionId {
    LoadAssetsSource,   // Source:Object(source/type/flag_enable_mmap)
    SetEmoji,           // Name:String
    HideEmoji,
    InsertAnimation,    // Emote:String, Duration:Number(ms)
    SetEventMessage,    // Type:String(EventMessageType) [, Text:String]
    HideEventMessage,
    SetQrcode,          // Content:String
};

enum class EventMessageType { System, Idle, Speak, Listen, User, Battery };
```

## HAL 接口（hal 命名空间）

头：`brookesia/hal_interface.hpp` / `brookesia/hal_adaptor.hpp`（来自 `examples/service/device`、`examples/service/audio`）

```cpp
namespace esp_brookesia::hal {

// 设备初始化（按设备名，或全部）
bool init_device(const char * device_name);
bool init_all_devices();
void deinit_all_devices();

// 设备名常量
struct StorageDevice { static constexpr const char * DEVICE_NAME = /* "storage" */; };
struct AudioDevice   { static constexpr const char * DEVICE_NAME = /* "audio"   */; };
struct DisplayDevice { static constexpr const char * DEVICE_NAME = /* "display" */; };

// 按接口类型取第一个实例
template <typename Iface>
auto get_first_interface() -> std::tuple<std::string /*name*/, Iface*>;

// 接口类与静态 NAME 常量
class BoardInfoIface        { static /*const*/ char* NAME; };
class DisplayPanelIface     { static char* NAME;
    struct Info { int h_res, v_res; /* pixel_format; */ int get_pixel_bits() const; bool is_valid() const; };
    const Info & get_info() const;
    bool draw_bitmap(int x, int y, int w, int h, const void * buf); };
class DisplayBacklightIface { static char* NAME; };
class DisplayTouchIface     { static char* NAME; };
class AudioCodecPlayerIface { static char* NAME;
    struct Config { int bits; int channels; int sample_rate; };
    bool open(const Config &); void close();
    bool write_data(const uint8_t *, size_t); };
class AudioCodecRecorderIface { static char* NAME; };
class StorageFsIface {
    static char* NAME;
    enum class FileSystemType { LittleFS, /* ... */ };
    struct Info { FileSystemType fs_type; const char * mount_point; /* ... */ };
    std::vector<Info> get_all_info() const;
};
class PowerBatteryIface {
    static char* NAME;
    enum class Ability { ChargeConfig, ChargerControl /*, ... */ };
    enum class ChargeState { Unknown, NotCharging /*, ... */ };
    struct State { std::optional<int> percentage; ChargeState charge_state; /* ... */ };
};

} // namespace
```

## Service Console 命令（examples/service/console）

控制台不是库 API，而是一组串口命令（由 `components/cmd_service/cmd_service.cpp`、`components/cmd_debug/cmd_debug.cpp` 注册）。服务名/函数名/事件名首字母大写，与 Helper 的 `FunctionId`/`EventId` 一一对应。

**服务命令（`svc_*`）**

| 命令 | 说明 |
|---|---|
| `svc_list` | 列出所有已注册服务及运行状态 |
| `svc_funcs <service>` | 列出某服务全部函数及 JSON schema |
| `svc_events <service>` | 列出某服务全部事件 |
| `svc_call <service> <function> [JSON]` | 调用函数（JSON 无空格，无参传 `{}`）。内部固定超时 5000ms |
| `svc_subscribe <service> <event>` | 订阅事件（内部持有 `EventRegistry::SignalConnection`） |
| `svc_unsubscribe <service> <event>` | 取消订阅 |
| `svc_stop <service>` | 释放服务 binding（停止服务） |
| `svc_call_periodic` / `svc_call_delayed` / `svc_call_cancel` / `svc_call_list` | 周期/延迟调用与取消（同组件） |

**RPC 命令（`svc_rpc_*`，默认 port=65500，timeout=2000ms）**

| 命令 | 说明 |
|---|---|
| `svc_rpc_server <start\|stop\|connect\|disconnect> [-p <port>] [-s <svc1,svc2>]` | 启停 RPC Server / 把服务挂上或卸下 |
| `svc_rpc_call <host> <service> <function> [JSON] [-p <port>] [-t <ms>]` | 远程调用函数 |
| `svc_rpc_subscribe <host> <service> <event> [-p <port>] [-t <ms>]` | 订阅远端事件 |
| `svc_rpc_unsubscribe <host> <service> <event> [-p <port>] [-t <ms>]` | 取消远端订阅 |

RPC Client 按 `host:port` 缓存复用、断线自动重连（`get_or_create_rpc_client`）。

## RPC（service::rpc，brookesia_service_manager）

头：`brookesia/service_manager.hpp`（托管）、`brookesia/service_manager/rpc/server.hpp`、`.../rpc/client.hpp`、`.../rpc/connection.hpp`、`.../rpc/protocol.hpp`

```cpp
namespace esp_brookesia::service {

class ServiceManager {
    // RPC 托管（命令内部与程序化用法均经此）
    bool start_rpc_server(const rpc::Server::Config &config = rpc::Server::Config(),
                          uint32_t timeout_ms = 100);
    void stop_rpc_server();
    bool connect_rpc_server_to_services(std::vector<std::string> names = {});   // 空=全部
    bool disconnect_rpc_server_from_services(std::vector<std::string> names = {});
    bool is_rpc_server_running() const;
};

namespace rpc {

class Server {
public:
    struct Config {
        uint16_t listen_port;     // 默认 CONFIG_BROOKESIA_SERVICE_MANAGER_RPC_SERVER_LISTEN_PORT (65500)
        size_t   max_connections; // 默认 CONFIG_BROOKESIA_SERVICE_MANAGER_RPC_SERVER_MAX_CONNECTIONS (2)
    };
    Server(boost::asio::io_context::executor_type executor, const Config &config = Config());
    bool init(); void deinit();
    bool start(uint32_t timeout_ms); void stop();
    bool add_connection(std::shared_ptr<ServerConnection> connection);
    bool remove_connection(const std::string &name);
    std::shared_ptr<ServerConnection> get_connection(const std::string &name);
    bool is_initialized() const; bool is_running() const;
};

class Client {
public:
    using DeinitCallback    = std::function<void()>;
    using DisconnectCallback = std::function<void()>;
    Client(DeinitCallback on_deinit_callback = nullptr);
    bool init(boost::asio::io_context::executor_type executor, DisconnectCallback on_disconnect_callback);
    void deinit();
    bool connect(const std::string &host, uint16_t port, uint32_t timeout_ms);
    void disconnect();
    bool is_initialized() const; bool is_connected() const;
    std::future<FunctionResult> call_function_async(
        const std::string &target, const std::string &method, boost::json::object &&params);
    FunctionResult call_function_sync(
        const std::string &target, const std::string &method,
        boost::json::object &&params, size_t timeout_ms);
    std::string subscribe_event(   // 返回 subscription_id，失败为空串
        const std::string &target, const std::string &event_name,
        EventDispatcher::NotifyCallback callback, size_t timeout_ms);
    bool unsubscribe_events(
        const std::string &target, const std::vector<std::string> &subscription_ids, size_t timeout_ms);
};

} // namespace rpc
} // namespace esp_brookesia::service
```

**相关 Kconfig（`service/brookesia_service_manager/Kconfig`）**：`BROOKESIA_SERVICE_MANAGER_RPC_SERVER_LISTEN_PORT`(65500)、`..._RPC_SERVER_MAX_CONNECTIONS`(2)、`..._RPC_CLIENT_CALL_FUNCTION_TIMEOUT_MS`(2000)、`..._RPC_GLOBAL_MAX_SOCKETS`、`..._DEFAULT_CALL_FUNCTION_TIMEOUT_MS`(500)；调试子开关 `BROOKESIA_SERVICE_MANAGER_ENABLE_DEBUG_LOG` 下的 `..._RPC_DATA_LINK_ENABLE_DEBUG_LOG` / `..._RPC_SERVER_ENABLE_DEBUG_LOG` / `..._RPC_CLIENT_ENABLE_DEBUG_LOG`。

## 运行期剖析器（lib_utils）

头：`brookesia/lib_utils/memory_profiler.hpp`、`.../thread_profiler.hpp`、`.../time_profiler.hpp`。控制台 `debug_*` 命令即封装下列静态/单例方法。

```cpp
namespace esp_brookesia::lib_utils {

// 内存剖析器（debug_mem）
class MemoryProfiler {
public:
    static MemoryProfiler &get_instance();
    static std::shared_ptr<ProfileSnapshot> take_snapshot(ProfileSnapshot *last_snapshot = nullptr);
    static void print_snapshot(const ProfileSnapshot &snapshot);
    static void sample_memory(MemoryInfo &mem_info);
    static bool check_threshold(const ProfileSnapshot &snapshot, ThresholdType type, uint32_t threshold_value);
};

// 线程剖析器（debug_thread）
class ThreadProfiler {
public:
    enum class PrimarySortBy   { None, CoreId };
    enum class SecondarySortBy { CpuPercent, Priority, StackUsage, Name };
    static std::shared_ptr<SampleResult>   sample_tasks();
    static std::shared_ptr<ProfileSnapshot> take_snapshot(
        const SampleResult &start_sample, const SampleResult &end_sample);
    static void sort_tasks(std::vector<TaskInfo> &tasks, PrimarySortBy p, SecondarySortBy s);
    static void print_snapshot(const ProfileSnapshot &snapshot, PrimarySortBy p, SecondarySortBy s);
};

// 时间剖析器（debug_time_report / debug_time_clear）
class TimeProfiler {
public:
    static TimeProfiler &get_instance();
    void report();   // 打印 Performance Tree（calls/total/self/avg/min/max/%parent/%total，单位 ms）
    void clear();
};

} // namespace
```

**打点宏**（`brookesia/lib_utils/time_profiler.hpp`，需在 menuconfig 启用 time profiler；禁用时为空宏，无开销）：

```cpp
BROOKESIA_TIME_PROFILER_SCOPE("name");              // 作用域自动 enter/leave
BROOKESIA_TIME_PROFILER_START_EVENT("name");        // 手动开始（须与 END_EVENT 配对）
BROOKESIA_TIME_PROFILER_END_EVENT("name");          // 手动结束
```

## TaskScheduler（lib_utils）

头：`brookesia/lib_utils.hpp`（来自 `examples/agent/chatbot`）

```cpp
namespace esp_brookesia::lib_utils {

class TaskScheduler {
public:
    struct WorkerConfig { const char * name; int core_id; int priority; int stack_size; };
    struct StartConfig { std::vector<WorkerConfig> worker_configs; };
    bool start(const StartConfig &);
    bool post(std::function<void()> task);
    bool post_delayed(std::function<void()> task, uint32_t delay_ms);
    using OnceTask = std::function<void()>;
};

} // namespace
```
