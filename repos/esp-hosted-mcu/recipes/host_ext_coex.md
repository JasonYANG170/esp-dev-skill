# 外部共存 EXT_COEX（从 host 配置协处理器 PTA）

> **适用摘要**: 通过 ESP-Hosted 链路远程配置协处理器（slave）的硬件 PTA（Packet Traffic Arbitrator），让 slave 的 Wi-Fi 与 host 上的另一颗外部射频（BLE/Zigbee/Thread 等）共享 2.4 GHz 频段。支持 1/2/3/4-wire 模式与 leader/follower 角色。host 侧平台无关，亦适用于非 ESP host（STM32/nRF）。

> Version: selected repo/component version; align with the user project and dependency manifest.
> Evidence: `repos/esp-hosted-mcu/resources/`, source/examples in `repos/esp-hosted-mcu/`, and this recipe path `repos/esp-hosted-mcu/recipes/host_ext_coex.md`.
> Validation: draft metadata added from repository routing; verify APIs, Kconfig symbols, and component dependencies against the selected version.

## 触发意图

- "ESP-Hosted 外部共存"
- "EXT_COEX 配置"
- "esp_hosted_cp_ext_coex_set_work_mode"
- "slave Wi-Fi 与 host BLE/Zigbee 共存"
- "PTA leader / follower"
- "grant delay / validate high"

## 前置条件

| 条件 | 要求 |
|---|---|
| 链路 | 已 `esp_hosted_init()` + `esp_hosted_connect_to_slave()` 成功 |
| 头文件 | `#include "esp_hosted.h"`（含 `esp_hosted_cp_ext_coex.h`） |
| Host Kconfig | `Component config -> ESP-Hosted config -> [*] Co-Processor External Coexistence`（进阶选项 `[*] Co-Processor External Coexistence - Advanced`） |
| Slave Kconfig | `Example Configuration -> [*] External Coexistence support` |
| Slave 芯片 | 仅 ESP32-C6 / C61 / C5 / S3（ESP32 不支持） |
| 协处理器 GPIO | request/priority/grant/tx_line 引脚未被传输/RF 占用 |
| 参考例程 | `examples/host_manage_copro_ext_coex/` |

> **关键限制（务必先读）**：在 **advanced-coex 芯片**（`SOC_EXTERNAL_COEX_ADVANCE` — C2/C5/C6/C61/H2/H4/H21）上 EXT_COEX 可与 BT/BLE 共存；在其它芯片（如 **ESP32-S3**）上启用 EXT_COEX 时**必须禁用 BT controller**。且 EXT_COEX + BT 同时启用目前需要 **ESP-IDF `master`**（稳定 tag 暂不含此支持）。

## 分步说明

### 1. 双方 menuconfig（决定 BT 是否能共存）

**Advanced-coex 芯片（C6/C61/C5）— BT/BLE 可保留**：
```text
# slave menuconfig
├── Component config → Bluetooth
│    └── [*] Bluetooth                          # 需 BLE 时保留
└── Example Configuration
     ├── [*] Enable BT sharing via hosted       # 需 BLE 时保留
     └── [*] External Coexistence support       # 启用 ✔
```

**其它芯片（如 ESP32-S3）— 必须禁用 BT**：
```text
├── Component config ��� Bluetooth
│    └── [ ] Bluetooth                          # 禁用 ✘
└── Example Configuration
     ├── [ ] Enable BT sharing via hosted       # 禁用 ✘
     └── [*] External Coexistence support       # 启用 ✔
```

**Host 侧**：
```text
Component config → ESP-Hosted config
    ├── [*] Co-Processor External Coexistence                # CONFIG_ESP_HOSTED_CP_EXT_COEX
    └── [*] Co-Processor External Coexistence - Advanced     # CONFIG_ESP_HOSTED_CP_EXT_COEX_ADVANCE（grant_delay / validate_high）
```

### 2. 配置结构体与模式宏（来自 `host/api/include/esp_hosted_cp_ext_coex.h`）

```c
// wire 数量（set_gpio_pin 的 wire_type 参数）
#define ESP_HOSTED_EXT_COEX_WIRE_1 0     // 仅 Request：peer 需要频段时拉高，Wi-Fi 让出
#define ESP_HOSTED_EXT_COEX_WIRE_2 1     // + Grant：PTA 仲裁后给出授权电平
#define ESP_HOSTED_EXT_COEX_WIRE_3 2     // + Priority：peer 申报中/高优先级
#define ESP_HOSTED_EXT_COEX_WIRE_4 3

typedef enum {
    ESP_HOSTED_EXT_COEX_LEADER_ROLE   = 0,   // host 领导仲裁
    ESP_HOSTED_EXT_COEX_FOLLOWER_ROLE = 2,   // host 跟随
    ESP_HOSTED_EXT_COEX_UNKNOWN_ROLE,        // 未指定
} esp_hosted_ext_coex_work_mode_t;

typedef struct {
    int32_t request;    // Request 信号 GPIO（slave 侧）
    int32_t priority;   // Priority 信号 GPIO
    int32_t grant;      // Grant 信号 GPIO
    int32_t tx_line;    // TX Line 信号 GPIO
} esp_hosted_ext_coex_gpio_set_t;
```

### 3. API 一览（受 `CONFIG_ESP_HOSTED_CP_EXT_COEX` / `_ADVANCE` 条件编译）

```c
// 基础（_CP_EXT_COEX 开启）
esp_err_t esp_hosted_cp_ext_coex_set_work_mode(esp_hosted_ext_coex_work_mode_t work_mode);
esp_err_t esp_hosted_cp_ext_coex_set_gpio_pin(uint32_t wire_type,
                                              const esp_hosted_ext_coex_gpio_set_t *gpio_pins);
esp_err_t esp_hosted_cp_ext_coex_disable(void);   // 释放 GPIO

// 进阶（_CP_EXT_COEX_ADVANCE 额外开启）
esp_err_t esp_hosted_cp_ext_coex_set_grant_delay(uint8_t delay_us);   // GRANT 延时（微秒）
esp_err_t esp_hosted_cp_ext_coex_set_validate_high(bool is_high_valid); // validate 极性
```

### 4. Host ↔ Slave API 映射（RPC 转发到 slave 的原生 ext_coex API）

| Host API | Slave 端原生 API | 作用 |
|---|---|---|
| `esp_hosted_cp_ext_coex_set_work_mode()` | `esp_external_coex_set_work_mode` | 设置 leader/follower |
| `esp_hosted_cp_ext_coex_set_gpio_pin()` | `esp_enable_extern_coex_gpio_pin` | 配置 REQ/PRI/GRANT/TX GPIO |
| `esp_hosted_cp_ext_coex_set_grant_delay()` | `esp_external_coex_set_grant_delay` | 调整 GRANT 时序 |
| `esp_hosted_cp_ext_coex_set_validate_high()` | `esp_external_coex_set_validate_high` | validate 极性 |
| `esp_hosted_cp_ext_coex_disable()` | `esp_disable_extern_coex_gpio_pin` | 关闭并释放 GPIO |

### 5. 典型用法（改编自 `examples/host_manage_copro_ext_coex/main/main.c`）

```c
#include "esp_log.h"
#include "nvs_flash.h"
#include "esp_hosted.h"

static const char *TAG = "ext_coex";

void app_main(void)
{
    ESP_ERROR_CHECK(nvs_flash_init());

    esp_hosted_init();
    esp_hosted_connect_to_slave();

#if CONFIG_ESP_HOSTED_CP_EXT_COEX
    /* a. 角色：Leader */
    esp_err_t err = esp_hosted_cp_ext_coex_set_work_mode(ESP_HOSTED_EXT_COEX_LEADER_ROLE);
    if (err != ESP_OK) {
        ESP_LOGE(TAG, "set work mode failed: %s", esp_err_to_name(err));
    }

    /* b. GPIO：3-wire（Request/Priority/Grant/TX Line）。
     *    这里 GPIO 号为协处理器侧引脚，需替换为实际可用引脚 */
    esp_hosted_ext_coex_gpio_set_t pins = {
        .request  = 11,
        .priority = 4,
        .grant    = 16,
        .tx_line  = 17,
    };
    err = esp_hosted_cp_ext_coex_set_gpio_pin(ESP_HOSTED_EXT_COEX_WIRE_3, &pins);
    if (err != ESP_OK) {
        ESP_LOGE(TAG, "set gpio pin failed: %s", esp_err_to_name(err));
    }

#if CONFIG_ESP_HOSTED_CP_EXT_COEX_ADVANCE
    /* c. 进阶：GRANT 延时 10 us + validate 高有效 */
    esp_hosted_cp_ext_coex_set_grant_delay(10);
    esp_hosted_cp_ext_coex_set_validate_high(true);
#endif

    /* 用完后（可选）：释放 GPIO */
    /* esp_hosted_cp_ext_coex_disable(); */
#else
    ESP_LOGW(TAG, "CONFIG_ESP_HOSTED_CP_EXT_COEX is not enabled.");
#endif
}
```

### 6. 1/2/3-wire 信号语义速查

| 模式 | 信号 | 行为 |
|---|---|---|
| 1-wire | Request | peer 拉高即请求频段，slave Wi-Fi 总是让出（简单，影响 Wi-Fi 性能） |
| 2-wire | + Grant | PTA 仲裁，Grant 高=peer 可发，Grant 低=Wi-Fi 可发 |
| 3-wire | + Priority | peer 申报优先级，PTA 与内部 Wi-Fi 优先级比较后决定 |
| 4-wire | + TX Line | 见应用笔记 |

## 常见错误

| 错误 | 原因 | 解决方法 |
|---|---|---|
| `set_work_mode` 返回错误 | host 未开 `CONFIG_ESP_HOSTED_CP_EXT_COEX` | host menuconfig 启用 External Coexistence |
| ESP32-S3 上 EXT_COEX + BT 同时启用失败 | 非 advanced-coex 芯片不能共存 | 禁用 slave 的 BT，或换 C5/C6/C61 |
| BT + EXT_COEX 在稳定 IDF 上不工作 | 该组合当前仅 ESP-IDF `master` 支持 | 切到 IDF master 测试 |
| 共存无效果 / GPIO 冲突 | 引脚与传输/RF/strapping 重叠 | 选协处理器未占用 GPIO；避开 strapping |
| `set_grant_delay` 找不到符号 | 未开 Advanced 选项 | host 启用 `Co-Processor External Coexistence - Advanced` |
| 例程警告 "not enabled" | Kconfig 未生效就编译 | menuconfig 后重新 `idf.py build` |
| ESP32 作协处理器 | 不支持 EXT_COEX | 换 C6/C61/C5/S3 |

## 参考项目

- `examples/host_manage_copro_ext_coex/` — 从 host 配置 slave EXT_COEX 的完整例程（含 main.c）
- `host/api/include/esp_hosted_cp_ext_coex.h` — 全部结构与 API（条件编译）
- `docs/features.md`（External Coexistence 段）
- 应用笔记：[External Coexistence Design](https://documentation.espressif.com/external_coexistence_design_en.pdf)
