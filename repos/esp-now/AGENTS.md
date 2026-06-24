# AGENTS.md — Supplementary Agent Guide

> 核心规则、场景索引、陷阱与执行流程均在 `SKILL.md`。
> 本文件只补充 `SKILL.md` 未涉及的约定与工具链指引，不重复内容。

## Project Context

**Language**: C · **Target**: 乐鑫 ESP32 系列 SoC（ESP32/ESP32-C2/C3/C6/S2/S3） · **Toolchain**: ESP-IDF (>= v4.4) + `idf.py` · **Component**: `espressif/esp-now`（v2.5.x，通过 ESP Component Registry 下发）

## 添加组件依赖

在 ESP-IDF 工程根目录：

```shell
idf.py add-dependency "espressif/esp-now=*"
```

这会向 `idf_component.yml` 写入依赖，CMake 阶段自动下载。不要手动复制 `src/` 到工程。

## 文件命名与包含

- 头文件统一位于 `src/<module>/include/`，包含时只用短名：
  - `#include "espnow.h"` — 核心收发、配置、分组、密钥
  - `#include "espnow_ctrl.h"` — 设备控制与绑定
  - `#include "espnow_ota.h"` — 批量 OTA
  - `#include "espnow_security.h"` + `"espnow_security_handshake.h"` — 安全
  - `#include "espnow_prov.h"` — Wi-Fi 配网
  - `#include "espnow_log.h"` / `"espnow_console.h"` / `"espnow_cmd.h"` — 调��
  - `#include "espnow_time.h"` — 节点间时间同步
  - `#include "espnow_storage.h"` / `"espnow_mem.h"` / `"espnow_utils.h"` — 工具
- 示例 app 入口统一命名 `main/app_main.c`，入口函数 `void app_main(void)`（ESP-IDF 约定）。

## 标准 ESP-NOW 工程结构

```
MyEspnowProject/
├── CMakeLists.txt
├── idf_component.yml          # 依赖 espressif/esp-now
├── sdkconfig                  # idf.py menuconfig 生成（含 ESP-NOW Configuration）
├── main/
│   ├── CMakeLists.txt
│   └── app_main.c             # app_main() 入口
└── managed_components/        # 自动下载，含 espressif__esp-now
    └── espressif__esp-now/
        ├── src/
        ├── include/(符号链接到 src/*/include)
        └── Kconfig
```

> `managed_components/` 与 `sdkconfig` 由工具生成，不要手动编辑。

## 标准 app_main 模式（参考 examples/*/main/app_main.c）

```c
#include "espnow.h"
#include "espnow_storage.h"
#include "espnow_utils.h"
#include "esp_wifi.h"

static void app_wifi_init(void)
{
    esp_event_loop_create_default();
    wifi_init_config_t cfg = WIFI_INIT_CONFIG_DEFAULT();
    ESP_ERROR_CHECK(esp_wifi_init(&cfg));
    ESP_ERROR_CHECK(esp_wifi_set_mode(WIFI_MODE_STA));
    ESP_ERROR_CHECK(esp_wifi_set_storage(WIFI_STORAGE_RAM));
    ESP_ERROR_CHECK(esp_wifi_set_ps(WIFI_PS_NONE));
    ESP_ERROR_CHECK(esp_wifi_start());
}

void app_main(void)
{
    espnow_storage_init();
    app_wifi_init();

    espnow_config_t espnow_config = ESPNOW_INIT_CONFIG_DEFAULT();
    // 按需开启: espnow_config.sec_enable = 1; receive_enable.xxx = 1;
    espnow_init(&espnow_config);

    // 注册接收回调 / 启动 control / ota / sec / prov 等模块
    espnow_set_config_for_data_type(ESPNOW_DATA_TYPE_DATA, true, app_recv_cb);
}
```

## 通用错误处理宏（来自 espnow_utils.h）

```c
ESP_PARAM_CHECK(con)          // 参数校验，失败返回 ESP_ERR_INVALID_ARG
ESP_ERROR_CHECK(err)          // 失败 abort
ESP_ERROR_RETURN(con, err, fmt, ...)  // 失败打印并 return err
ESP_ERROR_GOTO(con, label, fmt, ...)  // 失败 goto
ESP_ERROR_CONTINUE(con, fmt, ...)     // 失败 continue（循环内）
ESP_ERROR_BREAK(con, fmt, ...)        // 失败 break
```

## 内存宏（来自 espnow_mem.h，带调试记录）

```c
ESP_MALLOC(size)        // 等价 heap_caps_malloc，CONFIG_ESPNOW_MEM_DEBUG 时记录
ESP_CALLOC(n, size)
ESP_REALLOC(ptr, size)
ESP_REALLOC_RETRY(ptr, size)  // 失败重试直到成功
ESP_FREE(ptr)           // 释放并置 NULL
```

> 开启 `CONFIG_ESPNOW_MEM_DEBUG=y` 后，用 `espnow_mem_print_record()` 打印未释放的分配；用 `espnow_mem_print_heap()` / `espnow_mem_print_task()` 看堆与任务状态。

## 构建工作流

```shell
# 1. 设置目标芯片（必做，首次或换芯片）
idf.py set-target esp32c3      # 或 esp32 / esp32s3 / esp32c2 / esp32c6 / esp32s2

# 2. 配置（含 Component config → ESP-NOW Configuration）
idf.py menuconfig

# 3. 构建
idf.py build

# 4. 烧录（首次建议先擦除）
idf.py erase_flash
idf.py flash monitor
```

`ESP-NOW Configuration` 菜单关键项见 `resources/config_reference.md`（安全、light sleep、control 自动信道、OTA 重传、任务栈/优先级、NVS namespace、调试日志分区等）。

## 代码生成 Checklist

- [ ] `idf_component.yml` 含 `espressif/esp-now` 依赖
- [ ] `app_main` 顺序：`espnow_storage_init` → `app_wifi_init`（含 `esp_wifi_start`）→ `espnow_init`
- [ ] Wi-Fi 设为 `WIFI_MODE_STA` + `WIFI_STORAGE_RAM` + `WIFI_PS_NONE`
- [ ] 需要接收的数据类型已 `espnow_set_config_for_data_type(type, true, cb)` 或在 `receive_enable` 里开启
- [ ] `espnow_send` 的 `size` ≤ `ESPNOW_DATA_LEN`；加密时帧头 `security=true`
- [ ] 安全场景：`sec_enable=1` + `espnow_set_key` 与 `espnow_set_dec_key` 都调；监听 `ESP_EVENT_ESPNOW_SEC_OK/_FAIL`
- [ ] OTA initiator：固件已 `esp_ota_write` 到本地 `next update partition`，回调内用 `esp_partition_read`
- [ ] 所有 `*_scan` / `*_result` 内存配对释放：`*_scan_result_free`、`*_result_free`、`ESP_FREE`
- [ ] 控制绑定：responder `espnow_ctrl_responder_bind(wait_ms, rssi, cb)` 进入窗口后再触发 initiator `espnow_ctrl_initiator_bind`
- [ ] 硬币电池方案：`CONFIG_ESPNOW_LIGHT_SLEEP=y`，发包前 light sleep，`esp_now_set_wake_window(0)`
- [ ] Kconfig 任务栈（`CONFIG_ESPNOW_TASK_STACK_SIZE` 默认 4096）够用；回调内不做重活，移交应用任务

## Do Not Modify

- `managed_components/espressif__esp-now/src/` — 组件源码，由组件管理器维护
- `managed_components/espressif__esp-now/Kconfig` — 配置定义
- `sdkconfig` 中 ESP-NOW 段请通过 `menuconfig` 修改，不要手编
