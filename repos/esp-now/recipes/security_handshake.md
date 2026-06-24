# 安全握手与加解密收发

> **适用摘要**: initiator 扫描 responder、用 ECDH+PoP 完成握手并分发 app key，双方基于 AES-CCM 加解密 ESP-NOW 用户数据（参考 `examples/security`）。

## 触发意图

- "ESP-NOW 加密"
- "安全握手"
- "AES-CCM"
- "espnow_sec"
- "ECDH PoP"

## 前置条件

| 条件 | 要求 |
|---|---|
| Kconfig | `CONFIG_ESPNOW_APP_SECURITY=y`（默认开），可选 `CONFIG_ESPNOW_ALL_SECURITY` 加密所有数据 |
| 配置 | `espnow_config_t.sec_enable = 1` |
| 头文件 | `espnow_security.h`, `espnow_security_handshake.h` |
| 参考示例 | `examples/security/main/app_main.c` |

## 分步说明

### 1. 公共：开启 sec_enable 并注册数据回调

```c
#include "espnow.h"
#include "espnow_security.h"
#include "espnow_security_handshake.h"

const char *pop_data = CONFIG_APP_ESPNOW_SESSION_POP;  // Proof of Possession 字符串

espnow_config_t cfg = ESPNOW_INIT_CONFIG_DEFAULT();
cfg.sec_enable = 1;
espnow_init(&cfg);

espnow_set_config_for_data_type(ESPNOW_DATA_TYPE_DATA, true, app_recv_cb);
```

### 2. Initiator：扫描 → 分发 app key

```c
uint8_t key_info[APP_KEY_LEN];   // 32 字节
if (espnow_get_key(key_info) != ESP_OK) {
    esp_fill_random(key_info, APP_KEY_LEN);   // 首次随机生成
}
espnow_set_key(key_info);        // 发送加密
espnow_set_dec_key(key_info);    // 接收解密（两者都要）

espnow_sec_responder_t *info_list = NULL;
size_t num = 0;
espnow_sec_initiator_scan(&info_list, &num, pdMS_TO_TICKS(3000));
ESP_LOGW(TAG, "found %u responders", num);
if (num == 0) { ESP_FREE(info_list); return; }

espnow_addr_t *dest = ESP_MALLOC(num * ESPNOW_ADDR_LEN);
for (size_t i = 0; i < num; i++)
    memcpy(dest[i], info_list[i].mac, ESPNOW_ADDR_LEN);
espnow_sec_initiator_scan_result_free();

espnow_sec_result_t result = {0};
esp_err_t ret = espnow_sec_initiator_start(key_info, pop_data,
                                           (const uint8_t(*)[6])dest, num, &result);
ESP_ERROR_GOTO(ret != ESP_OK, EXIT, "sec_initiator_start");
ESP_LOGI(TAG, "successed %u, unfinished %u",
         result.successed_num, result.unfinished_num);
EXIT:
ESP_FREE(dest);
espnow_sec_initiator_result_free(&result);
```

### 3. Responder：启动握手并监听结果

```c
uint8_t key_info[APP_KEY_LEN];
// 若已有历史 key，先恢复（否则等握手成功后再 set）
if (espnow_get_key(key_info) == ESP_OK) {
    espnow_set_key(key_info);
    espnow_set_dec_key(key_info);
}

esp_event_handler_register(ESP_EVENT_ESPNOW, ESP_EVENT_ANY_ID,
                           app_sec_event_handler, NULL);
espnow_sec_responder_start(pop_data);

// 事件处理
static void app_sec_event_handler(void *a, esp_event_base_t base, int32_t id, void *data)
{
    if (base != ESP_EVENT_ESPNOW) return;
    if (id == ESP_EVENT_ESPNOW_SEC_OK) {
        ESP_LOGI(TAG, "SEC_OK " MACSTR, MAC2STR((uint8_t *)data));
        s_sec_flag = true;   // 握手成功，后续可加密发送
    } else if (id == ESP_EVENT_ESPNOW_SEC_FAIL) {
        ESP_LOGW(TAG, "SEC_FAIL " MACSTR, MAC2STR((uint8_t *)data));
        s_sec_flag = false;
    }
}
```

### 4. 加密发送：frame_head.security 必须置位

```c
espnow_frame_head_t fh = {
    .broadcast        = true,
    .retransmit_count = 10,
    .security         = s_sec_flag,   // true 时才加密；握手前为明文
};
uint8_t *data = ESP_CALLOC(1, ESPNOW_SEC_PACKET_MAX_SIZE);  // 加密净荷更小
size_t size = ...;
espnow_send(ESPNOW_DATA_TYPE_DATA, ESPNOW_ADDR_BROADCAST, data, size, &fh, portMAX_DELAY);
```

### 5. 底层加解密 API（通常无需手动调用，握手后组件自动处理）

```c
// espnow_security.h 提供的底层接口（需要自行管理 sec 上下文时使用）
esp_err_t espnow_sec_init(espnow_sec_t *sec);
esp_err_t espnow_sec_deinit(espnow_sec_t *sec);
esp_err_t espnow_sec_setkey(espnow_sec_t *sec, uint8_t app_key[APP_KEY_LEN]);
esp_err_t espnow_sec_auth_encrypt(espnow_sec_t *sec, const uint8_t *in, size_t ilen,
                                  uint8_t *out, size_t out_len, size_t *olen, size_t tag_len);
esp_err_t espnow_sec_auth_decrypt(espnow_sec_t *sec, const uint8_t *in, size_t ilen,
                                  uint8_t *out, size_t out_len, size_t *olen, size_t tag_len);
```

> 关键尺寸：`APP_KEY_LEN=32`, `KEY_LEN=16`, `IV_LEN=8`, `TAG_LEN=4`，`ESPNOW_SEC_PACKET_MAX_SIZE = ESPNOW_PAYLOAD_LEN - TAG_LEN - IV_LEN`。

## 常见错误

| 错误 | 原因 | 解决方法 |
|---|---|---|
| 数据仍是明文 | `frame_head.security=0` | 握手成功后置 `security=true` 再发 |
| 只加密发不能解密收 | 只调 `espnow_set_key` | 同时调 `espnow_set_dec_key` |
| 扫描不到 responder | responder 未 `espnow_sec_responder_start` | responder 先启动握手 |
| 加密净荷不足 | 用了 `ESPNOW_DATA_LEN` 分配 | 加密用 `ESPNOW_SEC_PACKET_MAX_SIZE` |
| PoP 不匹配握手失败 | 两端 `pop_data` 不一致 | 统一 `CONFIG_APP_ESPNOW_SESSION_POP` |
| 想加密所有功能数据 | 各功能独立 | 开 `CONFIG_ESPNOW_ALL_SECURITY` 或分别开 `_CONTROL_SECURITY`/`_OTA_SECURITY` 等 |

## 参考

- `examples/security/main/app_main.c` — initiator/responder 双角色完整示例
- `src/security/include/espnow_security.h` / `espnow_security_handshake.h`
- Kconfig：`CONFIG_ESPNOW_APP_SECURITY` / `CONFIG_ESPNOW_ALL_SECURITY` / `CONFIG_ESPNOW_*_SECURITY`
