# 存储 / 内存 / 重启工具（espnow_storage / espnow_mem / espnow_utils）

> **适用摘要**: 使用组件封装的 NVS 存储接口、带调试记录的内存宏、以及重启计数/异常判定等工具函数（基于 `src/utils/include/` 三个头文件）。

> Version: selected repo/component version; align with the user project and dependency manifest.
> Evidence: `repos/esp-now/resources/`, source/examples in `repos/esp-now/`, and this recipe path `repos/esp-now/recipes/storage_utils.md`.
> Validation: draft metadata added from repository routing; verify APIs, Kconfig symbols, and component dependencies against the selected version.

## 触发意图

- "保存配置到 NVS"
- "espnow_storage"
- "内存调试 ESP_MALLOC"
- "重启计数"
- "MAC 字符串转 hex"

## 前置条件

| 条件 | 要求 |
|---|---|
| 头文件 | `espnow_storage.h`, `espnow_mem.h`, `espnow_utils.h` |
| NVS | 初始化由 `espnow_storage_init()` 完成（app_main 首步） |

## 分步说明

### 1. NVS 存储封装（espnow_storage.h）

```c
#include "espnow_storage.h"

// 必须最先调用（各示例 app_main 首行）
espnow_storage_init();

// key 最长 15 字符；value 最长 1984 字节（或受分区大小限制）
uint32_t bulb_status = 1;
espnow_storage_set("bulb_key", &bulb_status, sizeof(bulb_status));

uint32_t read_val = 0;
espnow_storage_get("bulb_key", &read_val, sizeof(read_val));

espnow_storage_erase("bulb_key");
```

> NVS namespace 由 `CONFIG_ESPNOW_NVS_NAMESPACE`（默认 `"espnow"`）控制。

### 2. 内存调试宏（espnow_mem.h）

```c
#include "espnow_mem.h"

// 这些宏在 CONFIG_ESPNOW_MEM_DEBUG=y 时会记录分配/释放，便于排查泄漏
void *p = ESP_MALLOC(128);
void *q = ESP_CALLOC(8, 16);
void *r = ESP_REALLOC(p, 256);
// 失败一直重试直到成功:
void *s = ESP_REALLOC_RETRY(NULL, 512);
ESP_FREE(s);
ESP_FREE(r);
ESP_FREE(q);

// 排查工具
espnow_mem_print_record();   // 打印所有未释放的分配（需 CONFIG_ESPNOW_MEM_DEBUG=y 且日志级 ESP_LOG_INFO）
espnow_mem_print_heap();     // 堆与栈剩余
espnow_mem_print_task();     // 系统任务状态
```

> SPIRAM 场景：开 `CONFIG_ESPNOW_MEM_ALLOCATION_SPIRAM=y`（依赖芯片 SPIRAM 支持），`ESP_MALLOC` 等会分配到 SPIRAM。

### 3. 重启与异常工具（espnow_utils.h）

```c
#include "espnow_utils.h"

// 延时重启（允许其它任务收尾，如上报 OTA 成功）
espnow_reboot(pdMS_TO_TICKS(1000));

int unbroken = espnow_reboot_unbroken_count();  // 连续重启次数
int total    = espnow_reboot_total_count();     // 历史总重启次数
bool is_exc  = espnow_reboot_is_exception(true); // 本次是否异常重启，true 顺带擦 coredump

// 连续重启回滚: CONFIG_ESPNOW_REBOOT_UNBROKEN_INTERVAL_TIMEOUT(默认 5000ms)
//               CONFIG_ESPNOW_REBOOT_UNBROKEN_FALLBACK_COUNT(默认 30)
```

### 4. MAC 转换与系统信息（espnow_utils.h）

```c
const char *mac_str = "aa:bb:cc:dd:ee:ff";
uint8_t mac_hex[6];
espnow_mac_str2hex(mac_str, mac_hex);   // 返回 mac_hex 或 NULL

// 周期打印系统信息
espnow_print_system_info(5000);   // 每 5s 打印一次
```

### 5. 错误处理宏（espnow_utils.h）

```c
ESP_PARAM_CHECK(ptr != NULL);                  // 失败返回 ESP_ERR_INVALID_ARG
ESP_ERROR_RETURN(ret != ESP_OK, ret, "fail");  // 失败打印并 return ret
ESP_ERROR_GOTO(ret != ESP_OK, EXIT, "fail");   // 失败 goto EXIT
ESP_ERROR_CONTINUE(size <= 0, "");             // 循环内失败 continue
ESP_ERROR_BREAK(cond, "");                     // 失败 break
ESP_ERROR_ASSERT(espnow_init(&cfg));           // 失败 abort
```

## 常见错误

| 错误 | 原因 | 解决方法 |
|---|---|---|
| `espnow_storage_set` 失败 | 未 `espnow_storage_init` | app_main 首步调用 |
| key 超长 | 超过 15 字符 | 缩短 key |
| 内存记录不打印 | `CONFIG_ESPNOW_MEM_DEBUG=n` | 开启并 `esp_log_level_set(esp_mem, ESP_LOG_INFO)` |
| SPIRAM 分配失败 | 未开芯片 SPIRAM 支持 | 用默认分配或确认 `ESP32S3_SPIRAM_SUPPORT` 等 |
| `ESP_REALLOC_RETRY` 死循环 | size 合法但堆不足 | 检查可用堆，必要时减小请求 |
| `ESP_FREE` 后仍访问 | 误用 | 宏内已置 NULL，勿保留旧指针 |

## 参考

- `src/utils/include/espnow_storage.h` / `espnow_mem.h` / `espnow_utils.h`
- Kconfig：`CONFIG_ESPNOW_NVS_NAMESPACE` / `CONFIG_ESPNOW_MEM_DEBUG` / `CONFIG_ESPNOW_MEM_ALLOCATION_SPIRAM` / `CONFIG_ESPNOW_REBOOT_*`
- 各示例 app_main 首行均调用 `espnow_storage_init()`
