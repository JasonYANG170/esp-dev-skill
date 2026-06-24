# Matter OTA Requestor（含加密 OTA）

> **适用摘要**: 启用 Matter OTA Requestor，让设备能向 OTA Provider 拉取并验证 Matter OTA 镜像；进一步启用加密 OTA，用 RSA-3072 私钥解密应用镜像。

## 触发意图

- "Matter OTA"
- "OTA Requestor"
- "加密 OTA / encrypted OTA"
- "esp_matter_ota_requestor_encrypted_init"
- "CONFIG_ENABLE_OTA_REQUESTOR"

## 前置条件

| 条件 | 要求 |
|---|---|
| 配置 | `CONFIG_ENABLE_OTA_REQUESTOR=y`（加密再加 `CONFIG_ENABLE_ENCRYPTED_OTA=y`） |
| 参考工程 | `examples/light/`（已默认开 OTA Requestor，见其 `sdkconfig.defaults`） |
| Provider | `examples/ota_provider/`（host 端 OTA 供应方） |

## 分步说明

### 1. 在 sdkconfig.defaults 启用 OTA

来自 `examples/light/sdkconfig.defaults`：

```text
CONFIG_ENABLE_OTA_REQUESTOR=y
CONFIG_ENABLE_ENCRYPTED_OTA=y     # 仅加密 OTA 时加
```

> `esp_matter::start()` 内部会调用 `esp_matter_ota_requestor_init()` 把 OTA Requestor server cluster 加到 root node，并启动 `esp_matter_ota_requestor_start()`。

### 2. 普通 OTA：仅需上面的 Kconfig

启用 `CONFIG_ENABLE_OTA_REQUESTOR=y` 后，无需应用代码；框架自动挂载 Requestor。OTA Provider 端用 `examples/ota_provider/` 在 host 上运行。

### 3. 加密 OTA：在 start 之后初始化私钥

加密 OTA 需要在 `esp_matter::start()` 之后调用 `esp_matter_ota_requestor_encrypted_init()`（来自 `examples/light/main/app_main.cpp`）。**私钥 buffer 生命周期必须到关机**：

```cpp
#include <esp_matter_ota.h>

#if CONFIG_ENABLE_ENCRYPTED_OTA
extern const char decryption_key_start[] asm("_binary_esp_image_encryption_key_pem_start");
extern const char decryption_key_end[]   asm("_binary_esp_image_encryption_key_pem_end");

static const char *s_decryption_key = decryption_key_start;
static const uint16_t s_decryption_key_len = decryption_key_end - decryption_key_start;
#endif

extern "C" void app_main() {
    // ... nvs, node, endpoints ...
    esp_err_t err = esp_matter::start(app_event_cb);

#if CONFIG_ENABLE_ENCRYPTED_OTA
    err = esp_matter_ota_requestor_encrypted_init(s_decryption_key, s_decryption_key_len);
    ABORT_APP_ON_FAILURE(err == ESP_OK, ESP_LOGE(TAG, "Failed to init encrypted OTA, err: %d", err));
#endif
}
```

私钥文件作为组件资源嵌入（在 `main/CMakeLists.txt` 用 `target_add_binary_data` 注册 `_binary_esp_image_encryption_key_pem`）。

### 4. OTA 行为配置（可选）

`esp_matter_ota_requestor_set_config()` 可调周期查询与看门狗（来自 `esp_matter_ota.h` 的 `esp_matter_ota_config_t`）：

```cpp
esp_matter_ota_config_t cfg = {
    .periodic_query_timeout = 86400,   // 默认 24h 查一次默认 Provider
    .watchdog_timeout       = 300,     // 卡在非 idle 300s 触发
    .impl                   = nullptr, // 用默认 driver/user_consent/image_processor
};
esp_matter_ota_requestor_set_config(cfg);   // 必须在 esp_matter::start() 之后
```

### 5. 生成 OTA 镜像

按 connectedhomeip 的 OTA 指南：先生成应用镜像，再用 `ota_image_tool` 包成 Matter OTA 镜像。加密 OTA 还需先生成 RSA-3072 私钥并加密应用镜像（见 connectedhomeip 的 `docs/platforms/esp32/ota.md#encrypted-ota`）。

## 常见错误

| 错误 | 原因 | 解决方法 |
|---|---|---|
| 设备不查询 Provider | `CONFIG_ENABLE_OTA_REQUESTOR=n` | 在 sdkconfig.defaults 打开 |
| 加密 OTA 崩溃 | 私钥 buffer 被释放 | 嵌入为二进制符号，常驻不释放 |
| `encrypted_init` 报错 | 在 `start()` 之前调用 | 必须在 `esp_matter::start()` 之后 |
| OTA 卡住 | watchdog 触发 | 检查 Provider 可达性、镜像签名 |
| 私钥长度错 | `key_len` 没含结束符 | 用 `end - start` 自然含 PEM 全部字节 |

## 参考

- `D:/esp-skill/espressif-repos/esp-matter/components/esp_matter/esp_matter_ota.h`（`esp_matter_ota_requestor_init/start/encrypted_init/set_config`、`esp_matter_ota_config_t`）
- `D:/esp-skill/espressif-repos/esp-matter/examples/light/main/app_main.cpp`（加密 OTA 初始化片段）
- `D:/esp-skill/espressif-repos/esp-matter/examples/light/sdkconfig.defaults`（`CONFIG_ENABLE_OTA_REQUESTOR=y`）
- `D:/esp-skill/espressif-repos/esp-matter/examples/ota_provider/`（OTA Provider host 工程）
- `D:/esp-skill/espressif-repos/esp-matter/docs/en/developing.rst` — Matter OTA / Encrypted Matter OTA
