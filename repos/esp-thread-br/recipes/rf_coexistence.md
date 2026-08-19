# RF External Coexistence（Wi-Fi ↔ 802.15.4）

> **适用摘要**: 在 ESP32-S3（Wi-Fi）与 ESP32-H2（802.15.4 RCP）双芯片 BR 上启用外部 RF 共存，用 3 线或 4 线握手信号降低同频段干扰；重点说明何时有用、3/4 线差异、以及两端 Kconfig 必须同时开启。

> Version: selected repo/component version; align with the user project and dependency manifest.
> Evidence: `repos/esp-thread-br/resources/`, source/examples in `repos/esp-thread-br/`, and this recipe path `repos/esp-thread-br/recipes/rf_coexistence.md`.
> Validation: draft metadata added from repository routing; verify APIs, Kconfig symbols, and component dependencies against the selected version.

## 触发意图
- "RF 共存"
- "Wi-Fi 与 802.15.4 干扰"
- "external coexistence"
- "ot_external_coexist_init"
- "3-wire / 4-wire coex"
- "ESP32-H2 TX while Wi-Fi RX"

## 前置条件

| 条件 | 要求 |
|---|---|
| 硬件 | ESP Thread Border Router 板（ESP32-S3 + ESP32-H2），两颗芯片的共存引脚已正确连到对方 |
| 适用场景 | Wi-Fi 与 802.15.4 工作在**相近信道频率**（干扰显著时）才有效；否则无收益 |
| Kconfig（两端） | BR 端：`CONFIG_ESP_COEX_EXTERNAL_COEXIST_ENABLE=y`；ot_rcp 端：同样开启（见步骤 2） |
| 引脚 | 通过 menuconfig `ESP Thread Border Router Example → External coexist wire type and pin config` 配置（官方 BR 板默认已设） |
| 参考 | `docs/en/dev-guide/build_and_run.rst`（2.1.3.5）、`docs/en/hardware_platforms.rst` |

> 重要前提（来自 build_and_run.rst 2.1.3.5）：外部共存**仅在 Wi-Fi 与 802.15.4 信道频率相近、干扰明显时才有帮助**。若两者信道相距较远，开启此特性无意义。

## 分步说明

### 1. 理解 3 线 vs 4 线

| 模式 | 信号 | 说明 |
|---|---|---|
| 3-wire | `request` + `priority` + `grant` | 经典外部共存：follower 发 request/priority，leader 回 grant |
| 4-wire | 3-wire + `tx_line` | 额外的 `tx_line` 信号由 leader（Wi-Fi SoC）发给 follower，指示 Wi-Fi **是否正在发送**；这样 802.15.4 可在 **Wi-Fi 仅接收**时发送，提升 TX 机会 |

对应枚举与结构体（`esp-idf/components/esp_coex/include/esp_coexist.h`）：
```c
#define EXTERNAL_COEXIST_WIRE_1 0   // request
#define EXTERNAL_COEXIST_WIRE_2 1   // request + grant
#define EXTERNAL_COEXIST_WIRE_3 2   // request + priority + grant
#define EXTERNAL_COEXIST_WIRE_4 3   // 3-wire + tx_line

typedef enum {
    EXTERN_COEX_WIRE_1 = EXTERNAL_COEXIST_WIRE_1,
    EXTERN_COEX_WIRE_2,
    EXTERN_COEX_WIRE_3,
    EXTERN_COEX_WIRE_4,
    EXTERN_COEX_WIRE_NUM,
} external_coex_wire_t;

typedef struct {
    gpio_num_t request;    // follower → leader
    gpio_num_t priority;   // follower → leader
    gpio_num_t grant;      // leader → follower
    gpio_num_t tx_line;    // leader → follower（4-wire 才有）
} esp_external_coex_gpio_set_t;

typedef enum {
    EXTERNAL_COEX_LEADER_ROLE = 0,
    EXTERNAL_COEX_FOLLOWER_ROLE = 2,
    EXTERNAL_COEX_UNKNOWN_ROLE,
} esp_extern_coex_work_mode_t;
```

> 在 ESP32-S3(host Wi-Fi)+ESP32-H2(RCP 802.15.4) 的 BR 上：ESP32-S3 是 **Leader**，ESP32-H2 是 **Follower**。

### 2. 两端 Kconfig 必须同时开启

外部共存要求 **BR 端**与 **ot_rcp 端**镜像都启用，否则两颗芯片握手不上。

**ot_rcp 端**（`esp-idf/examples/openthread/ot_rcp`）——参考 `sdkconfig.ci.ext_coex`：
```
CONFIG_ESP_COEX_EXTERNAL_COEXIST_ENABLE=y
CONFIG_ESP_COEX_SW_COEXIST_ENABLE=n
CONFIG_EXTERNAL_COEX_WORK_MODE_FOLLOWER=y
```

**BR 端**（`basic_thread_border_router`）：
```bash
idf.py menuconfig
# Component config → ESP COEX → [*] Enable external coexistence   # CONFIG_ESP_COEX_EXTERNAL_COEXIST_ENABLE
# Config for OpenThread Examples → External coexist wire type and pin config
#   Wire type (3 or 4) 与 request/grant/priority/tx_line GPIO
```

引脚 Kconfig（`esp-idf/examples/openthread/ot_common_components/ot_examples_common/Kconfig.projbuild`）：

| Kconfig | 默认 | 依赖 |
|---|---|---|
| `CONFIG_EXTERNAL_COEX_WIRE_TYPE` | 3 | 范围 0~3 |
| `CONFIG_EXTERNAL_COEX_REQUEST_PIN` | 0 | `WIRE_TYPE >= 0` |
| `CONFIG_EXTERNAL_COEX_GRANT_PIN` | 1 | `WIRE_TYPE >= 1` |
| `CONFIG_EXTERNAL_COEX_PRIORITY_PIN` | 2 | `WIRE_TYPE >= 2` |
| `CONFIG_EXTERNAL_COEX_TX_LINE_PIN` | 3 | `WIRE_TYPE == 3`（4-wire 专用） |

> 官方 BR 板的默认引脚已就绪，仅需勾选 enable；Standalone/自定义板需按原理图改 GPIO。

### 3. BR 启动期自动初始化共存

`examples/common/thread_border_router/src/border_router_launch.c` 在 `launch_openthread_border_router()` 中按宏自动调用初始化：

```c
#if CONFIG_ESP_COEX_EXTERNAL_COEXIST_ENABLE
    ot_external_coexist_init();
#endif
```

`ot_external_coexist_init()` 定义在 `esp-idf/examples/openthread/ot_common_components/ot_examples_common/ot_external_coexist.c`，声明于 `ot_examples_common.h`：

```c
void ot_external_coexist_init(void)
{
    esp_extern_coex_work_mode_t mode =
    #if CONFIG_EXTERNAL_COEX_WORK_MODE_LEADER
        EXTERNAL_COEX_LEADER_ROLE;
    #elif CONFIG_EXTERNAL_COEX_WORK_MODE_FOLLOWER
        EXTERNAL_COEX_FOLLOWER_ROLE;
    #else
        EXTERNAL_COEX_UNKNOWN_ROLE;
    #endif

    esp_external_coex_gpio_set_t gpio_pin = ESP_OPENTHREAD_DEFAULT_EXTERNAL_COEX_CONFIG();
    ESP_ERROR_CHECK(esp_external_coex_set_work_mode(mode));
    ESP_ERROR_CHECK(esp_enable_extern_coex_gpio_pin(CONFIG_EXTERNAL_COEX_WIRE_TYPE, gpio_pin));
}
```

`ESP_OPENTHREAD_DEFAULT_EXTERNAL_COEX_CONFIG()` 按 `WIRE_TYPE` 填充对应引脚字段（`ot_examples_common.h`）：
```c
// WIRE_4 分支示例
#define ESP_OPENTHREAD_DEFAULT_EXTERNAL_COEX_CONFIG()   \
    {                                                   \
        .request  = CONFIG_EXTERNAL_COEX_REQUEST_PIN,   \
        .priority = CONFIG_EXTERNAL_COEX_PRIORITY_PIN,  \
        .grant    = CONFIG_EXTERNAL_COEX_GRANT_PIN,     \
        .tx_line  = CONFIG_EXTERNAL_COEX_TX_LINE_PIN,   \
    }
```

底层 ESP-IDF API（`esp_coexist.h`）：
```c
esp_err_t esp_external_coex_set_work_mode(esp_extern_coex_work_mode_t work_mode);
esp_err_t esp_enable_extern_coex_gpio_pin(external_coex_wire_t wire_type,
                                          esp_external_coex_gpio_set_t gpio_pin);
```

> 这两步在 BR app 中**不需要手写**——`border_router_launch.c` 已包好；自写 `app_main` 直接复用 `launch_openthread_border_router()` 即可。

### 4. 构建、烧录、验证

```bash
# 1) 先构建 ot_rcp（含 EXTERNAL_COEX_ENABLE）
cd $IDF_PATH/examples/openthread/ot_rcp
idf.py set-target esp32h2
idf.py menuconfig   # 开 CONFIG_ESP_COEX_EXTERNAL_COEXIST_ENABLE / FOLLOWER
idf.py build

# 2) 构建 BR（含 EXTERNAL_COEX_ENABLE，与 ot_rcp 引脚/线数一致）
cd esp-thread-br/examples/basic_thread_border_router
idf.py set-target esp32s3
idf.py menuconfig   # 开 CONFIG_ESP_COEX_EXTERNAL_COEXIST_ENABLE，选 wire type 与 pin
idf.py build
idf.py -p PORT flash monitor
```

验证：起网成功后，把 Wi-Fi 与 Thread 信道设到相近频率，对比开/关共存时的吞吐 / 丢包（可用 `iperf` 或 `ot ping`），开启后 802.15.4 在 Wi-Fi 业务期间受干扰应明显减少。

## 常见错误

| 错误 | 原因 | 解决方法 |
|---|---|---|
| 两端都开了但无效果 | Wi-Fi 与 802.15.4 信道相距远，本就无干扰 | 仅在相近信道下启用；否则关闭以省引脚 |
| `ot_external_coexist_init` 报 `ESP_ERR_INVALID_ARG` | 引脚号冲突或未设 | 核对 menuconfig 中 request/grant/priority/tx_line GPIO 与板载其他外设不冲突 |
| 握手不上 / 共存不生效 | 只在 BR 端开，ot_rcp 端漏开 | 两端 `CONFIG_ESP_COEX_EXTERNAL_COEXIST_ENABLE` 都要 `=y`，且 `WIRE_TYPE` 一致 |
| 想让 802.15.4 在 Wi-Fi RX 时 TX 不工作 | 用了 3-wire | 改 `WIRE_TYPE=4` 并接好 `tx_line` 引脚 |
| BR 启动报 coex 引脚冲突 | Leader/Follower 角色设错 | ESP32-S3(SoC Wi-Fi) 应为 Leader，ESP32-H2(802.15.4) 为 Follower |
| `SW_COEXIST_ENABLE` 与 external 冲突 | 同时开了软件共存 | ot_rcp 端设 `CONFIG_ESP_COEX_SW_COEXIST_ENABLE=n`（见 `sdkconfig.ci.ext_coex`） |

## 参考项目
- `docs/en/dev-guide/build_and_run.rst`（2.1.3.5 RF External Coexistence）— 3/4 线说明与启用方式
- `docs/en/hardware_platforms.rst` — 官方 BR 板硬件
- `examples/common/thread_border_router/src/border_router_launch.c` — `#if CONFIG_ESP_COEX_EXTERNAL_COEXIST_ENABLE` → `ot_external_coexist_init()`
- `examples/basic_thread_border_router/` — 默认 BR 示例
- ESP-IDF `examples/openthread/ot_common_components/ot_examples_common/ot_external_coexist.c` — `ot_external_coexist_init()` 实现
- ESP-IDF `examples/openthread/ot_common_components/ot_examples_common/include/ot_examples_common.h` — `ESP_OPENTHREAD_DEFAULT_EXTERNAL_COEX_CONFIG()` 宏
- ESP-IDF `examples/openthread/ot_common_components/ot_examples_common/Kconfig.projbuild` — `EXTERNAL_COEX_WIRE_TYPE` / 引脚 / 工作模式 choice
- ESP-IDF `examples/openthread/ot_rcp/sdkconfig.ci.ext_coex` — ot_rcp 端共存配置参考
- ESP-IDF `components/esp_coex/include/esp_coexist.h` — `external_coex_wire_t` / `esp_external_coex_gpio_set_t` / `esp_extern_coex_work_mode_t` / `esp_external_coex_set_work_mode` / `esp_enable_extern_coex_gpio_pin`
