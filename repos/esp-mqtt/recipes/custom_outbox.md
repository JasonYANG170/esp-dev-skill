# 自定义 Outbox 实现

> **适用摘要**: 通过 `CONFIG_MQTT_CUSTOM_OUTBOX` 替换默认 outbox 实现（如持久化到 NVM、用 C++ 内存资源等），对应 `examples/custom_outbox/`。

> Version: selected repo/component version; align with the user project and dependency manifest.
> Validation: draft metadata added from repository routing; verify APIs, Kconfig symbols, and component dependencies against the selected version.

## 触发意图

- "自定义 outbox"
- "MQTT outbox 持久化"
- "CONFIG_MQTT_CUSTOM_OUTBOX"
- "替换 MQTT outbox"

## 前置条件

| 条件 | 要求 |
|---|---|
| Kconfig | `CONFIG_MQTT_CUSTOM_OUTBOX=y` |
| 参考示例 | `examples/custom_outbox/`（C++ 实现） |

## 分步说明

### 1. 启用自定义 outbox（来自 `Kconfig`）

```
Component config > ESP-MQTT Configurations
  [*] Enable custom outbox implementation (CONFIG_MQTT_CUSTOM_OUTBOX)
```

> 开启后默认 outbox 实现不再被定义，必须自行提供实现并加入 mqtt 组件源码。

### 2. 将自定义实现追加到 mqtt 组件（来自 `Kconfig` help 与 `docs/en/index.rst`）

在项目 `CMakeLists.txt`（顶层）追加：
```cmake
idf_component_get_property(mqtt mqtt COMPONENT_LIB)
set_property(TARGET ${mqtt} PROPERTY SOURCES ${PROJECT_DIR}/custom_outbox.c APPEND)
```

> C++ 实现还需启用异常等（`examples/custom_outbox/sdkconfig.defaults` 自动配置），并在源码中提供等价于默认 outbox 的接口。

### 3. 客户端使用与默认一致（来自 `examples/custom_outbox/main/app_main.c`）

```c
esp_mqtt_client_config_t mqtt_cfg = {
    .broker.address.uri = CONFIG_BROKER_URL,
};
esp_mqtt_client_handle_t client = esp_mqtt_client_init(&mqtt_cfg);
esp_mqtt_client_register_event(client, ESP_EVENT_ANY_ID, mqtt_event_handler, NULL);

// 用 enqueue 演示 outbox 分配
int msg_id = esp_mqtt_client_enqueue(client, "topic/qos1", "data_3", 0, 1, 0, true);
ESP_LOGI(TAG, "Enqueued msg_id=%d", msg_id);
msg_id = esp_mqtt_client_enqueue(client, "topic/qos2", "QoS2 message", 0, 2, 0, true);
ESP_LOGI(TAG, "Enqueued msg_id=%d", msg_id);

esp_mqtt_client_start(client);
```

### 4. outbox 接口来源

outbox 内部接口定义在 `lib/include/mqtt_outbox.h`。自定义实现需提供等价的 outbox 创建 / 入队 / 取出 / 删除 / 大小查询能力，签名需与 `mqtt_outbox.h` 一致（库内部按这些原型调用）。`examples/custom_outbox/` 以 C++ 重新实现了相同功能，用多态内存资源（polymorphic memory resource）控制分配。

### 5. CMake 与依赖

`examples/custom_outbox/main/CMakeLists.txt`：
```cmake
idf_component_register(SRCS "app_main.c"
                       INCLUDE_DIRS "."
                       PRIV_REQUIRES mqtt nvs_flash esp_netif)
```
（自定义 outbox 源码通过顶层 CMake 的 `set_property ... APPEND` 注入 mqtt 组件，而非 main 组件。）

## 常见错误

| 错误 | 原因 | 解决方法 |
|---|---|---|
| 链接错误（重复定义 / 未定义 outbox 符号） | 未追加实现或未开 Kconfig | 开 `CONFIG_MQTT_CUSTOM_OUTBOX` 且 `set_property APPEND` 追加源码 |
| C++ 实现链接失败 | 未启用 C++ 异常 / RTTI | 参考 `examples/custom_outbox/sdkconfig.defaults` |
| 默认 outbox 仍生效 | Kconfig 未刷新 | `idf.py fullclean` 后重新构建 |

## 参考

- `examples/custom_outbox/main/app_main.c`
- `examples/custom_outbox/README.md`
- `lib/include/mqtt_outbox.h`（outbox 内部接口）
- `Kconfig`（`CONFIG_MQTT_CUSTOM_OUTBOX` help）
