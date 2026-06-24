# 经典蓝牙 A2DP 音频接收（Sink）+ AVRCP

> **适用摘要**: 用 Bluedroid 实现 A2DP Sink（Advanced Audio Distribution Profile，接收手机推送的音频流）与 AVRCP CT（音量/播放控制）：`esp_a2d_sink_init`、`esp_a2d_register_callback`、`esp_a2d_sink_register_audio_data_callback` 收 PCM 数据、`esp_a2d_sink_connect` 主动连接、`esp_avrc_ct_init` 控制器。**仅 esp32 支持，需 `CONFIG_BT_A2DP_ENABLE`。** 适配自 `examples/bluetooth/bluedroid/classic_bt/a2dp_sink_stream`。

## 触发意图

- "蓝牙音箱 / 蓝牙耳机"
- "A2DP 音频"
- "蓝牙音乐"
- "音频流接收"
- "esp_a2d_sink"
- "AVRCP 音量控制"

## 前置条件

| 条件 | 要求 |
|---|---|
| 目标芯片 | **仅 esp32**（需 Classic BT；其余 SoC 无 Classic BT 不支持 A2DP） |
| Kconfig | `CONFIG_BT_ENABLED=y`、`CONFIG_BT_BLUEDROID_ENABLED=y`、`CONFIG_BT_CLASSIC_ENABLED=y`、`CONFIG_BT_A2DP_ENABLE=y`（menuconfig：`Component config → Bluetooth → Bluedroid Options → A2DP`）；可选 `CONFIG_BT_A2DP_CODEC_AAC_ENABLED` 支持 AAC |
| 硬件 | 需外部 I2S DAC/Codec（如 MAX98357A）播放 PCM 数据；仅 A2DP 解码到 PCM，不含喇叭驱动 |
| 组件 | `bt`、`nvs_flash`、`esp_driver_i2s`（输出 PCM） |
| 头文件 | `esp_bt.h`、`esp_bt_main.h`、`esp_gap_bt_api.h`、`esp_bt_device.h`、`esp_a2dp_api.h`、`esp_avrc_api.h` |
| 参考 | `examples/bluetooth/bluedroid/classic_bt/a2dp_sink_stream`（含内部/外部 codec）、`a2dp_source`（源端发送）、`avrcp_absolute_volume`、`avrcp_ct_metadata` |

> 本 recipe 聚焦 A2DP Sink API 主链路。完整示例 `a2dp_sink_stream` 使用了 `bredr_app_common_init` / `bt_app_work_dispatch` 等任务派发辅助层（见 `examples/bluetooth/bluedroid/classic_bt/common/`），把协议栈回调转到独立任务处理，避免在回调里做重活（I2S 写、解码）。生产代码建议参照该模式。

## 分步说明

### 初始化 + Sink 注册 + 被动等待连接

A2DP Sink 模式下，设备名 + 可被发现后，手机（A2DP Source）主动连接并推送音频。

```c
#include "esp_bt.h"
#include "esp_bt_main.h"
#include "esp_gap_bt_api.h"
#include "esp_bt_device.h"
#include "esp_a2dp_api.h"
#include "esp_avrc_api.h"
#include "nvs_flash.h"
#include "esp_log.h"

#define BT_AV_TAG        "BT_AV"
#define LOCAL_DEV_NAME   "ESP_A2DP_SINK"

/* A2DP 事件回调（建议派发到独立任务，此处仅示意主流程） */
static void bt_app_a2d_cb(esp_a2d_cb_event_t event, esp_a2d_cb_param_t *param)
{
    switch (event) {
    case ESP_A2D_CONNECTION_STATE_EVT:
        ESP_LOGI(BT_AV_TAG, "A2DP connection: %d", param->conn_stat.state);
        /* ESP_A2D_CONNECTION_STATE_CONNECTED / DISCONNECTED */
        break;
    case ESP_A2D_AUDIO_STATE_EVT:
        ESP_LOGI(BT_AV_TAG, "audio state: %d", param->audio_stat.state);
        /* ESP_A2D_AUDIO_STATE_STARTED 时开始把 PCM 送 I2S */
        break;
    case ESP_A2D_AUDIO_CFG_EVT:
        /* 音频配置（采样率、声道）协商完成，据此配 I2S 采样率 */
        ESP_LOGI(BT_AV_TAG, "audio cfg, sample_rate=%lu",
                 (unsigned long)param->audio_cfg.mcc.cie.sbc_info.samp_freq);
        break;
    case ESP_A2D_PROF_STATE_EVT:
        /* A2DP 协议本身初始化完成 */
        break;
    default:
        break;
    }
}

/* 音频数据回调：协议栈把 SBC 解码后的 PCM 送到这里 */
static void bt_app_a2d_data_cb(const uint8_t *data, uint32_t len)
{
    /* 把 PCM 写入 I2S DAC（此处省略 i2s_write 实现） */
    size_t bytes_written = 0;
    /* i2s_write(i2s_tx_handle, data, len, &bytes_written, portMAX_DELAY); */
}

void app_main(void)
{
    esp_err_t ret = nvs_flash_init();
    if (ret == ESP_ERR_NVS_NO_FREE_PAGES || ret == ESP_ERR_NVS_NEW_VERSION_FOUND) {
        ESP_ERROR_CHECK(nvs_flash_erase());
        ret = nvs_flash_init();
    }
    ESP_ERROR_CHECK(ret);

    /* 仅用 Classic BT（A2DP 属经典蓝牙） */
    ESP_ERROR_CHECK(esp_bt_controller_mem_release(ESP_BT_MODE_BLE));
    esp_bt_controller_config_t bt_cfg = BT_CONTROLLER_INIT_CONFIG_DEFAULT();
    ESP_ERROR_CHECK(esp_bt_controller_init(&bt_cfg));
    ESP_ERROR_CHECK(esp_bt_controller_enable(ESP_BT_MODE_CLASSIC_BT));

    esp_bluedroid_config_t bluedroid_cfg = BT_BLUEDROID_INIT_CONFIG_DEFAULT();
    ESP_ERROR_CHECK(esp_bluedroid_init_with_cfg(&bluedroid_cfg));
    ESP_ERROR_CHECK(esp_bluedroid_enable());

    /* 经典蓝牙 GAP：设名 + 可发现可连接（手机才能搜到） */
    esp_bt_gap_set_device_name(LOCAL_DEV_NAME);
    esp_bt_gap_register_callback(bt_app_gap_cb);   /* 处理配对（同 SPP recipe） */

    /* A2DP Sink：注册事件回调 → 初始化 → 注册音频数据回调 */
    esp_a2d_register_callback(bt_app_a2d_cb);
    assert(esp_a2d_sink_init() == ESP_OK);
    esp_a2d_sink_register_audio_data_callback(bt_app_a2d_data_cb);   /* PCM 数据入口 */

    /* 可选：AVRCP Controller，用于音量/播放控制（sink 端作为 CT 控制手机） */
    esp_avrc_ct_register_callback(bt_app_avrc_ct_cb);
    esp_avrc_ct_init();

    /* 设为可被发现、可被连接，等待手机发起 A2DP 连接 */
    esp_bt_gap_set_scan_mode(ESP_BT_CONNECTABLE, ESP_BT_GENERAL_DISCOVERABLE);
}
```

### 主动连接手机（Sink 主动连）

若已知手机蓝牙地址，Sink 可主动发起连接：

```c
esp_bd_addr_t peer_bda = {0xAA, 0xBB, 0xCC, 0xDD, 0xEE, 0xFF};
esp_a2d_sink_connect(peer_bda);   /* 异步，结果在 ESP_A2D_CONNECTION_STATE_EVT */
```

### AVRCP CT：发送播放/暂停/音量命令

```c
/* AVRCP CT 回调：接收手机回传的通知（如播放状态变化、音量变化） */
static void bt_app_avrc_ct_cb(esp_avrc_ct_cb_event_t event, esp_avrc_ct_cb_param_t *param)
{
    /* ESP_AVRC_CT_CONNECTION_STATE_EVT / PASSTHROUGH_RSP_EVT / METADATA_RSP_EVT 等 */
}

/* 主动发送 passthrough 命令（播放、暂停、上下曲） */
/* key_state: ESP_AVRC_PT_CMD_STATE_PRESSED / RELEASED
 * key_code:  ESP_AVRC_PT_CMD_PLAY / PAUSE / STOP / FORWARD / BACKWARD 等 */
esp_avrc_ct_send_passthrough_cmd(0 /* tl */, ESP_AVRC_PT_CMD_PLAY, ESP_AVRC_PT_CMD_STATE_PRESSED);
esp_avrc_ct_send_passthrough_cmd(0, ESP_AVRC_PT_CMD_PLAY, ESP_AVRC_PT_CMD_STATE_RELEASED);

/* 设置绝对音量（0~127） */
esp_avrc_ct_send_set_absolute_volume_cmd(0, 100);
```

### 关键 API

```c
/* esp_a2dp_api.h —— Sink */
esp_err_t esp_a2d_register_callback(esp_a2d_cb_t callback);
esp_err_t esp_a2d_sink_register_audio_data_callback(esp_a2d_sink_audio_data_cb_t callback);  /* PCM 数据 */
esp_err_t esp_a2d_sink_init(void);
esp_err_t esp_a2d_sink_deinit(void);
esp_err_t esp_a2d_sink_connect(esp_bd_addr_t remote_bda);
esp_err_t esp_a2d_sink_disconnect(esp_bd_addr_t remote_bda);
esp_err_t esp_a2d_media_ctrl(esp_a2d_media_ctrl_t ctrl);   /* ESP_A2D_MEDIA_CTRL_START / SUSPEND */
/* esp_a2dp_api.h —— Source（发送端） */
esp_err_t esp_a2d_source_init(void);
esp_err_t esp_a2d_source_connect(esp_bd_addr_t remote_bda);
esp_err_t esp_a2d_source_audio_data_send(esp_a2d_conn_hdl_t conn_hdl, esp_a2d_audio_buff_t *audio_buf);
/* esp_avrc_api.h —— AVRCP Controller（Sink 端作 CT） */
esp_err_t esp_avrc_ct_register_callback(esp_avrc_ct_cb_t callback);
esp_err_t esp_avrc_ct_init(void);
esp_err_t esp_avrc_ct_deinit(void);
esp_err_t esp_avrc_ct_send_passthrough_cmd(uint8_t tl, uint8_t key_code, uint8_t key_state);
esp_err_t esp_avrc_ct_send_set_absolute_volume_cmd(uint8_t tl, uint8_t volume);   /* 0~127 */
esp_err_t esp_avrc_ct_send_metadata_cmd(uint8_t tl, uint8_t attr_mask);
esp_err_t esp_avrc_ct_send_register_notification_cmd(uint8_t tl, uint8_t event_id, uint32_t event_parameter);
```

关键 A2DP 事件：`ESP_A2D_PROF_STATE_EVT`（协议就绪）→ 手机连入触发 `ESP_A2D_CONNECTION_STATE_EVT`（`CONNECTED`）→ 协商完成 `ESP_A2D_AUDIO_CFG_EVT`（拿到采样率，配 I2S）→ `ESP_A2D_AUDIO_STATE_EVT`（`STARTED`，开始收 PCM）→ `audio_data_callback` 持续送 PCM。

> **外部 codec（v5.x 新）**：`CONFIG_BT_A2DP_USE_EXTERNAL_CODEC` 时，用 `esp_a2d_sink_register_stream_endpoint` 注册 SEP + `esp_a2d_sink_register_audio_data_callback`（参数变为带 `esp_a2d_audio_buff_t` 的未解码数据），应用层自行解码。内部 codec（默认 SBC）由协议栈解码为 PCM。本 recipe 以内部 codec 为例。

## 常见错误

| 错误 | 原因 | 解决方法 |
|---|---|---|
| `esp_a2d_sink_init` 链接报 undefined | 未启用 A2DP | menuconfig 开 `Bluedroid Options → A2DP`（`CONFIG_BT_A2DP_ENABLE=y`） |
| 手机搜不到设备 | 未设可发现模式 | `esp_bt_gap_set_scan_mode(ESP_BT_CONNECTABLE, ESP_BT_GENERAL_DISCOVERABLE)` |
| 连上但无声 | I2S 未配或采样率不匹配 | `AUDIO_CFG_EVT` 里读 `cie.sbc_info.samp_freq` 配 I2S；确认 I2S DAC 接线与 MCLK/BCLK/LRCK/DATA |
| 回调里做 I2S 写导致协议栈堵塞 | 回调上下文不适合重活 | 用任务派发（参照示例的 `bt_app_work_dispatch`），回调仅转发到队列 |
| AAC 音乐播放失败 | 未启用 AAC codec | 开 `CONFIG_BT_A2DP_CODEC_AAC_ENABLED`（占用更多 flash） |
| 音质差/断续 | I2S 缓冲太小或优先级低 | 增大 I2S DMA buffer；提高输出任务优先级 |
| Classic BT 在 esp32c3 不可用 | SoC 无 Classic BT | A2DP 仅 esp32 支持；其他芯片改用 BLE Audio（`esp-ble-audio.rst`，需支持 LE Audio 的 SoC） |

## 参考

- `examples/bluetooth/bluedroid/classic_bt/a2dp_sink_stream` — Sink + I2S 输出（本 recipe 主来源，含内部/外部 codec 分支）
- `examples/bluetooth/bluedroid/classic_bt/a2dp_source` — A2DP Source 发送端
- `examples/bluetooth/bluedroid/classic_bt/avrcp_absolute_volume` — AVRCP 绝对音量控制
- `examples/bluetooth/bluedroid/classic_bt/avrcp_ct_metadata` — AVRCP 元数据（曲目/艺术家）
- ESP-IDF `components/bt/host/bluedroid/api/include/api/esp_a2dp_api.h`、`esp_avrc_api.h`、`esp_gap_bt_api.h`
- 文档 `docs/en/api-reference/bluetooth/esp_a2dp.rst`、`esp_avrc.rst`、`classic_bt.rst`
