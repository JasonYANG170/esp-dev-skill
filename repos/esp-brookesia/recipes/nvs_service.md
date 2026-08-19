# NVS 服务：键值存储

> **适用摘要**: 使用 `brookesia_service_nvs` 做基于命名空间的键值存储。提供两套 API：类型安全的 `save_key_value`/`get_key_value`（推荐）与通用 JSON 的 `Set`/`Get`/`List`/`Erase`（细粒度控制）。

> Version: selected repo/component version; align with the user project and dependency manifest.
> Evidence: `repos/esp-brookesia/resources/`, source/examples in `repos/esp-brookesia/`, and this recipe path `repos/esp-brookesia/recipes/nvs_service.md`.
> Validation: draft metadata added from repository routing; verify APIs, Kconfig symbols, and component dependencies against the selected version.

## 触发意图

- "用 Brookesia 存配置到 NVS"
- "保存/读取键值"
- "命名空间管理"
- "存复杂 struct 到 NVS"

## 前置条件

| 条件 | 要求 |
|---|---|
| 依赖 | `espressif/brookesia_service_nvs` |
| 参考示例 | `examples/service/nvs` |

## 分步说明

### 1. 启动并绑定

```cpp
using NVS_Helper = service::helper::NVS;
auto &service_manager = service::ServiceManager::get_instance();
service_manager.init();
service_manager.start();
BROOKESIA_CHECK_FALSE_EXIT(NVS_Helper::is_available(), "NVS service is not available");
auto binding = service_manager.bind(NVS_Helper::get_name().data());
BROOKESIA_CHECK_FALSE_EXIT(binding.is_valid(), "Failed to bind NVS service");
```

### 2. 类型安全 API（推荐）

直接存储类型（bool / int32_t / uint8_t / int16_t 等 ≤32 位整数）：

```cpp
NVS_Helper::save_key_value("config", "enable_feature", true);
NVS_Helper::save_key_value("config", "volume", int16_t(-32768));
auto b = NVS_Helper::get_key_value<bool>("config", "enable_feature");
if (b) { BROOKESIA_LOGI("feature=%1%", b.value()); }
```

序列化存储类型（string / int64 / uint64 / float / double / vector / 描述过的 struct）：

```cpp
// 自定义 struct 必须先描述
struct Point { int x; int y; };
BROOKESIA_DESCRIBE_STRUCT(Point, (), (x, y))

struct Address { std::string city; int zip; };
BROOKESIA_DESCRIBE_STRUCT(Address, (), (city, zip))

NVS_Helper::save_key_value("config", "greeting", std::string("Hello, NVS!"));
auto s = NVS_Helper::get_key_value<std::string>("config", "greeting");

std::vector<int> nums = {1, 2, 3};
NVS_Helper::save_key_value("config", "number_list", nums);
auto v = NVS_Helper::get_key_value<std::vector<int>>("config", "number_list");
```

可选超时：`get_key_value<T>(nspace, key, timeout_ms)`、`save_key_value(nspace, key, value, timeout_ms)`。

### 3. 通用 JSON API（细粒度）

```cpp
using KVMap = NVS_Helper::KeyValueMap;   // std::map<string, std::variant<bool,int32_t,string,...>>

// Set：namespace(String), KV(Object)
KVMap kv = {
    {"key_str", std::string("Espressif")},
    {"key_int", int32_t(100)},
    {"key_bool", bool(true)},
};
NVS_Helper::call_function_sync(
    NVS_Helper::FunctionId::Set, "storage",
    BROOKESIA_DESCRIBE_TO_JSON(kv).as_object());

// List：namespace(String) -> Array<EntryInfo>
auto list = NVS_Helper::call_function_sync<boost::json::array>(
    NVS_Helper::FunctionId::List, "storage");
std::vector<NVS_Helper::EntryInfo> entries;
BROOKESIA_DESCRIBE_FROM_JSON(list.value(), entries);

// Get：namespace(String), keys(Array) -> Object
std::vector<std::string> keys = {"key_str", "key_int"};
auto got = NVS_Helper::call_function_sync<boost::json::object>(
    NVS_Helper::FunctionId::Get, "storage",
    BROOKESIA_DESCRIBE_TO_JSON(keys).as_array());
std::string val = got.value().at("key_str").as_string().c_str();

// Erase：namespace(String), keys(Array)
std::vector<std::string> to_erase = {"key_bool"};
NVS_Helper::call_function_sync(
    NVS_Helper::FunctionId::Erase, "storage",
    BROOKESIA_DESCRIBE_TO_JSON(to_erase).as_array());
```

### 4. 命名空间管理与批量清除

```cpp
// 清空整个命名空间（第二个参数传空 {} 表示全部）
NVS_Helper::erase_keys("config", {});
NVS_Helper::erase_keys("config");          // 仅传 namespace 也清空该空间
```

### 5. 错误处理

```cpp
// 取不存在的 key -> expected 为否
auto miss = NVS_Helper::get_key_value<std::string>("storage", "non_existent_key", 100);
if (!miss) { BROOKESIA_LOGI("Expected miss: %1%", miss.error()); }

// 类型不匹配 -> expected 为否
NVS_Helper::save_key_value("storage", "type_test", std::string("txt"));
auto bad = NVS_Helper::get_key_value<int32_t>("storage", "type_test", 100);
if (!bad) { BROOKESIA_LOGI("Type mismatch: %1%", bad.error()); }
```

## 常见错误

| 错误 | 原因 | 解决方法 |
|---|---|---|
| struct 存/取失败 | 未用 `BROOKESIA_DESCRIBE_STRUCT` 描述 | 在 struct 定义后加描述宏 |
| 通用 API 参数报错 | KV 不是 Object / keys 不是 Array | 用 `BROOKESIA_DESCRIBE_TO_JSON(x).as_object()/.as_array()` |
| List 返回解析失败 | 直接当字符串用 | 返回是 Array，反序列化为 `std::vector<NVS_Helper::EntryInfo>` |
| 命名空间串数据 | 多模块共用同一空间 | 用不同 namespace（storage / config / stats 等）|
| 大整数丢精度 | 直接存 int64 走序列化 | 用 `save_key_value<int64_t>` 显式类型 |

## 参考

- `examples/service/nvs/main/main.cpp` — basic / type-safe / namespace / error handling 四套演示
- `docs/en/service/nvs.rst` — NVS 服务接口契约
