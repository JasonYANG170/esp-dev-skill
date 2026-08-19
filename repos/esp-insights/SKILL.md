---
name: esp-insights-skill
description: >-
  AI Skill for ESP-Insights remote diagnostics / observability firmware development on ESP32. Used when
  users need to integrate, configure, or debug ESP-Insights in their ESP-IDF project, including log/metric/
  variable reporting, core dump capture, RTC data store, HTTPS/MQTT transports, and the dashboard workflow.
  Trigger words: "ESP-Insights", "ESP Insights", "esp_insights", "esp_diagnostics", "diagnostics",
  "遥测", "远程诊断", "可观测性", "日志上报", "指标", "coredump", "core dump", "ESP32", "ESP-IDF", "RainMaker"
license: Apache-2.0
metadata:
  author: Community
  version: "1.0.0"
---

# esp-insights-skill

ESP-Insights 远程诊断 / 可观测性固件开发 AI Skill。提供基于真实仓库代码与文档的场景化配方（recipes）、完整 API 速查、配置项参考与常见陷阱。ESP-Insights 是 Espressif 提供的远程诊断方案，在设备运行时采集错误/告警日志、自定义事件、重启原因、core dump 摘要、指标（metrics）与变量（variables），经 CBOR 编码后通过 HTTPS 或 MQTT(TLS) 上报到 ESP Insights / ESP RainMaker 云端，开发者可在 Web 仪表盘查看设备现场健康状态。

## Core Principles

1. **绝不臆造 API** — 先查 `resources/api_reference.md`；查不到即视为不存在。所有函数名、结构体、宏、配置项必须来自真实头文件。
2. **初始化入口只有一个** — `esp_insights_init(&config)` 会自动初始化 transport（按 sdkconfig 选择 HTTPS/MQTT）、diagnostics、数据存储与（如启用）command-response。不要手动重复内部初始化。
3. **日志靠 `--wrap` 拦截** — 启用 `CONFIG_ESP_INSIGHTS_ENABLED=y` 会自动选中 `CONFIG_DIAG_ENABLE_WRAP_LOG_FUNCTIONS`，包装 `esp_log_write` / `esp_log_writev` 以采集 `ESP_LOGE`/`ESP_LOGW`。若另一组件也要包装日志，改用 `CONFIG_DIAG_USE_EXTERNAL_LOG_WRAP` 并调用 `esp_diag_log_writev()` / `esp_diag_log_write()`。
4. **`log_type` 决定采集范围** — `esp_insights_config_t.log_type` 是 `ESP_DIAG_LOG_TYPE_ERROR | ESP_DIAG_LOG_TYPE_WARNING | ESP_DIAG_LOG_TYPE_EVENT` 的位或；只有 error/warning 受 `esp_log_level_set()` 影响，事件（`ESP_DIAG_EVENT`）不受其影响。
5. **HTTPS 需 Auth Key，MQTT 需 Claiming** — HTTPS 传输必须在 `config.auth_key` 填入仪表盘生成的 Auth Key；MQTT(TLS) 需通过 RainMaker CLI claim 设备并把证书写入 `fctry` 分区。`auth_key` 仅对 HTTPS 有效。
6. **核心数据落 RTC，跨重启保留** — Critical（error/warning/event）与 Non-critical（metrics/variables）默认存入 RTC memory，软复位后仍可上报上次未发送的数据。Critical 每条约 121 字节，Non-critical 每条约 49 字节，据此估算容量。
7. **core dump 上报需三件事** — ①启用 `CONFIG_ESP*_ENABLE_COREDUMP_TO_FLASH` + `..._DATA_FORMAT_ELF`；②分区表加 `coredump, data, coredump, , 64K`；③`CONFIG_ESP_INSIGHTS_COREDUMP_ENABLE`（默认依赖前两项）。崩溃后下次启动上报“摘要”（PC、cause、vaddr、通用寄存器、backtrace），上报后从 flash 擦除。
8. **上报间隔动态自适应** — `CONFIG_ESP_INSIGHTS_CLOUD_POST_MIN_INTERVAL_SEC`（默认 60）与 `..._MAX_INTERVAL_SEC`（默认 240）：若上一周期发了数据则间隔翻倍，否则减半。`esp_insights_send_data()` 可手动触发异步发送。
9. **metadata 1.0 vs 2.0 是破坏性差异** — 默认 `CONFIG_ESP_INSIGHTS_META_VERSION_10=y`（旧 1.0，仅用 `key`）；2.0（设 `=n`）用 `tag`+`key` 组合唯一标识，且 API 形参多一个 `tag`（如 `esp_diag_metrics_report_int(tag,key,i)`）。例子默认开启 2.0，迁移到 2.0 后旧指标数据在新面板不再显示。
10. **自定义 transport 用 enable 而非 init** — 覆盖默认 transport 时：`esp_insights_transport_register(&tcfg)` 然后 `esp_insights_enable(&config)`；反之为 `esp_insights_disable()` + `esp_insights_transport_unregister()`。
11. **命令响应仅限 RainMaker MQTT** — `CONFIG_ESP_INSIGHTS_CMD_RESP_ENABLED` 依赖 `ESP_INSIGHTS_ENABLED` 且 `TRANSPORT_MQTT`，仅对 RainMaker MQTT 节点生效；若只用 `esp_insights_enable()`（如 RainMaker `app_insights`），需自行调用 `esp_insights_cmd_resp_enable()`。
12. **上传固件包以交叉引用 rodata** — 设备只上报只读字符串在 ELF 中的地址，云端用上传的固件包（`.zip` 含 bin/elf/map）还原字符串。不上传则日志显示 “Firmware Image missing”。

## When to Use

**Applicable（适用）:**
- 在 ESP-IDF 项目中集成 ESP-Insights，采集错误/告警日志、自定义事件、重启原因
- 配置 core dump 上报（flash 分区、ELF 格式、摘要）
- 注册并上报自定义 metrics（如室温、信号强度）与 variables（如 IP、SSID、关联站点数）
- 启用系统级 heap metrics / Wi-Fi RSSI metrics / network variables
- 选择与配置 HTTPS / MQTT(TLS) 传输、Auth Key、RainMaker Claiming
- 实现自定义 transport（复用已有 TLS/MQTT 连接以节省内存）
- 调整 RTC/RAM 数据存储容量、上报间隔、上报水位线
- 启用 command-response 远程控制（仅 RainMaker MQTT）

**Not applicable（不适用）:**
- 非 ESP32 系列芯片（STM32、CH57x、Nordic 等）— 框架依赖 ESP-IDF
- ESP-IDF 4.x 项目（除非使用 `idf_4_x_compat` 分支与 esp_insights 1.2.x）
- 纯本地调试（无网络/无云端）— Insights 是“远程”诊断，本机 printf 不属于此范畴
- PCB / 原理图设计、硬件选型
- ESP RainMaker 业务逻辑本身（仅当涉及 Insights 复用其 transport 时才相关）

---

## Scenario Quick Reference (Recipes)

当用户意图匹配下列场景时，**先阅读对应配方** — 其中包含完整调用链、分步说明、可复制代码与常见错误。

### 入门与传输

| 配方 | 场景 |
|---|---|
| `recipes/https_quickstart.md` | HTTPS 传输快速接入：Auth Key、`esp_insights_init`、最小示例 |
| `recipes/mqtt_transport.md` | MQTT(TLS) 传输 + RainMaker Claiming、`fctry` 分区、证书 |
| `recipes/custom_transport.md` | 自定义 transport：复用已有 TLS/MQTT 连接，节省内存 |

### 诊断数据采集

| 配方 | 场景 |
|---|---|
| `recipes/log_capture.md` | 错误/告警日志采集：`log_type`、按 tag 调级别、`ESP_DIAG_EVENT` 事件 |
| `recipes/core_dump.md` | Core dump 捕获与上报：Kconfig、分区表、ELF、摘要 |
| `recipes/custom_metrics.md` | 自定义 metrics：注册 + 各数据类型上报（metadata 1.0/2.0） |
| `recipes/custom_variables.md` | 自定义 variables：注册 + 各数据类型上报、与 metrics 区别 |
| `recipes/system_metrics.md` | 系统级 heap metrics / Wi-Fi RSSI metrics / network variables |

### 存储与运行时控制

| 配方 | 场景 |
|---|---|
| `recipes/data_store_tuning.md` | RTC/RAM 数据存储容量、上报水位线、低内存事件 |
| `recipes/runtime_control.md` | 运行时开关：`reporting_enable/disable`、`send_data`、command-response |

---

## Component & Transport Support Matrix

| 维度 | 取值 | 说明 |
|---|---|---|
| 目标芯片 | ESP32 全系 | ESP32 / ESP32-S2 / S3 / C2 / C3 / C6 等（RISC-V 板上无 on-device backtrace 解析，见 `esp_diag_task_snapshot_get` 注释） |
| ESP-IDF | `>= release/v5.1`（推荐 v5.5） | 4.x 仅 `idf_4_x_compat` 分支；v6.0 已兼容（见 CHANGELOG） |
| 主组件 | `esp_insights` | 拉取 `esp_diagnostics`、`esp_diag_data_store`、`qcbor/cbor`、`rmaker_common` |
| 默认传输 | HTTPS | `CONFIG_ESP_INSIGHTS_TRANSPORT_HTTPS`（默认）/ `_MQTT` |
| 默认 host | `https://client.insights.espressif.com` | `CONFIG_ESP_INSIGHTS_TRANSPORT_HTTPS_HOST` |
| 默认数据存储 | RTC memory | `CONFIG_DIAG_DATA_STORE_RTC`（依赖 `SOC_RTC_MEM_SUPPORTED`）；可选 `RAM`；`FLASH` 占位暂不支持 |
| 认证 | HTTPS: Auth Key；MQTT: Claiming 证书 | Auth Key 来自仪表盘 Manage Auth Keys |

## Key Configuration Reference (节选)

> 完整清单见 `resources/config_reference.md`。所有符号均来自仓库 `components/*/Kconfig`。

| Kconfig 选项 | 默认 | 作用 |
|---|---|---|
| `CONFIG_ESP_INSIGHTS_ENABLED` | n | 总开关，自动选中 `DIAG_ENABLE_WRAP_LOG_FUNCTIONS` |
| `CONFIG_ESP_INSIGHTS_TRANSPORT_MQTT` / `_HTTPS` | HTTPS | 选择默认传输 |
| `CONFIG_ESP_INSIGHTS_COREDUMP_ENABLE` | y | core dump 摘要（依赖 coredump-to-flash + ELF） |
| `CONFIG_ESP_INSIGHTS_CLOUD_POST_MIN_INTERVAL_SEC` | 60 | 上报最小间隔（秒） |
| `CONFIG_ESP_INSIGHTS_CLOUD_POST_MAX_INTERVAL_SEC` | 240 | 上报最大间隔（秒） |
| `CONFIG_ESP_INSIGHTS_META_VERSION_10` | y | 旧元数据格式（仅 key）；置 n 启用 2.0（tag+key） |
| `CONFIG_ESP_INSIGHTS_CMD_RESP_ENABLED` | n | command-response（仅 RainMaker MQTT） |
| `CONFIG_DIAG_ENABLE_METRICS` / `_VARIABLES` | y / y | metrics / variables 总开关 |
| `CONFIG_DIAG_ENABLE_HEAP_METRICS` | y | free/最大块/历史最小空闲 |
| `CONFIG_DIAG_ENABLE_WIFI_METRICS` | y | RSSI / 历史 RSSI |
| `CONFIG_DIAG_ENABLE_NETWORK_VARIABLES` | y | SSID/BSSID/channel/auth/IP/netmask/gateway |
| `CONFIG_DIAG_LOG_MSG_ARG_FORMAT_TLV` / `_STRING` | TLV | 日志参数存储格式（TLV 或完整字符串） |
| `CONFIG_DIAG_LOG_MSG_ARG_MAX_SIZE` | 64 | 日志参数缓冲上限（32–255） |
| `CONFIG_RTC_STORE_DATA_SIZE` | 6144（ESP32:3072） | RTC 存储总大小（512–7168） |
| `CONFIG_RTC_STORE_CRITICAL_DATA_SIZE` | 4096（ESP32:2048） | Critical 区大小，余下给 Non-critical |
| `CONFIG_DIAG_DATA_STORE_REPORTING_WATERMARK_PERCENT` | 80 | 触发低内存事件的水位线（50–90） |

## Reporting Lifecycle (状态流)

```
app_main()
   │
   ├─ nvs_flash_init / esp_netif_init / esp_event_loop_create_default / example_connect
   ├─ esp_rmaker_time_sync_init(NULL)        # 可选；未同步则用相对启动时间(µs)
   ├─ esp_insights_init(&config)             # 初始化 transport + diagnostics + 数据存储
   │        ├─ log hook 拦截 ESP_LOGE/ESP_LOGW + ESP_DIAG_EVENT
   │        ├─ heap/wifi/network metrics & variables 按 Kconfig 启用
   │        └─ 启动动态间隔上报定时器 (MIN..MAX sec)
   │
   ├─ 运行期: ESP_LOGE/W / ESP_DIAG_EVENT(...) / esp_diag_metrics_report_*() / esp_diag_variable_report_*()
   │        └─ 写入 RTC store (Critical | Non-critical)
   │
   ├─ (崩溃) coredump 落 flash → 下次启动上报摘要并擦除
   │
   └─ (运行期事件) INSIGHTS_EVENT_TRANSPORT_SEND_SUCCESS/FAILED/RECV (默认事件循环)
```

---

## Critical Pitfalls (Must Read)

以下是最常见的错误，违反任意一条都会导致固件功能异常。

### 1. HTTPS 漏填 auth_key，或给 MQTT 填 auth_key

```c
// ❌ WRONG — HTTPS 传输却没给 auth_key，云端拒绝
esp_insights_config_t config = {
    .log_type = ESP_DIAG_LOG_TYPE_ERROR,
};
esp_insights_init(&config);  // 上报失败

// ✅ CORRECT — HTTPS 必须填 auth_key
extern const char insights_auth_key_start[] asm("_binary_insights_auth_key_txt_start");
esp_insights_config_t config = {
    .log_type = ESP_DIAG_LOG_TYPE_ERROR,
    .auth_key = insights_auth_key_start,  // 仅 HTTPS 有效
};
esp_insights_init(&config);
```

### 2. Auth Key 文件没被链接进固件

```make
# ❌ WRONG — CMakeLists.txt 没有把 auth key 文件作为二进制嵌入
idf_component_register(SRCS "app_main.c" INCLUDE_DIRS ".")

# ✅ CORRECT — HTTPS 时嵌入文本（来自 examples/*/main/CMakeLists.txt）
idf_component_register(SRCS "app_main.c"
                       INCLUDE_DIRS ".")
if (CONFIG_ESP_INSIGHTS_TRANSPORT_HTTPS)
    target_add_binary_data(${COMPONENT_TARGET} "insights_auth_key.txt" TEXT)
endif()
```
对应源码用 `asm("_binary_insights_auth_key_txt_start")` 取首地址。

### 3. core dump 三件套缺一不可

```c
// ❌ WRONG — 只开了 insights，没开 coredump 落 flash 也没加分区
// 上报时拿不到 core dump 摘要

// ✅ CORRECT — sdkconfig.defaults（来自 examples/minimal_diagnostics）
// CONFIG_ESP_INSIGHTS_ENABLED=y
// CONFIG_ESP32_ENABLE_COREDUMP=y
// CONFIG_ESP32_ENABLE_COREDUMP_TO_FLASH=y
// CONFIG_ESP32_COREDUMP_DATA_FORMAT_ELF=y
// CONFIG_ESP32_COREDUMP_CHECKSUM_CRC32=y
// CONFIG_ESP32_CORE_DUMP_MAX_TASKS_NUM=64
// CONFIG_ESP32_CORE_DUMP_STACK_SIZE=1024
//
// partitions.csv 末尾追加（来自 examples/minimal_diagnostics/partitions.csv）：
// coredump, data, coredump, 0x330000, 64K,
```
注意 `CONFIG_ESP_INSIGHTS_COREDUMP_ENABLE` 依赖 coredump-to-flash **且** ELF 格式同时成立。

### 4. 没自定义分区表导致 coredump/fctry 无处可写

```csv
# ❌ WRONG — 用默认单一分区表，coredump / fctry 分区缺失
# partitions.csv 不存在或只有 factory

# ✅ CORRECT — 自定义分区表（来自 examples/minimal_diagnostics/partitions.csv）
# Name,   Type, SubType,  Offset,   Size,    Flags
nvs,      data, nvs,      0x9000,   24K,
phy_init, data, phy,      0xf000,   4K,
factory,  app,  factory,  0x10000,  0x1E0000,
coredump, data, coredump, 0x330000, 64K,
fctry,    data, nvs,      0x340000, 0x6000,   # MQTT claiming 证书存储
```
并设 `CONFIG_PARTITION_TABLE_CUSTOM=y`、`CONFIG_PARTITION_TABLE_CUSTOM_FILENAME="partitions.csv"`。

### 5. metadata 2.0 与 1.0 API 不能混用

```c
// ❌ WRONG — 默认 META_VERSION_10=y（1.0）却调用 2.0 的 report_* 接口
esp_diag_metrics_report_int("temp", "temp1", 25);  // 1.0 下该符号不存在

// ✅ CORRECT — 按当前 Kconfig 选择对应 API
// 1.0 (CONFIG_ESP_INSIGHTS_META_VERSION_10=y):
esp_diag_metrics_add_uint("temp1", room_temp);
// 2.0 (CONFIG_ESP_INSIGHTS_META_VERSION_10=n):
esp_diag_metrics_register("temp", "temp1", "Room temp", "room", ESP_DIAG_DATA_TYPE_UINT);
esp_diag_metrics_report_uint("temp", "temp1", room_temp);
```

### 6. 把 Default log verbosity 设成 No output 会导致 error/warn 不上报

```c
// ❌ WRONG — menuconfig 里 Log output > Default log verbosity = No output
// ESP_LOGE/ESP_LOGW 在预处理期被移除，diagnostics 钩子也一并消失，仪表盘看不到 error/warn

// ✅ CORRECT — 二选一：
// (a) 将 Default log verbosity 保持 >= Warning；
// (b) 若不想串口有日志：把 ESP System Settings > Channel for console output 设为 None，
//     此时 Default log verbosity 仍保留 ESP_LOGE/W，仪表盘可见、串口不可见。
// 注：ESP_DIAG_EVENT 不受 Default log verbosity 影响。
```

### 7. 不上报 ESP_DIAG_EVENT 事件就漏掉自定义事件

```c
// ❌ WRONG — 用 ESP_LOGI 当“事件”，但 insights 默认只采集 error/warning/event 三类
ESP_LOGI(TAG, "user_login user=%s", user);  // INFO 不在采集范围，仪表盘看不到

// ✅ CORRECT — 用 ESP_DIAG_EVENT 宏上报自定义事件（内部同时 ESP_LOGI + esp_diag_log_event）
ESP_DIAG_EVENT(TAG, "user_login user=%s", user);
// 且 config.log_type 必须包含 ESP_DIAG_LOG_TYPE_EVENT
```

### 8. 自定义 transport 用错启动函数

```c
// ❌ WRONG — 注册了自定义 transport 却又调 esp_insights_init()，内部会再注册默认 transport
esp_insights_transport_register(&tcfg);
esp_insights_init(&config);  // 冲突

// ✅ CORRECT — 自定义 transport 用 enable 而非 init
esp_insights_transport_register(&tcfg);  // 覆盖默认 transport
esp_insights_enable(&config);            // 不再内部注册 transport
// 关闭顺序：esp_insights_disable() 后再 esp_insights_transport_unregister()
```

### 9. RISC-V 板上误用 on-device backtrace 解析

```c
// ❌ WRONG — 在 ESP32-C3 等 RISC-V 上假设 esp_diag_task_snapshot_get() 能给出每任务 backtrace
esp_diag_task_info_t tasks[16];
uint32_t n = esp_diag_task_snapshot_get(tasks, 16);
// tasks[i].bt_info 在 RISC-V 上根本不存在（结构体被 #ifndef CONFIG_IDF_TARGET_ARCH_RISCV 包裹）

// ✅ CORRECT — RISC-V 上 bt_info 字段缺失，仅 Xtensa 目标可用；勿在 RISC-V 引用该字段
```

### 10. RTC 存储爆满导致丢日志，却没监听事件

```c
// ❌ WRONG — 只管疯狂 ESP_LOGE，从不调上报，也不监听低内存事件
// RTC 满后框架丢弃新日志，且无任何告警

// ✅ CORRECT — 监听 ESP_DIAG_DATA_STORE_EVENT 并调高上报频率 / 增大存储
ESP_ERROR_CHECK(esp_event_handler_register(ESP_DIAG_DATA_STORE_EVENT,
                  ESP_DIAG_DATA_STORE_EVENT_CRITICAL_DATA_LOW_MEM,
                  on_low_mem, NULL));
// 触发时：esp_insights_send_data(); 或调大 CONFIG_RTC_STORE_CRITICAL_DATA_SIZE
```

### 11. 上传固件包与烧录的二进制不一致

```bash
# ❌ WRONG — 改了代码后只重新 flash，却没重新上传 build/<project>-<ver>.zip
# 仪表盘无法用 ELF 交叉引用 rodata 字符串 → 显示 "Firmware Image missing"

# ✅ CORRECT — 每次构建后同步上传 build/minimal_diagnostics-v1.0.zip
# （idf.py build 会重新生成 zip，即使代码未变；务必与板上二进制对齐）
idf.py build
# 上传 build/<project_name>-<fw_version>.zip 到 Dashboard > Firmware Images
```

### 12. 切换到 MQTT 后忘记 claim 或缺 fctry 分区

```bash
# ❌ WRONG — 只在 menuconfig 选 MQTT，没 claim 也没加 fctry 分区 → 连不上云端

# ✅ CORRECT — MQTT 需要 RainMaker Claiming（见 recipes/mqtt_transport.md）
# partitions.csv 含: fctry, data, nvs, 0x340000, 0x6000,
# 烧录后: cd path/to/esp-insights/cli && ./rainmaker.py claim <serial-port>
```

### 13. command-response 用在 HTTPS 节点上

```c
// ❌ WRONG — HTTPS 传输下期望 dashboard 能远程控制设备
// CONFIG_ESP_INSIGHTS_CMD_RESP_ENABLED 依赖 TRANSPORT_MQTT，HTTPS 下不可用

// ✅ CORRECT — command-response 仅限 RainMaker MQTT 节点
// menuconfig: Component config → ESP Insights → Enable command response module
// 且仅当通过 esp_insights_enable()（非 init）启动时，需自行调用：
esp_insights_cmd_resp_enable();
```

### 14. 重复 esp_insights_init 而不 deinit

```c
// ❌ WRONG — 在某分支重新 init 而不先 deinit，资源泄漏/二次注册 transport
esp_insights_init(&config);
// ... 后来想换配置 ...
esp_insights_init(&config);  // 重复

// ✅ CORRECT — 先 deinit 再重启，或用 enable/disable 切换
esp_insights_deinit();
esp_insights_init(&new_config);
```

---

## Execution Workflow

| 步骤 | 名称 | 描述 |
|------|------|------|
| 1 | 理解需求 | 确认采集类型（日志/事件/metrics/variables/core dump）、传输（HTTPS/MQTT/自定义）、芯片型号 |
| 2 | 选配方 | 在 `recipes/` 匹配场景，先读对应配方的调用链与前置条件 |
| 3 | 查 API | 配方未覆盖的接口，查 `resources/api_reference.md`（按真实头文件分组） |
| 4 | 查配置 | Kconfig 选项查 `resources/config_reference.md`；分区表/`sdkconfig.defaults` 参照 `resources/example_list.md` |
| 5 | 校验陷阱 | 对照 `## Critical Pitfalls` 与 `resources/pitfalls.md` 逐条排查 |
| 6 | 呈现方案 | 向用户说明：依赖、include、init 顺序、`log_type`、Kconfig、分区表 |
| 7 | 执行 | 新工程：拷贝最近 example（`minimal_diagnostics` 或 `diagnostics_smoke_test`）后改；已有工程：原地编辑 |
| 8 | 构建 | `idf.py set-target <chip>` → `idf.py menuconfig` → `idf.py build` |
| 9 | 烧录监控 | `idf.py -p <port> erase_flash flash monitor`；记录启动日志中的 `Insights enabled for Node ID ...` |
| 10 | 仪表盘 | 登录 dashboard.insights.espressif.com（HTTPS）或 dashboard.rainmaker.espressif.com（MQTT）；上传固件包；按 Node ID 查看 |

### 步骤 7 细节 — 工程创建策略

**目标目录无既有工程（首次创建）：**

1. 按 user 需求选最近 example：
   - 最小集成（HTTPS）→ `examples/minimal_diagnostics`
   - 端到端冒烟（含崩溃/metrics/事件）→ `examples/diagnostics_smoke_test`
   - MQTT + Claiming → `examples/minimal_diagnostics`（按其 README “ESP Insights Over MQTT” 段配置）
   - 自定义 transport → 参考 RainMaker `app_insights.c` 模式 + 本 skill `recipes/custom_transport.md`
2. 复制整目录（含 `main/`、`partitions.csv`、`sdkconfig.defaults`、`CMakeLists.txt`）。
3. 按需求改：`app_main.c` 的 `log_type`、自定义 metrics/variables、Kconfig、分区表。
4. 向用户说明复制了什么、改了什么。

**目标目录已有工程：** 原地编辑，勿覆盖既有文件。

---

## Failure Strategies

| 情形 | 处置 |
|---|---|
| API 在 `resources/api_reference.md` 查不到 | 停下，告知用户该接口不存在；勿臆造 |
| 不确定传输选型 | 默认 HTTPS（需 Auth Key）；已在用 RainMaker 则选 MQTT 复用连接省内存 |
| core dump 不上报 | 核对三件套：`ENABLE_COREDUMP_TO_FLASH` + `DATA_FORMAT_ELF` + `coredump` 分区 + `ESP_INSIGHTS_COREDUMP_ENABLE` |
| 仪表盘显示 Firmware Image missing | 重新构建并上传 `build/<project>-<ver>.zip`，确保与板上二进制一致 |
| RTC 存储频繁满 | 监听 `*_LOW_MEM` 事件；调小日志量或调大 `CONFIG_RTC_STORE_CRITICAL_DATA_SIZE`；手动 `esp_insights_send_data()` |
| metadata 1.0/2.0 不确定 | 看是否调用含 `tag` 形参的 `report_*` API；不确定则默认 1.0（`add_*`） |
| error/warn 仪表盘看不到 | 检查 `Default log verbosity` 是否被设为 No output；改用 console channel=None |
| 想停上报但保留 transport | `esp_insights_reporting_disable()`（meta/boot 消息仍发）；彻底停用 `esp_insights_disable()` |

## References

- 场景配方 → `recipes/` 目录
- API 速查（按头文件分组）→ `resources/api_reference.md`
- Kconfig 配置项 → `resources/config_reference.md`
- 常见陷阱汇总 → `resources/pitfalls.md`
- 真实 example 索引 → `resources/example_list.md`
- 仓库本体 → `D:/esp-skill/espressif-repos/esp-insights`
