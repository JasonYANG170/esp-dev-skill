# SNTP 网络对时与系统时间

> **适用摘要**: 联网后用 LwIP SNTP 模块从 NTP 服务器（如 `pool.ntp.org`）同步系统时间，通过 `time()`/`localtime_r()` 读取，用 POSIX `setenv("TZ",...)` + `tzset()` 设置时区。时间同步是 TLS 证书校验、日志时间戳、定时任务的前置需求。

> Version: ESP8266 RTOS SDK version used by the project.
> Evidence: `repos/ESP8266_RTOS_SDK/resources/`, source/examples in `repos/ESP8266_RTOS_SDK/`, and this recipe path `repos/ESP8266_RTOS_SDK/recipes/sntp_time.md`.
> Validation: draft metadata added from repository routing; verify APIs, Kconfig symbols, and component dependencies against the selected version.

## 触发意图

- "ESP8266 对时 / NTP"
- "SNTP 获取时间"
- "设置时区 / 本地时间"
- "时间戳 / 定时任务"

## 前置条件

| 条件 | 要求 |
|---|---|
| 参考示例 | `examples/protocols/sntp`（`main/sntp_example_main.c`） |
| 联网 | 先 `example_connect()`（依赖 `protocol_examples_common`），需已 `IP_EVENT_STA_GOT_IP` |
| 组件 | LwIP SNTP（`lwip/apps/sntp.h`，随 LwIP 组件提供） |
| 栈 | SNTP 任务建议 ≥ 2048（示例用 2048，README 提示"allocate large stack space"） |

## 分步说明

### 1. app_main 初始化（改编自示例）

```c
#include "nvs_flash.h"
#include "esp_netif.h"
#include "esp_event.h"
#include "protocol_examples_common.h"
#include "freertos/FreeRTOS.h"
#include "freertos/task.h"

void app_main(void)
{
    ESP_ERROR_CHECK(nvs_flash_init());
    ESP_ERROR_CHECK(esp_netif_init());
    ESP_ERROR_CHECK(esp_event_loop_create_default());
    ESP_ERROR_CHECK(example_connect());               // 连 WiFi

    // SNTP 走 LwIP，建议分配较大栈
    xTaskCreate(sntp_example_task, "sntp_example_task", 2048, NULL, 10, NULL);
}
```

### 2. 初始化 SNTP

```c
#include "lwip/apps/sntp.h"

static void initialize_sntp(void)
{
    ESP_LOGI(TAG, "Initializing SNTP");
    sntp_setoperatingmode(SNTP_OPMODE_POLL);          // 轮询模式
    sntp_setservername(0, "pool.ntp.org");            // 服务器 0
    sntp_init();
}
```

> 也可用 `sntp_setserver()` 传 `ip_addr_t` 直接指定 IP。

### 3. 等待时间同步完成

示例用 `tm_year` 判断（未同步时年份是 1970）：

```c
#include <time.h>

static void obtain_time(void)
{
    initialize_sntp();

    time_t now = 0;
    struct tm timeinfo = { 0 };
    int retry = 0;
    const int retry_count = 10;

    while (timeinfo.tm_year < (2016 - 1900) && ++retry < retry_count) {
        ESP_LOGI(TAG, "Waiting for system time to be set... (%d/%d)", retry, retry_count);
        vTaskDelay(2000 / portTICK_PERIOD_MS);
        time(&now);
        localtime_r(&now, &timeinfo);
    }
}
```

> LwIP SNTP 收到响应后内部调 `settimeofday()` 更新系统时间（见 sntp README）。所以应用侧只需 `time()` 读。

### 4. 设置时区（POSIX TZ）

在读取时间后、进入循环前设置 `TZ` 环境变量并 `tzset()`：

```c
// 中国标准时间（UTC+8）
setenv("TZ", "CST-8", 1);
tzset();

// 美东（EST，带夏令时）
// setenv("TZ", "EST5EDT,M3.2.0/2,M11.1.0", 1);
// tzset();
```

> TZ 字符串格式见 [libc 文档](https://www.gnu.org/software/libc/manual/html_node/TZ-Variable.html)。`setenv` 第 3 参 `overwrite=1` 表示覆盖现有值。

### 5. 读取并格式化时间

```c
time_t now;
struct tm timeinfo;
char strftime_buf[64];

time(&now);
localtime_r(&now, &timeinfo);                        // 应用 TZ

strftime(strftime_buf, sizeof(strftime_buf), "%c", &timeinfo);
ESP_LOGI(TAG, "The current date/time is: %s", strftime_buf);
```

### 6. 关键 API / 函数

| API | 作用 |
|---|---|
| `sntp_setoperatingmode(SNTP_OPMODE_POLL)` | 设为轮询模式 |
| `sntp_setservername(idx, "pool.ntp.org")` | 设第 idx 个 NTP 服务器域名 |
| `sntp_init()` | 启动 SNTP（内部收到响应调 `settimeofday`） |
| `time(time_t *)` | 读 UTC 时间（秒，自 epoch） |
| `localtime_r(&now, &timeinfo)` | 转 `struct tm`（应用 TZ） |
| `gettimeofday` / `settimeofday` | 取/设精确时间（含微秒） |
| `setenv("TZ", "...", 1)` + `tzset()` | 设时区 |

> 完整 C 时间函数（`asctime/ctime/gmtime/mktime/strftime/difftime/clock`）均可用（见 sntp README）。

## 常见错误

| 错误 | 原因 | 解决方法 |
|---|---|---|
| 一直 `Waiting for system time` | 没拿到 IP / NTP 服务器不通 | 先确认联网；换 NTP 服务器（如 `ntp.aliyun.com`） |
| 年份是 1970 | 还没同步 / 时区未设 | 增加重试次数；`setenv("TZ")+tzset()` |
| 时间差 8 小时 | 时区未设或设错 | `setenv("TZ","CST-8",1)` 后 `tzset()` |
| TLS 握手失败（证书时间相关） | 系统时间不对 | 先做 SNTP 再发 TLS 请求 |
| 启动 panic | 栈太小 | 任务栈加到 2048+ |

## 参考

- `examples/protocols/sntp/main/sntp_example_main.c` — 完整示例（`initialize_sntp` / `obtain_time` / TZ 设置）
- `examples/protocols/sntp/README.md` — LwIP SNTP 说明、时间函数、时区文档链接
- `resources/api_reference.md` — socket / WiFi 函数速查
