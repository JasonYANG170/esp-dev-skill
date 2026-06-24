# 数据存储调优（RTC / RAM）与低内存事件

> **适用摘要**: 调整 ESP-Insights 诊断数据存储（默认 RTC memory）的容量与水位线，监听低内存事件以应对 RTC 满载丢日志。

## 触发意图

- "RTC 存储满了怎么办"
- "调大 critical data 存储"
- "诊断数据丢日志"
- "data store 低内存事件"
- "用 RAM 代替 RTC 存储"

## 前置条件

| 条件 | 要求 |
|---|---|
| Kconfig | `CONFIG_DIAG_DATA_STORE_RTC`（默认，依赖 `SOC_RTC_MEM_SUPPORTED`）或 `CONFIG_DIAG_DATA_STORE_RAM` |
| 运行 | 已 `esp_insights_init(&config)` |
| 参考 | `components/esp_diag_data_store/Kconfig`、头文件 `esp_diag_data_store.h`、`README.md` Behind the Scenes 段 |

## 分步说明

### 1. 数据存储类型（来自 `components/esp_diag_data_store/Kconfig`）

| 选项 | 说明 |
|---|---|
| `DIAG_DATA_STORE_RTC` | 默认，存 RTC memory（依赖 `SOC_RTC_MEM_SUPPORTED`），软复位保留 |
| `DIAG_DATA_STORE_RAM` | 内部 RAM（断电丢，跨重启不保留） |
| `DIAG_DATA_STORE_FLASH` | 占位，当前不支持 |

> RTC store 内部用 `rtc_store`（见 `components/esp_diag_data_store/README.md`）。

### 2. 容量与分区（来自 Kconfig）

RTC 模式：
```
CONFIG_RTC_STORE_DATA_SIZE=6144           # 总大小；ESP32 默认 3072；范围 512–7168
CONFIG_RTC_STORE_CRITICAL_DATA_SIZE=4096  # critical 区；ESP32 默认 2048；余下给 non-critical
```
RAM 模式：
```
CONFIG_RAM_STORE_DATA_SIZE=6144
CONFIG_RAM_STORE_CRITICAL_DATA_SIZE=4096
```

> 容量估算（来自 README）：critical 每条 ~121 字节，non-critical 每条 ~49 字节。例：2048B critical ≈ 16 条；1024B non-critical ≈ 21 条。

### 3. 水位线与低内存事件（来自 Kconfig + 头文件）

```
CONFIG_DIAG_DATA_STORE_REPORTING_WATERMARK_PERCENT=80   # 范围 50–90
```
buffer 填到该比例时发事件（`components/esp_diag_data_store/include/esp_diag_data_store.h`）：
```c
ESP_DIAG_DATA_STORE_EVENT_CRITICAL_DATA_WRITE_FAIL
ESP_DIAG_DATA_STORE_EVENT_NON_CRITICAL_DATA_WRITE_FAIL
ESP_DIAG_DATA_STORE_EVENT_CRITICAL_DATA_LOW_MEM
ESP_DIAG_DATA_STORE_EVENT_NON_CRITICAL_DATA_LOW_MEM
```

监听：
```c
#include "esp_event.h"
#include "esp_diag_data_store.h"

static void on_ds_event(void *arg, esp_event_base_t base, int32_t id, void *data)
{
    if (id == ESP_DIAG_DATA_STORE_EVENT_CRITICAL_DATA_LOW_MEM) {
        /* 立即触发上报以腾出空间 */
        esp_insights_send_data();
    }
}

void register_ds_events(void)
{
    ESP_ERROR_CHECK(esp_event_handler_register(ESP_DIAG_DATA_STORE_EVENT,
                                               ESP_EVENT_ANY_ID, on_ds_event, NULL));
}
```

### 4. 直接读写/释放（高级，来自头文件）

异步发送时可自行 read → send → release：
```c
uint8_t buf[1024];
int n = esp_diag_data_store_critical_read(buf, sizeof(buf));
/* 发送 n 字节 ... */
esp_diag_data_store_critical_release(n);

int m = esp_diag_data_store_non_critical_read(buf, sizeof(buf));
esp_diag_data_store_non_critical_release(m);

/* 主动写入（一般不需要，metrics/log API 内部已调用）：
 * esp_diag_data_store_critical_write(data, len);
 * esp_diag_data_store_non_critical_write("tag", data, len);
 */
/* 丢弃当前缓冲数据：esp_diag_data_discard_data();（init 之后调用） */
```

### 5. 调试打印

```
CONFIG_DIAG_DATA_STORE_DBG_PRINTS=y   # 来自 sdkconfig.ci
```

## 常见错误

| 错误 | 原因 | 解决方法 |
|---|---|---|
| 日志出现 gap / 丢失 | RTC 满 | 调大 `RTC_STORE_CRITICAL_DATA_SIZE`，或监听 low_mem 立即 send |
| ESP32 上 RTC 容量上不去 | 旧 IDF 限 4K | 用 ESP-IDF v4.3+ 可达 8K（max 7168） |
| 想跨硬重启保留 | RTC 仅软复位保留 | 现版本 FLASH 存储未支持；降低对硬重启保留的预期 |
| 选 RAM 模式后跨重启丢 | RAM 断电即失 | 这是预期行为；要跨重启请用 RTC |
| 加密 flash 报错 | data 分区需加密 | 见 Kconfig 注释：启用 Flash Encryption 时诊断分区须加密 |

## 参考

- `components/esp_diag_data_store/Kconfig` — 存储类型、容量、水位线
- `components/esp_diag_data_store/include/esp_diag_data_store.h` — read/write/release、事件枚举
- `components/esp_diag_data_store/README.md` — RTC/RAM 抽象说明
- `README.md`（仓库）“Behind the Scenes”/“RTC data store” 段 — 容量估算
- `examples/minimal_diagnostics/sdkconfig.ci` — `CONFIG_DIAG_DATA_STORE_DBG_PRINTS=y`
