# HTTP 服务器 esp_http_server（URI handler）

> **适用摘要**: 用 `esp_http_server` 组件在 ESP8266 上跑 RESTful/控制平面 HTTP 服务，注册 URI handler（`HTTP_GET`/`HTTP_POST`/`HTTP_PUT`）处理 `/hello`、`/echo` 等，支持读请求头/查询串、`httpd_resp_send`/`httpd_resp_send_chunk` 响应、运行期动态注册/注销 handler。是 `http_request.md`（出站请求）的入站对应。

> Version: ESP8266 RTOS SDK version used by the project.
> Evidence: `repos/ESP8266_RTOS_SDK/resources/`, source/examples in `repos/ESP8266_RTOS_SDK/`, and this recipe path `repos/ESP8266_RTOS_SDK/recipes/http_server.md`.
> Validation: draft metadata added from repository routing; verify APIs, Kconfig symbols, and component dependencies against the selected version.

## 触发意图

- "ESP8266 HTTP server"
- "REST 接口 / Web 配置页"
- "httpd URI handler"
- "POST 接收数据 / 回显"

## 前置条件

| 条件 | 要求 |
|---|---|
| 参考示例 | `examples/protocols/http_server/simple`、`/persistent_sockets` |
| 组件 | `esp_http_server`（`#include <esp_http_server.h>`） |
| 联网 | 先 `example_connect()`；server 默认监听 `server_port=80` |
| 配置 | menuconfig → `Component config → HTTP Server`（`CONFIG_HTTPD_MAX_REQ_HDR_LEN`、`CONFIG_HTTPD_MAX_URI_LEN` 等） |

## 分步说明

### 1. app_main 初始化（改编自 simple 示例）

```c
#include "nvs_flash.h"
#include "esp_netif.h"
#include "esp_event.h"
#include "protocol_examples_common.h"
#include <esp_http_server.h>

static httpd_handle_t server = NULL;

void app_main()
{
    ESP_ERROR_CHECK(nvs_flash_init());
    ESP_ERROR_CHECK(esp_netif_init());
    ESP_ERROR_CHECK(esp_event_loop_create_default());
    ESP_ERROR_CHECK(example_connect());

    server = start_webserver();
}
```

### 2. 启动 server 并注册 URI handler

```c
httpd_handle_t start_webserver(void)
{
    httpd_handle_t server = NULL;
    httpd_config_t config = HTTPD_DEFAULT_CONFIG();   // port=80, stack=4096, max_uri_handlers=8

    if (httpd_start(&server, &config) == ESP_OK) {
        httpd_register_uri_handler(server, &hello);   // GET /hello
        httpd_register_uri_handler(server, &echo);    // POST /echo
        return server;
    }
    return NULL;
}
```

> `HTTPD_DEFAULT_CONFIG()`（esp_http_server.h）：`task_priority=tskIDLE_PRIORITY+5`、`stack_size=4096`、`server_port=80`、`max_open_sockets=7`、`max_uri_handlers=8`、`recv/send_wait_timeout=5s`。

### 3. GET handler：读头/查询，回字符串

```c
/* GET /hello?query1=abc → "Hello World!" */
esp_err_t hello_get_handler(httpd_req_t *req)
{
    // 读请求头（示例：Host）
    size_t buf_len = httpd_req_get_hdr_value_len(req, "Host") + 1;
    if (buf_len > 1) {
        char *buf = malloc(buf_len);
        if (httpd_req_get_hdr_value_str(req, "Host", buf, buf_len) == ESP_OK) {
            ESP_LOGI(TAG, "Host: %s", buf);
        }
        free(buf);
    }

    // 读 URL 查询串并解析参数
    buf_len = httpd_req_get_url_query_len(req) + 1;
    if (buf_len > 1) {
        char *buf = malloc(buf_len);
        if (httpd_req_get_url_query_str(req, buf, buf_len) == ESP_OK) {
            char param[32];
            if (httpd_query_key_value(buf, "query1", param, sizeof(param)) == ESP_OK) {
                ESP_LOGI(TAG, "query1=%s", param);
            }
        }
        free(buf);
    }

    // 自定义响应头
    httpd_resp_set_hdr(req, "Custom-Header-1", "Custom-Value-1");

    // 发响应（user_ctx 在注册时传入）
    const char *resp_str = (const char *) req->user_ctx;
    httpd_resp_send(req, resp_str, strlen(resp_str));     // 默认 200, text/html
    return ESP_OK;
}

httpd_uri_t hello = {
    .uri      = "/hello",
    .method   = HTTP_GET,
    .handler  = hello_get_handler,
    .user_ctx = "Hello World!"
};
```

### 4. POST handler：循环收 body，分块回显

```c
/* POST /echo → 回显 body */
esp_err_t echo_post_handler(httpd_req_t *req)
{
    char buf[100];
    int ret, remaining = req->content_len;

    while (remaining > 0) {
        ret = httpd_req_recv(req, buf, MIN(remaining, sizeof(buf)));
        if (ret <= 0) {
            if (ret == HTTPD_SOCK_ERR_TIMEOUT) { continue; }   // 超时重试
            return ESP_FAIL;                                    // 其它错，关 socket
        }
        httpd_resp_send_chunk(req, buf, ret);                  // 分块回显
        remaining -= ret;
    }
    httpd_resp_send_chunk(req, NULL, 0);                        // 结束响应
    return ESP_OK;
}

httpd_uri_t echo = {
    .uri = "/echo", .method = HTTP_POST, .handler = echo_post_handler
};
```

> `HTTPD_SOCK_ERR_FAIL=-1` / `HTTPD_SOCK_ERR_INVALID=-2` / `HTTPD_SOCK_ERR_TIMEOUT=-3`（esp_http_server.h）。handler 返回非 `ESP_OK` 会关闭该 socket。

### 5. 运行期动态注册/注销 handler

```c
/* PUT /ctrl body=0 注销 /hello /echo；body=1 再注册 */
esp_err_t ctrl_put_handler(httpd_req_t *req)
{
    char buf;
    int ret = httpd_req_recv(req, &buf, 1);
    if (ret <= 0) {
        if (ret == HTTPD_SOCK_ERR_TIMEOUT) httpd_resp_send_408(req);   // 408 超时
        return ESP_FAIL;
    }
    if (buf == '0') {
        httpd_unregister_uri(req->handle, "/hello");
        httpd_unregister_uri(req->handle, "/echo");
    } else {
        httpd_register_uri_handler(req->handle, &hello);
        httpd_register_uri_handler(req->handle, &echo);
    }
    httpd_resp_send(req, NULL, 0);                              // 空响应体
    return ESP_OK;
}
```

### 6. 联网/断网自动启停（simple 示例模式）

把 server handle 存全局，在 IP/断开事件里启停：

```c
static void connect_handler(void* arg, esp_event_base_t base, int32_t id, void* data) {
    httpd_handle_t *s = (httpd_handle_t*) arg;
    if (*s == NULL) { *s = start_webserver(); }
}
static void disconnect_handler(void* arg, esp_event_base_t base, int32_t id, void* data) {
    httpd_handle_t *s = (httpd_handle_t*) arg;
    if (*s) { httpd_stop(*s); *s = NULL; }
}

// app_main 里：
esp_event_handler_register(IP_EVENT, IP_EVENT_STA_GOT_IP, &connect_handler, &server);
esp_event_handler_register(WIFI_EVENT, WIFI_EVENT_STA_DISCONNECTED, &disconnect_handler, &server);
```

### 7. 关键 API

| API | 作用 |
|---|---|
| `HTTPD_DEFAULT_CONFIG()` | 默认配置（port 80, stack 4096, max_uri_handlers 8） |
| `httpd_start(handle*, config*)` | 启动 server（返回 `ESP_OK` 时 handle 有效） |
| `httpd_stop(handle)` | 停止并释放 |
| `httpd_register_uri_handler(handle, &httpd_uri_t)` | 注册 handler |
| `httpd_unregister_uri(handle, uri)` / `httpd_unregister_uri_handler(handle, uri, method)` | 注销 |
| `httpd_req_recv(req, buf, len)` | 读 body（返回字节 / `HTTPD_SOCK_ERR_*`） |
| `httpd_req_get_hdr_value_str(req, field, val, size)` / `_len` | 读请求头 |
| `httpd_req_get_url_query_str` / `_len` / `httpd_query_key_value` | 读查询串 |
| `httpd_resp_send(req, buf, len)` | 发完整响应（len=-1 用 strlen） |
| `httpd_resp_send_chunk(req, buf, len)` | 分块发（最后 `buf=NULL,len=0` 结束） |
| `httpd_resp_set_status` / `set_type` / `set_hdr` | 设状态码/Content-Type/自定义头 |
| `httpd_resp_send_408(req)` | 发 408 |

> `HTTP_GET`/`HTTP_POST`/`HTTP_PUT` 来自 `http_parser.h`（esp_http_server.h 内部 include）。`HTTPD_200/400/404/408/500`、`HTTPD_TYPE_JSON/TEXT/OCTET` 等宏在 esp_http_server.h 定义。

## 常见错误

| 错误 | 原因 | 解决��法 |
|---|---|---|
| `httpd_start` 失败（返回非 OK） | 内存不足 / task 创建失败 | 调大栈或减少并发连接 |
| 注册返回 `ESP_ERR_HTTPD_HANDLERS_FULL` | 超过 `max_uri_handlers`（默认 8） | 配置里调大 `max_uri_handlers` |
| 注册返回 `ESP_ERR_HTTPD_HANDLER_EXISTS` | 同 URI+method 重复注册 | 先 `unregister` 再注册 |
| POST body 收不全 | `content_len` 没循环读 | 用 `while(remaining>0)` 循环 `httpd_req_recv` |
| handler 返回 ESP_FAIL 后 socket 关闭 | 这是预期行为（错误即关） | 正常处理返回 `ESP_OK` |
| 收不到请求 | 还没拿到 IP / 端口被占 | 等 `IP_EVENT_STA_GOT_IP`；检查 80 端口 |
| 超时断开 | `recv_wait_timeout` 默认 5s | 配置里调大，或 handler 内处理 `HTTPD_SOCK_ERR_TIMEOUT` |
| 长响应被截断 | 单次 send 有上限 | 改用 `httpd_resp_send_chunk` 分块 |

## 参考

- `examples/protocols/http_server/simple/main/main.c` — GET/POST/PUT + 动态注册注销 + 网络事件启停
- `examples/protocols/http_server/simple/README.md` — curl 测试命令（`curl IP:80/hello`、`POST /echo`）
- `examples/protocols/http_server/persistent_sockets/` — 持久连接（session context）
- `components/esp_http_server/include/esp_http_server.h` — 全部 API、`HTTPD_DEFAULT_CONFIG`、错误码、状态码宏
- `recipes/http_request.md` — 出站 HTTP GET（与本文入站互补）
