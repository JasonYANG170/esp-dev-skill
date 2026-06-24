# WASM Native HTTP 与 MQTT

> **适用摘要**: 启用并使用 ESP-WASMachine 暴露给 WASM 应用的 HTTP client 与 MQTT native API（基于 `esp_http_client` 与 `esp-mqtt`，通过函数 ID 派发）。

## 触发意图

- "WASM 应用发 HTTP 请求"
- "WASM 里用 MQTT"
- "wasm_http_client_call_native_func"
- "wasm_mqtt_* API"

## 前置条件

| 条件 | 要求 |
|---|---|
| HTTP | `CONFIG_WASMACHINE_WASM_EXT_NATIVE=y` + `CONFIG_WASMACHINE_WASM_EXT_NATIVE_HTTP_CLIENT=y`（默认 n） |
| MQTT | 同上 + `CONFIG_WASMACHINE_WASM_EXT_NATIVE_MQTT=y`（默认 n），且 **`CONFIG_WASMACHINE_APP_MGR=y`** |
| 网络 | 设备已联网（`sta` 或 `example_connect`） |

## 分步说明

### 1. 启用（sdkconfig.defaults）

```ini
CONFIG_WASMACHINE_WASM_EXT_NATIVE=y
CONFIG_WASMACHINE_WASM_EXT_NATIVE_HTTP_CLIENT=y
CONFIG_WASMACHINE_APP_MGR=y                  # MQTT 必需
CONFIG_WASMACHINE_WASM_EXT_NATIVE_MQTT=y
```

### 2. HTTP client native API

来源 `components/wasmachine_ext_wasm_native/src/wm_ext_wasm_native_http_client.c`。仅注册一个 import，按 `func_id` 派发：

| import | 签名 |
|---|---|
| `wasm_http_client_call_native_func` | `(ii*)i`  即 `(func_id, argc, args_buf)` |

顶层 `func_id`：

| ID 宏 | 值 | 含义 |
|---|---|---|
| `HTTP_CLIENT_INIT` | 0 | 创建 client |
| `HTTP_CLIENT_SET_STR` | 1 | 设字符串型属性 |
| `HTTP_CLIENT_GET_STR` | 2 | 取字符串型属性 |
| `HTTP_CLIENT_SET_INT` | 3 | 设整型属性 |
| `HTTP_CLIENT_GET_INT` | 4 | 取整型属性 |
| `HTTP_CLIENT_COMMON` | 5 | perform/close/cleanup 等 |

子 ID 举例（详见 `resources/api_reference.md` §4）：
- SET_STR：`HTTP_CLIENT_SET_URL`(0)、`_SET_POST_FILED`(1)、`_SET_HEADER`(2)、`_SET_USERNAME`(3)、`_SET_PASSWORD`(4)、`_DELETE_HEADER`(5)、`_WRITE_DATA`(6)
- GET_STR：`_READ_DATA`(4)、`_READ_RESP`(5)、`_GET_URL`(6)
- SET_INT：`_SET_AUTHTYPE`(0)、`_SET_METHOD`(1)、`_SET_TIMEOUT`(2)、`_OPEN`(3)
- GET_INT：`_GET_STATUS_CODE`(3)、`_GET_CONTENT_LENGTH`(4)、`_IS_CHUNKED`(2)、`_GET_ERRNO`(0)
- COMMON：`_PERFORM`(0)、`_CLOSE`(1)、`_CLEANUP`(2)、`_SET_REDIRECTION`(3)、`_ADD_AUTH`(4)

> 调用模式：`INIT` 拿到 handle → `SET_*` 配置（URL/method/headers）→ `OPEN`/`PERFORM` → `READ_DATA`/`GET_STATUS_CODE` → `CLEANUP`。底层是 `esp_http_client_*`。

### 3. MQTT native API

来源 `components/wasmachine_ext_wasm_native/src/wm_ext_wasm_native_mqtt.c`。注册到 `"env"`：

| import | 签名 |
|---|---|
| `wasm_mqtt_init` | `(**)i` |
| `wasm_mqtt_destory` | `(i)i` （源码拼写为 `destory`） |
| `wasm_mqtt_start` / `stop` / `reconnect` / `disconnect` | `(i)i` |
| `wasm_mqtt_publish` | `(i$*ii)i` (handle, topic, data, len, qos) |
| `wasm_mqtt_subscribe` | `(i$i)i` |
| `wasm_mqtt_unsubscribe` | `(i$)i` |
| `wasm_mqtt_enqueue` | `(i$*iiii)i` |
| `wasm_mqtt_set_uri` | `(i$)i` |
| `wasm_mqtt_config` | `(i*)i` |
| `wasm_mqtt_get_outbox_size` | `(i)i` |

`wasm_mqtt_init` / `wasm_mqtt_config` 用 attr-container 传键值，键名（字符串）：
`"host"`、`"uri"`、`"port"`、`"client_id"`、`"username"`、`"password"`、`"lwt_topic"`、`"lwt_msg"`、`"lwt_qos"`、`"lwt_retain"`、`"keepalive"`（默认 120）、`"disable_clean_session"`、`"disable_auto_reconnect"`、`"cert_pem"`/`"cert_len"`、`"client_cert_pem"`/`"client_cert_len"`、`"client_key_pem"`/`"client_key_len"`、`"path"` 等。

事件以 `MQTT_EVENT_WASM`（`WASM_Msg_Start + 5`）回送 WASM 应用，事件 attr-container 键：`"event_id"`、`"data"`、`"data_len"`、`"total_len"`、`"offset"`、`"topic"`、`"topic_len"`、`"msg_id"`、`"session"`、`"error_code"`、`"retain"`、`"qos"`、`"dup"`。

### 4. 调用模式（伪代码）

```c
/* MQTT: init(配置) → start → subscribe/publish → ... → destory */
handle = wasm_mqtt_init(config_in, config_out);   /* uri/port/username/password 等 */
wasm_mqtt_start(handle);
wasm_mqtt_subscribe(handle, "cmd/device1", 1);
wasm_mqtt_publish(handle, "telemetry", payload, len, 1);
/* 事件经 MQTT_EVENT_WASM 异步回调到 WASM 应用 */
wasm_mqtt_destory(handle);     /* 注意拼写 */
```

## 常见错误

| 错误 | 原因 | 解决方法 |
|---|---|---|
| MQTT import 找不到 | 未开 App Manager | 先 `CONFIG_WASMACHINE_APP_MGR=y` |
| 调 `destory` 报未定义 | 拼成 `destroy` | 源码拼写就是 `wasm_mqtt_destory` |
| HTTP `CLEANUP` 前崩溃 | handle 流程不完整 | 严格按 INIT→配置→PERFORM→(读)→CLEANUP |
| MQTT 连不上 | 网络/cert/uri 错 | `sta` 已连？`uri`/`port`/cert 正确？看 `error_code` 事件键 |
| 配置项无效 | 键名拼错 | 用上面列出的精确键名（区分大小写） |

## 参考

- `components/wasmachine_ext_wasm_native/src/wm_ext_wasm_native_http_client.c` — 全部 func_id 与 wrapper
- `components/wasmachine_ext_wasm_native/src/wm_ext_wasm_native_mqtt.c` — MQTT import 表与配置键
- `resources/api_reference.md` §4、§5 — 完整 import 表
- `recipes/shell_wifi.md` — 联网
