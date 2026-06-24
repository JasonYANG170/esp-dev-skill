# Peer / Custom 数据传输（host ↔ 协处理器原始二进制通道）

> **适用摘要**: 在 host 与协处理器之间建立一条独立于 RPC 控制流量与网络/BT 数据之外的私有二进制通道。用 `msg_id` 区分不同业务，发送任意原始字节（最大 8166 字节/包），host 与 slave 两侧 API 同名。适用于 host ↔ co-processor 的应用级命令/传感数据透传。

## 触发意图

- "ESP-Hosted 自定义数据传输"
- "host 与 slave 传原始字节"
- "esp_hosted_send_custom_data"
- "esp_hosted_register_custom_callback"
- "peer data / custom RPC channel"
- "msg_id 私有通道"

## 前置条件

| 条件 | 要求 |
|---|---|
| 链路 | 已 `esp_hosted_init()` + `esp_hosted_connect_to_slave()` 成功 |
| 头文件 | `#include "esp_hosted.h"` |
| 传输 | SDIO / SPI FD / SPI HD / UART 全部支持 |
| Slave 例程 | `slave/main/example_peer_data_transfer.c`（在 slave menuconfig：`Example Configuration → Additional higher layer examples to run → Peer Data Transfer Example`） |
| 版本一致 | 若用新的 `local_context` 回调签名，host 与 slave 均 ≥ v2.12.4 |
| 参考例程 | `examples/host_peer_data_transfer/` |

> `msg_id` 可为任意 `uint32_t`，但 `0xFFFFFFFF` 保留。

## 分步说明

### 1. 核心 API（来自 `host/esp_hosted.h`，slave 侧 `slave/main/esp_hosted_peer_data.h` 同名）

```c
// 发送（host → slave 或反向，msg_id 标识业务）
esp_err_t esp_hosted_send_custom_data(uint32_t msg_id_to_send,
                                      const uint8_t *data_to_send,
                                      size_t data_len_to_send);

// 注册回调：当收到指定 msg_id 的数据时被调用
esp_err_t esp_hosted_register_custom_callback(uint32_t msg_id_exp,
    void (*callback)(uint32_t msg_id_recvd,
                     const uint8_t *data_recvd,
                     size_t data_len_recvd,
                     void *local_context),     // v2.12.4+ 新增
    void *local_context);                       // 注册时传入，每次回调原样回传
```

### 2. 回调签名变更（v2.12.4，见 `docs/migration_guide.md` 与 SKILL Pitfall #10）

| 版本 | 签名 |
|---|---|
| < 2.12.4 | `void (*)(uint32_t msg_id, const uint8_t *data, size_t data_len)` |
| ≥ 2.12.4 | `void (*)(uint32_t msg_id_recvd, const uint8_t *data_recvd, size_t data_len_recvd, void *local_context)` + 额外 `local_context` 入参 |

host 与 slave **必须同时 ≥ 2.12.4** 才能用新签名。

### 3. 典型用法（改编自 `examples/host_peer_data_transfer/main/peer_data_example.c`）

定义成对的 msg_id（请求/响应），并为每个响应注册回调：

```c
#include "esp_log.h"
#include "esp_hosted.h"

/* 业务 msg_id（避开 0xFFFFFFFF） */
#define MSG_ID_CAT    1   /* 请求：小数据 */
#define MSG_ID_MEOW   2   /* 响应：回显小数据 */
#define MSG_ID_DOG    3
#define MSG_ID_WOOF   4
#define MSG_ID_HUMAN  5
#define MSG_ID_HELLO  6

#define PEER_DATA_MAX_PAYLOAD_SIZE  8166   /* 经验上限 */

/* 每个 callback 可绑定一个静态 context，框架原样回传 */
typedef struct { char home[32]; char likes[32]; } animal_ctx_t;
static animal_ctx_t meow_ctx  = { "cozy apartment", "sunny window" };

static void meow_callback(uint32_t msg_id, const uint8_t *data,
                          size_t data_len, void *user)
{
    /* user == 注册时传入的 &meow_ctx，无需全局变量 */
    ESP_LOGI(TAG, "MEOW %zu bytes, ctx=%s", data_len,
             user ? ((animal_ctx_t *)user)->home : "(null)");
    /* ... 校验 data 并处理 ... */
}

void app_main(void)
{
    ESP_ERROR_CHECK(nvs_flash_init());
    esp_hosted_init();
    esp_hosted_connect_to_slave();

    /* 注册响应回调 */
    ESP_ERROR_CHECK(esp_hosted_register_custom_callback(MSG_ID_MEOW, meow_callback, &meow_ctx));
    ESP_ERROR_CHECK(esp_hosted_register_custom_callback(MSG_ID_WOOF, woof_callback, &woof_ctx));
    ESP_ERROR_CHECK(esp_hosted_register_custom_callback(MSG_ID_HELLO, hello_callback, &hello_ctx));

    /* 发送请求（slave 侧 example_peer_data_transfer.c 收到后回显为对应响应 msg_id） */
    uint8_t buf[512];
    for (int i = 0; i < sizeof(buf); i++) buf[i] = (uint8_t)(i & 0xFF);
    esp_hosted_send_custom_data(MSG_ID_CAT, buf, sizeof(buf));
}
```

### 4. slave 侧（协处理器）

在 slave 工程启用：
```text
Example Configuration → Additional higher layer examples to run → Select Examples to run → [*] Peer Data Transfer Example
```

slave 例程 `slave/main/example_peer_data_transfer.c` 注册了对应 msg_id 的回调，把收到的数据原样回发为配对的响应 msg_id（echo）。自定义业务时替换该文件即可。

### 5. 期望输出（echo 测试，来自例程 README）

```text
========================================
Peer Data Transfer Test (max: 8166 bytes)
========================================
copro <-- host : 1 byte stream, sent ✅
host --> copro : 1 byte stream received, verification: ✅
copro --> host : 1 byte stream received, verification: ✅
...
copro <-- host : 8166 byte stream, sent ✅
copro --> host : 8166 byte stream received, verification: ✅
copro <-- host : 8200 byte stream (exceeds limit - skipped)
```

## 常见错误

| 错误 | 原因 | 解决方法 |
|---|---|---|
| > 8166 字节发送失败 | 单包上限 8166 | 应用层分包；超大结构用多包 |
| 回调拿不到上下文 | 用了 < 2.12.4 旧签名 | host 与 slave 都升到 ≥ 2.12.4 并用新签名 |
| 结构体两端解析错位 | 默认对齐不一致 | 加 `__attribute__((packed))` 保证布局一致 |
| 收不到数据 | slave 未启用 Peer Data Transfer 例程 / 未注册对端 msg_id | slave menuconfig 启用；双方注册互补的 msg_id |
| 用了 `0xFFFFFFFF` 作 msg_id | 该值保留 | 换其它 uint32 |
| context 指针不一致 | 注册后又被释放/改地址 | 用静态/堆上稳定的对象作 `local_context` |

## 参考项目

- `examples/host_peer_data_transfer/` — host 侧 echo 测试例程（多 msg_id、大小覆盖、context 校验）
- `examples/host_peer_data_transfer/main/peer_data_example.c` — 注册回调 + 发送 + 校验完整流程
- `slave/main/example_peer_data_transfer.c` — slave 侧 echo 实现（slave 端 API 同名）
- `slave/main/esp_hosted_peer_data.h` — slave 端 peer data API
- `docs/migration_guide.md` — v2.12.4 回调签名变更说明（对应 SKILL Pitfall #10）
