# 固件 OTA 升级

> **适用摘要**: 两种真实 OTA 路线 —— (A) native OTA：用 socket 从 HTTP 服务器拉取镜像，经 `esp_ota_begin/write/end/set_boot_partition` 写入备用 OTA 分区并重启；(B) esp_https_ota 简易接口：一行调用基于 HTTPS 拉取升级。

## 触发意图

- "OTA 升级"
- "远程更新固件"
- "esp_ota"
- "esp_https_ota"

## 前置条件

| 条件 | 要求 |
|---|---|
| 参考示例 (A) | `examples/system/ota/native_ota/`（含 `1MB_flash`、`2+MB_flash/{new_to_new_no_old,new_to_new_with_old}`） |
| 参考示例 (B) | `examples/system/ota/simple_ota_example/` |
| 分区表 | 必须含 `ota_0` / `ota_1`（app） + `otadata`（data/ota，0x2000）。menuconfig 选 "Two OTA definitions" 或自定义 CSV |
| 配置 | `CONFIG_PARTITION_TABLE_OTA=y`（或 Custom CSV）；`CONFIG_ESPTOOLPY_FLASHSIZE` 与 flash 匹配 |
| 联网 | 先连上 WiFi（示例用 `example_connect()`） |
| 服务器 | 放置待升级镜像（native: HTTP IP+端口+文件名；simple: HTTPS URL + CA 证书） |

## 分步说明

### 路线 A：native OTA（socket + esp_ota_ops，改编自示例）

```c
#include "esp_ota_ops.h"
#include "esp_system.h"
#include "esp_log.h"
#include "nvs_flash.h"
#include "esp_netif.h"
#include "esp_event.h"
#include "protocol_examples_common.h"
#include "freertos/FreeRTOS.h"
#include "freertos/task.h"
#include <sys/socket.h>
#include <netdb.h>

#define EXAMPLE_SERVER_IP    CONFIG_SERVER_IP
#define EXAMPLE_SERVER_PORT  CONFIG_SERVER_PORT
#define EXAMPLE_FILENAME     CONFIG_EXAMPLE_FILENAME
#define BUFFSIZE             1500
#define TEXT_BUFFSIZE        1024
#define OTA_SIZE_UNKNOWN     0xffffffff

static const char *TAG = "ota";

static void ota_example_task(void *pv)
{
    esp_err_t err;
    esp_ota_handle_t update_handle = 0;
    const esp_partition_t *update_partition = NULL;

    // 当前 boot 与运行分区（诊断用）
    const esp_partition_t *configured = esp_ota_get_boot_partition();
    const esp_partition_t *running    = esp_ota_get_running_partition();

    // 1. 连 HTTP 服务器（socket + connect，略，见 recipes/http_request.md）
    // connect_to_http_server();

    // 2. 发 GET，recv 到 text[]（略）

    // 3. 取下一个待写分区
    update_partition = esp_ota_get_next_update_partition(NULL);
    assert(update_partition != NULL);

    // 4. begin（image_size 可传 OTA_SIZE_UNKNOWN）
    err = esp_ota_begin(update_partition, OTA_SIZE_UNKNOWN, &update_handle);
    if (err != ESP_OK) { ESP_LOGE(TAG, "begin failed"); vTaskDelete(NULL); }

    // 5. 循环 recv → esp_ota_write（处理 HTTP 头/体切片，详见示例）
    while (/* 还有数据 */ 1) {
        // int n = recv(...);
        // 解析出 body 后：
        err = esp_ota_write(update_handle, ota_write_data, buff_len);
        if (err != ESP_OK) { ESP_LOGE(TAG, "write failed"); vTaskDelete(NULL); }
        // if 收完 break;
    }

    // 6. end → set_boot_partition → restart
    if (esp_ota_end(update_handle) != ESP_OK) {
        ESP_LOGE(TAG, "end failed"); vTaskDelete(NULL);
    }
    err = esp_ota_set_boot_partition(update_partition);
    if (err != ESP_OK) { ESP_LOGE(TAG, "set_boot failed"); vTaskDelete(NULL); }
    ESP_LOGI(TAG, "Prepare to restart!");
    esp_restart();
}

void app_main()
{
    ESP_ERROR_CHECK(nvs_flash_init());
    ESP_ERROR_CHECK(esp_netif_init());
    ESP_ERROR_CHECK(esp_event_loop_create_default());
    ESP_ERROR_CHECK(example_connect());
    xTaskCreate(&ota_example_task, "ota_example_task", 8192, NULL, 5, NULL);
}
```

> 完整 HTTP 头解析与状态机（`ESP_OTA_INIT/PREPARE/START/RECEVED/FINISH`）见 `examples/system/ota/native_ota/2+MB_flash/new_to_new_no_old/main/ota_example_main.c`。

### 路线 B：esp_https_ota 简易接口（改编自 simple_ota 示例）

```c
#include "esp_ota_ops.h"
#include "esp_http_client.h"
#include "esp_https_ota.h"
#include "nvs_flash.h"
#include "esp_netif.h"
#include "esp_event.h"
#include "protocol_examples_common.h"
#include "freertos/FreeRTOS.h"
#include "freertos/task.h"

// CA 证书由组件以 .pem 形式嵌入（components/ + CMakeLists.txt 的 EMBED_FILES / TEXT_FILES）
extern const uint8_t server_cert_pem_start[] asm("_binary_ca_cert_pem_start");
extern const uint8_t server_cert_pem_end[]   asm("_binary_ca_cert_pem_end");

static const char *TAG = "simple_ota";

void simple_ota_example_task(void *pv)
{
    ESP_LOGI(TAG, "Starting OTA...");
    esp_http_client_config_t config = {
        .url = CONFIG_FIRMWARE_UPGRADE_URL,
        .cert_pem = (char *)server_cert_pem_start,
        .event_handler = _http_event_handler,        // 可选，记录事件
    };
    esp_err_t ret = esp_https_ota(&config);          // 一行升级
    if (ret == ESP_OK) {
        esp_restart();
    } else {
        ESP_LOGE(TAG, "Firmware Upgrades Failed");
    }
    while (1) vTaskDelay(1000 / portTICK_PERIOD_MS);
}

void app_main()
{
    ESP_ERROR_CHECK(nvs_flash_init());
    ESP_ERROR_CHECK(esp_netif_init());
    ESP_ERROR_CHECK(esp_event_loop_create_default());
    ESP_ERROR_CHECK(example_connect());
    xTaskCreate(&simple_ota_example_task, "ota", 8192, NULL, 5, NULL);
}
```

### 关键 API

| API | 作用 |
|---|---|
| `esp_ota_get_boot_partition()` | 当前配置的启动分区 |
| `esp_ota_get_running_partition()` | 实际运行分区（诊断配置与运行不一致） |
| `esp_ota_get_next_update_partition(NULL)` | 下一个待写的 OTA 分区 |
| `esp_ota_begin(partition, image_size, &handle)` | 开始；`image_size` 可为 `OTA_SIZE_UNKNOWN` |
| `esp_ota_write(handle, data, size)` | 增量写入 |
| `esp_ota_end(handle)` | 结束并校验 |
| `esp_ota_set_boot_partition(partition)` | 设为下次启动分区（**必做**） |
| `esp_https_ota(const esp_http_client_config_t *)` | 简易 HTTPS OTA（封装上述流程） |

## 常见错误

| 错误 | 原因 | 解决方法 |
|---|---|---|
| 升级后仍跑旧固件 | 漏 `esp_ota_set_boot_partition` | end 成功后必调用 |
| `esp_ota_end` 失败 | 镜像不完整/写错分区 | 校验 Content-Length 与写入字节数一致 |
| 分区找不到 | 分区表无 ota_0/ota_1/otadata | menuconfig 选 "Two OTA definitions" 或自定义 CSV |
| app 分区跨 1MB | ota_0/ota_1 偏移错 | ota_1 = ota_0 + 0x100000，各自不跨 1MB |
| simple_ota 证书错 | CA 不匹配/未嵌入 | 把 `server_certs/ca_cert.pem` 嵌入，url 用对应 HTTPS |
| `begin` 返回内存/参数错 | image_size 非法 | 用 `OTA_SIZE_UNKNOWN` 或准确长度 |
| 升级中途断网 | 未做断点续传 | 重试整轮；native OTA 不自带续传 |

## 参考

- `examples/system/ota/native_ota/2+MB_flash/new_to_new_no_old/main/ota_example_main.c` — native OTA 完整实现
- `examples/system/ota/simple_ota_example/main/simple_ota_example.c` — esp_https_ota 简易示例
- `examples/system/ota/native_ota/README.md`、`OTA_workflow.png` — OTA 工作流说明
- `components/app_update/include/esp_ota_ops.h` — `esp_ota_*` 原型
- `docs/en/api-guides/fota-from-old-new.rst` — OTA 迁移说明
