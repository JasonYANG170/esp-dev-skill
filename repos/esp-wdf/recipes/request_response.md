# 应用间请求/响应

> **适用摘要**：用 `api_register_resource_handler` 注册资源处理者，用 `init_request` + `api_send_request` 异步发起请求并接收响应，构造 `response_t` 回送结果。

## 触发意图

- "应用间请求响应"
- "注册资源处理器"
- "request handler sender"
- "WASM app 远程调用"

## 前置条件

| 条件 | 要求 |
|---|---|
| Kconfig | `CONFIG_WAMR_APP_FRAMEWORK=y` |
| 参考 | `examples/simple/request_handler/`、`examples/simple/request_sender/` |

## 分步说明

### 1. 资源处理者（取自 examples/simple/request_handler）

```c
#include "wasm_app.h"
#include "wa-inc/request.h"

static void
url1_request_handler(request_t *request)
{
    response_t response[1];          /* 数组取址天然非空，避免 NULL 崩溃 */
    attr_container_t *payload;

    printf("[resp] ### user resource 1 handler called\n");

    if (request->payload != NULL && request->fmt == FMT_ATTR_CONTAINER)
        attr_container_dump((attr_container_t *)request->payload);

    payload = attr_container_create("wasm app response payload");
    if (payload == NULL)
        return;

    attr_container_set_string(&payload, "key1", "value1");
    attr_container_set_string(&payload, "key2", "value2");

    /* 根据请求生成配对响应 */
    make_response_for_request(request, response);
    /* 设置响应：状态码 CONTENT_2_05、格式、负载、负载长度 */
    set_response(response, CONTENT_2_05, FMT_ATTR_CONTAINER, (void *)payload,
                 attr_container_get_serialize_length(payload));
    api_response_send(response);

    attr_container_destroy(payload);
}

static void
url2_request_handler(request_t *request)
{
    response_t response[1];
    make_response_for_request(request, response);
    set_response(response, DELETED_2_02, 0, NULL, 0);   /* 无负载响应 */
    api_response_send(response);
    printf("### user resource 2 handler called\n");
}

void
on_init()
{
    /* 注册两个资源 URL */
    api_register_resource_handler("/url1", url1_request_handler);
    api_register_resource_handler("/url2", url2_request_handler);
}

void
on_destroy() {}
```

### 2. 请求发送者（取自 examples/simple/request_sender）

```c
#include "wasm_app.h"
#include "wa-inc/request.h"

static void
my_response_handler(response_t *response, void *user_data)
{
    char *tag = (char *)user_data;

    if (response == NULL) {
        printf("[req] request timeout!\n");
        return;
    }

    printf("[req] response handler called mid:%d, status:%d, fmt:%d, "
           "payload:%p, len:%d, tag:%s\n",
           response->mid, response->status, response->fmt, response->payload,
           response->payload_len, tag);

    if (response->payload != NULL && response->payload_len > 0
        && response->fmt == FMT_ATTR_CONTAINER) {
        printf("[req] dump the response payload:\n");
        attr_container_dump((attr_container_t *)response->payload);
    }
}

static void
test_send_request(char *url, char *tag)
{
    request_t request[1];

    /* 初始化请求：url、方法、格式、负载、负载长度 */
    init_request(request, url, COAP_PUT, 0, NULL, 0);
    /* 异步发送；响应到达回调 my_response_handler，tag 作为 user_data 透传 */
    api_send_request(request, my_response_handler, tag);
}

void
on_init()
{
    /* 跨应用寻址用 /app/<app_name> 前缀 */
    test_send_request("/app/request_handler/url1", "a request to target app");
    /* 通用请求（不带前缀） */
    test_send_request("url1", "a general request");
}

void
on_destroy() {}
```

### 3. 关键点

- 跨应用寻址：URL 加 `/app/<目标应用名>` 前缀（如 `/app/request_handler/url1`）。
- `make_response_for_request` / `set_response` / `init_request` 的指针不可为 NULL，故用 `response_t response[1];` 形式声明。
- CoAP 状态码：成功用 `CONTENT_2_05`、`DELETED_2_02`、`CHANGED_2_04` 等；方法用 `COAP_GET/POST/PUT/DELETE`。

## 常见错误

| 错误 | 原因 | 解决方法 |
|---|---|---|
| 请求超时（`response==NULL`） | URL 写错或目标应用未注册该资源 | 跨应用加 `/app/<name>` 前缀；确认目标已 `api_register_resource_handler` |
| 处理者收到请求但无响应 | 忘记 `api_response_send(response)` | 构造完 response 必须发送 |
| 崩溃 | `response`/`request` 指针为 NULL | 用 `xxx_t var[1];` 声明再取址 |
| 响应负载读不出 | 未校验 `response->fmt == FMT_ATTR_CONTAINER` | 先判格式再转 `attr_container_t*` |

## 参考

- `examples/simple/request_handler/main/request_handler.c`
- `examples/simple/request_sender/main/request_sender.c`
- `components/wamr/app-framework/include/wa-inc/request.h`
- `components/wamr/app-framework/include/bi-inc/shared_utils.h`
- `resources/api_reference.md` —— 第 1.3、1.4 节
