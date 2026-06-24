# Wi-Fi iTWT（Individual Target Wake Time，Wi-Fi 6 省电）

> **适用摘要**: 在 host 上经 ESP-Hosted 协商 iTWT（802.11ax 个别目标唤醒时间），让 STA 与 AP 协商自己的唤醒/睡眠周期以降低 Wi-Fi 空口功耗。仅支持 Wi-Fi 6 协处理器（ESP32-C5 / C6），STA 模式，工作在 modem-sleep（默认）或 light-sleep（规划中）。

## 触发意图

- "ESP-Hosted iTWT"
- "Wi-Fi 6 省电"
- "esp_wifi_sta_itwt_setup"
- "TWT setup / teardown / suspend"
- "C6 / C5 目标唤醒时间"

## 前置条件

| 条件 | 要求 |
|---|---|
| 协处理器 | ESP32-C5 或 ESP32-C6（支持 Wi-Fi 6 / HE） |
| Host | 例程验证于 ESP32-P4 |
| AP | 必须支持 iTWT 能力，且 STA 协商到 HE20 物理模式 |
| 头文件 | `#include "esp_wifi_he.h"`（iTWT 结构体/事件/错误码）、`#include "esp_wifi.h"` |
| Wi-Fi 模式 | STA，`WIFI_PROTOCOL_11AX`，`WIFI_PS_MIN_MODEM` |
| 参考例程 | `examples/host_wifi_itwt/` |

> iTWT 与 host deep sleep（见 `host_power_save.md`）是两件事：iTWT 调度的是 **Wi-Fi 空口** 的醒/睡（modem-sleep），host deep sleep 调度的是 **host MCU** 的睡眠。两者可叠加。

## 分步说明

### 1. 例程 Kconfig（`examples/host_wifi_itwt/main/Kconfig.projbuild`）

```text
Example Configuration
├── EXAMPLE_WIFI_SSID / EXAMPLE_WIFI_PASSWORD
├── EXAMPLE_ENABLE_STATIC_IP  (+ STATIC_IP_ADDR / NETMASK / GW_ADDR)
├── EXAMPLE_TWT_ENABLE_KEEP_ALIVE_QOS_NULL   # TWT 期间发 QOS NULL 保活
└── iTWT Configuration
    ├── EXAMPLE_ITWT_TRIGGER_ENABLE   # 1=trigger-enabled, 0=non-trigger
    ├── EXAMPLE_ITWT_ANNOUNCED        # 1=announced, 0=unannounced
    ├── EXAMPLE_ITWT_MIN_WAKE_DURA    # [1,255]，单位 256us
    ├── EXAMPLE_ITWT_WAKE_DURATION_UNIT # 0=256us, 1=TU(1024us)
    ├── EXAMPLE_ITWT_WAKE_INVL_EXPN   # [0,31]
    ├── EXAMPLE_ITWT_WAKE_INVL_MANT   # [1,65535]
    ├── EXAMPLE_ITWT_ID               # [0,32767]
    └── EXAMPLE_ITWT_SETUP_TIMEOUT_TIME_MS  # [100,65535]
```

> TWT Wake Interval = Mantissa × (2 ^ Exponent)。

### 2. setup 配置结构体（节选自 `itwt_main.c`）

ESP-IDF v5.3.1 以下用 `wifi_twt_setup_config_t`，以上用 `wifi_itwt_setup_config_t`（字段一致）：

```c
#if ESP_IDF_VERSION > ESP_IDF_VERSION_VAL(5, 3, 0)
    wifi_itwt_setup_config_t setup_config
#else
    wifi_twt_setup_config_t setup_config
#endif
    = {
        .setup_cmd         = TWT_REQUEST,
        .flow_id           = 0,
        .twt_id            = CONFIG_EXAMPLE_ITWT_ID,
        .flow_type         = flow_type_announced ? 0 : 1,   // 0=announced
        .min_wake_dura     = CONFIG_EXAMPLE_ITWT_MIN_WAKE_DURA,
        .wake_duration_unit= CONFIG_EXAMPLE_ITWT_WAKE_DURATION_UNIT,
        .wake_invl_expn    = CONFIG_EXAMPLE_ITWT_WAKE_INVL_EXPN,
        .wake_invl_mant    = CONFIG_EXAMPLE_ITWT_WAKE_INVL_MANT,
        .trigger           = trigger_enabled,
        .timeout_time_ms   = CONFIG_EXAMPLE_ITWT_SETUP_TIMEOUT_TIME_MS,
    };
```

### 3. 协商 iTWT（在 GOT_IP 后）

```c
static void got_ip_handler(void *arg, esp_event_base_t base,
                           int32_t id, void *data)
{
    wifi_phy_mode_t phymode;
    esp_wifi_sta_get_negotiated_phymode(&phymode);
    if (phymode != WIFI_PHY_MODE_HE20) {
        ESP_LOGE(TAG, "Must be in 11ax mode to support itwt");
        return;
    }
    /* 填 setup_config（见上）后调用： */
    esp_err_t err = esp_wifi_sta_itwt_setup(&setup_config);
    if (err != ESP_OK) {
        ESP_LOGE(TAG, "itwt setup failed: %s", esp_err_to_name(err));
    }
}
```

### 4. 关键 Wi-Fi 设置（强制 HE + MIN_MODEM）

```c
esp_wifi_set_protocol(WIFI_IF_STA,
    WIFI_PROTOCOL_11B | WIFI_PROTOCOL_11G | WIFI_PROTOCOL_11N | WIFI_PROTOCOL_11AX);
esp_wifi_set_bandwidth(WIFI_IF_STA, WIFI_BW20);
esp_wifi_set_ps(WIFI_PS_MIN_MODEM);    // modem-sleep（默认支持模式）

#if ESP_IDF_VERSION > ESP_IDF_VERSION_VAL(5, 3, 0)
    wifi_twt_config_t twt_cfg = {
        .post_wakeup_event = false,
    #if ESP_IDF_VERSION > ESP_IDF_VERSION_VAL(5, 3, 1)
        .twt_enable_keep_alive = keep_alive_enabled,
    #endif
    };
    esp_wifi_sta_twt_config(&twt_cfg);
#endif
```

### 5. iTWT 事件（全部需注册）

| 事件 | 携带结构体 | 含义 |
|---|---|---|
| `WIFI_EVENT_ITWT_SETUP` | `wifi_event_sta_itwt_setup_t` | setup 成功（`status==1`）/ 超时 / 发送失败 / 被拒 |
| `WIFI_EVENT_ITWT_TEARDOWN` | `wifi_event_sta_itwt_teardown_t` | 拆除 flow（`flow_id==8` 表示全部） |
| `WIFI_EVENT_ITWT_SUSPEND` | `wifi_event_sta_itwt_suspend_t` | 挂起（含每 flow 实际挂起时长） |
| `WIFI_EVENT_ITWT_PROBE` | `wifi_event_sta_itwt_probe_t` | probe 状态：`ITWT_PROBE_FAIL/SUCCESS/TIMEOUT/STA_DISCONNECTED` |

setup 成功时的关键字段（来自 `itwt_main.c` 日志）：
```text
target wake time: <us>, wake duration: <min_wake_dura << (unit==1 ? 10 : 8)> us,
service period:   <wake_invl_mant << wake_invl_expn> us
```

### 6. 运行期控制台命令（modem-sleep 模式）

例程 `wifi_itwt_cmd.c` 注册了 `itwt>` 控制台命令（需未启用 `CONFIG_PM_ENABLE` 动态调频时才有 REPL）：

```text
itwt --setup <cmd> [--trigger 0|1] [--flowtype 0|1] [--wakeinvlexp N]
     [--wakeduraunit 0|1] [--wakeinvlman N] [--minwakedur N]
     [--flowid N] [--twtid N] [--setup_timeout_time_ms N]
itwt --teardown [--flowid N] [--all_twt N]
itwt --suspend [--suspend_time_ms N] [--all_twt N]
itwt --getflowid
itwt --waketimeoffset <offset>
probe --timeout <ms>     # 发 probe request 同步 TSF
```

参数范围与上面 Kconfig 一致（mant [1,65535]、expn [0,31]、min_wake_dura [1,255] 等）。

### 7. iTWT 相关 RPC（host → slave，见 `docs/implemented_rpcs.md`）

| RPC | 命令 ID | 自 |
|---|---|---|
| WifiStaItwtSetup | 355 | 2.2.2 |
| WifiStaItwtTeardown | 356 | 2.2.2 |
| WifiStaItwtSuspend | 357 | 2.2.2 |
| WifiStaItwtGetFlowIdStatus | 358 | 2.2.2 |
| WifiStaItwtSendProbeReq | 359 | 2.2.2 |
| WifiStaItwtSetTargetWakeTimeOffset | 360 | 2.2.2 |

## 常见错误

| 错误 | 原因 | 解决方法 |
|---|---|---|
| "Must be in 11ax mode" | 未协商到 HE20 | AP 需支持 802.11ax；确认 protocol 含 `WIFI_PROTOCOL_11AX`、带宽 HE20 |
| setup 超时 (`ESP_ERR_WIFI_TWT_SETUP_TIMEOUT`) | AP 不支持 iTWT 或响应慢 | 换支持 iTWT 的 AP；提高 `setup_timeout_time_ms` |
| setup 被拒 (`ESP_ERR_WIFI_TWT_SETUP_REJECT`) | AP 拒绝该 TWT 参数 | 调整 trigger / flow_type / 间隔 |
| 非 C5/C6 报无 iTWT | 仅 Wi-Fi 6 协处理器支持 | 换 ESP32-C6 / C5 |
| 控制台无 `itwt>` 命令 | 启用了 `CONFIG_PM_ENABLE`（走 PM 分支，无 REPL） | 关 PM 或直接在代码里调 API |
| 电流未下降 | 未真正进 modem-sleep / interval 太短 | `WIFI_PS_MIN_MODEM`，核对 wake interval/duration |
| IDF < 5.3.1 编译失败 | `wifi_itwt_setup_config_t` 仅 5.3.1+ 才有 | 用条件编译回落到 `wifi_twt_setup_config_t` |

## 参考项目

- `examples/host_wifi_itwt/` — iTWT 完整例程（ESP32-P4 host，C5/C6 协处理器）
- `examples/host_wifi_itwt/main/itwt_main.c` — setup 流程、事件处理、PM 配置
- `examples/host_wifi_itwt/main/wifi_itwt_cmd.c` — `itwt` / `probe` 控制台命令注册
- `examples/host_wifi_itwt/main/Kconfig.projbuild` — 全部 iTWT Kconfig 符号与取值范围
- `docs/features.md`（iTWT 段）
- `docs/implemented_rpcs.md`（WifiStaItwt* RPC 列表，v2.2.2 起）
