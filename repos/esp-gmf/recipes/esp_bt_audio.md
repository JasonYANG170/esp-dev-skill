# 蓝牙音频（经典 BT A2DP/HFP + LE Audio）

> **适用摘要**: 用 `esp_bt_audio` 统一管理经典蓝牙（A2DP Sink/Source 扬声器耳机、HFP HF/AG 免提通话、AVRCP、PBAP 通讯录）与 LE Audio（TMAP/BAP 单播与广播、VCP/MCP/CSIP 等）。一次 `esp_bt_audio_init` 自动初始化对应协议栈，profile 差异收敛到单一事件回调；音频数据可经 `esp_bt_audio_stream_*` 直读直写，或用 `esp_gmf_io_bt` 接入 GMF 流水线解码/编码后送喇叭/麦克风。

> Version: selected repo/component version; align with the user project and dependency manifest.
> Evidence: `repos/esp-gmf/resources/`, source/examples in `repos/esp-gmf/`, and this recipe path `repos/esp-gmf/recipes/esp_bt_audio.md`.
> Validation: draft metadata added from repository routing; verify APIs, Kconfig symbols, and component dependencies against the selected version.

## 触发意图

- "蓝牙音箱/耳机"
- "A2DP Sink/Source"
- "HFP 免提通话"
- "LE Audio / TMAP / Auracast 广播"
- "esp_bt_audio"
- "AVRCP 播放控制/元数据"
- "PBAP 通讯录/通话记录"

## 前置条件

| 条件 | 要求 |
|---|---|
| 硬件 | 经典 BT：`lyrat_mini_v1_1` 等含 DAC/ADC/I2S 的板；LE Audio：支持 BLE ISO 的目标（ESP32-S31/C5 等）+ 音频 codec |
| ESP-IDF | `release/v5.5` (>= v5.5.2)；经典需 `CONFIG_BT_BLUEDROID_ENABLED` + `CONFIG_BT_CLASSIC_ENABLED`；LE 需 `CONFIG_BT_NIMBLE_ENABLED` + `CONFIG_BT_AUDIO` + `CONFIG_BT_ISO` |
| 组件 | `espressif/esp_bt_audio`（含 `esp_gmf_io_bt`）；板级资源经 `esp_board_manager` |
| 参考头文件 | `packages/esp_bt_audio/include/esp_bt_audio.h`、`_classic.h`、`_le.h`、`_host.h`、`_event.h`、`_stream.h`、`_playback.h`、`_tel.h`、`_vol.h`、`_media.h` |
| 参考示例 | `packages/esp_bt_audio/examples/bt_audio`（含串口命令控制、可选 LVGL UI） |

## 分步说明

`esp_bt_audio` 把协议栈细节封装到三件事：**初始化**（一次 `init` + 角色位掩码）、**事件**（统一 `event_cb` 拿连接/流/媒体/电话/通讯录/音量）、**数据流**（按 `stream_handle` 直读写，或挂 `esp_gmf_io_bt` 进 pipeline）。下面的配置代码改编自示例 `main.c`，按 `CONFIG_GMF_EXAMPLE_AUDIO_TECH_*` 区分经典/LE 分支。

### 1. 初始化 BT 控制器与 host（Bluedroid 或 NimBLE）

```c
#include "esp_bt.h"
#include "esp_bt_main.h"
#include "esp_bt_audio.h"
#include "esp_bt_audio_host.h"

// 1) BT controller（经典 BR/EDR 还是 BLE 取决于 CONFIG）
uint32_t btmode =
#if CONFIG_BTDM_CTRL_MODE_BTDM && CONFIG_BT_CLASSIC_ENABLED
    ESP_BT_MODE_BTDM;
#else
    ESP_BT_MODE_BLE;
#endif
esp_bt_controller_config_t bt_cfg = BT_CONTROLLER_INIT_CONFIG_DEFAULT();
esp_bt_controller_init(&bt_cfg);
esp_bt_controller_enable(btmode);

// 2) host config（按编译开关二选一；init 时统一传入 host_config）
void *host_config = NULL;
#if CONFIG_BT_NIMBLE_ENABLED
esp_bt_audio_host_nimble_cfg_t nimble_cfg = ESP_BT_AUDIO_HOST_NIMBLE_CFG_DEFAULT();   // dev_name = "esp_ble"
// 可在 dev_name 后追加 MAC 后缀以便多机区分（见示例 append_mac_suffix_to_device_name）
host_config = &nimble_cfg;
#elif CONFIG_BT_BLUEDROID_ENABLED
esp_bt_audio_host_bluedroid_cfg_t bd_cfg = ESP_BT_AUDIO_HOST_BLUEDROID_CFG_DEFAULT(); // dev_name="esp_classic", PIN="1234"
host_config = &bd_cfg;
#endif
```

> `ESP_BT_AUDIO_HOST_NIMBLE_CFG_DEFAULT()` 仅给 `dev_name`；`ESP_BT_AUDIO_HOST_BLUEDROID_CFG_DEFAULT()` 还带 PIN/pairing IO capability（默认 `ESP_BT_IO_CAP_NONE`、`ESP_BT_SP_IOCAP_MODE`）。

### 2. 组装 classic + le 角色位掩码

角色用位或叠加，`esp_bt_audio_init` 内部按启用位自动起对应 profile。

```c
// 经典角色
uint32_t classic_roles = 0;
classic_roles |= ESP_BT_AUDIO_CLASSIC_ROLE_A2DP_SNK;   // 接收音乐（音箱/耳机）
classic_roles |= ESP_BT_AUDIO_CLASSIC_ROLE_A2DP_SRC;   // 发送音乐（推给远端耳机）
classic_roles |= ESP_BT_AUDIO_CLASSIC_ROLE_HFP_HF;     // 免提端（接听电话）
classic_roles |= ESP_BT_AUDIO_CLASSIC_ROLE_HFP_AG;     // 音频网关（拨号方）
classic_roles |= ESP_BT_AUDIO_CLASSIC_ROLE_AVRC_CT;    // 远端控制（按键→手机）
classic_roles |= ESP_BT_AUDIO_CLASSIC_ROLE_AVRC_TG;    // 被控端（响应 play/pause）
classic_roles |= ESP_BT_AUDIO_CLASSIC_ROLE_PBAP_PCE;   // 拉通讯录/通话记录

// LE Audio：TMAP 角色来自 ESP_BLE_AUDIO_TMAP_ROLE_*（CT/UMR/BMR/BMS）
uint32_t le_roles = 0;
le_roles |= ESP_BLE_AUDIO_TMAP_ROLE_CT;     // Call Terminal（单播通话/媒体接收）
le_roles |= ESP_BLE_AUDIO_TMAP_ROLE_BMR;    // Broadcast Media Receiver
le_roles |= ESP_BLE_AUDIO_TMAP_ROLE_UMR;    // Unicast Media Receiver
```

经典角色枚举值：`A2DP_SRC=0x01 / A2DP_SNK=0x02 / HFP_HF=0x04 / HFP_AG=0x08 / AVRC_CT=0x10 / AVRC_TG=0x20 / PBAP_PCE=0x40`（`esp_bt_audio_role_t`）。LE 角色：`UNICAST_SERVER / BROADCAST_SINK / BROADCAST_SOURCE / SCAN_DELEGATOR`（`esp_bt_audio_le_role_t`）。

### 3. esp_bt_audio_init（含 PACS/CSIP/VCP/broadcast 配置）

```c
static void bt_event_cb(esp_bt_audio_event_t event, void *data, void *ctx);   // 见第 5 步

esp_bt_audio_config_t bt_config = {
    .host_config     = host_config,
    .event_cb        = bt_event_cb,
    .event_user_ctx  = NULL,
#if CONFIG_BT_CLASSIC_ENABLED
    .classic.roles   = classic_roles,
    .classic.a2dp_src_send_task_core_id   = 0,        // A2DP Source 才用
    .classic.a2dp_src_send_task_prio      = 10,
    .classic.a2dp_src_send_task_stack_size = 4096,
#endif
#if CONFIG_BT_AUDIO
    .le.user_case = ESP_BT_AUDIO_LE_USER_CASE_TMAP,   // 或 HAP / PBP / UNKNOWN
    .le.roles     = le_roles,
    .le.snk_cnt   = 1,                                 // 单播 server 注册的 sink ASE 数
    .le.src_cnt   = 0,                                 // source ASE 数（上行麦克风给手机）
    .le.pacs = {
        .sink_enabled        = 1,
        .sink_context_mask   = ESP_BLE_AUDIO_CONTEXT_TYPE_ANY,   // 接受任意上下文
        .sink_locations      = ESP_BT_AUDIO_AUDIO_LOC_FRONT_LEFT, // 双耳组用时左/右各一
        .source_enabled      = 0,
        .source_context_mask = 0,
        .source_locations    = 0,
    },
    .le.vcp = { .volume = 50, .mute = 0, .step = 10 },          // VCP volume renderer
    .le.csip = {                                                 // CSIP 协调组（双耳）
        .coordinate_set_size = 2,
        .rank = 1,
        .sirk = { /* 16 字节 Set Identity Resolving Key */ },
    },
    .le.bsrc = { .broadcast_code = {0}, .broadcast_name = {0}, .stream_num = 1 },  // 广播源才用
#endif
};

ESP_ERROR_CHECK(esp_bt_audio_init(&bt_config));

// 按 A2DP 方向设 scan 模式：Sink 可被发现可被连；Source 可连不发现
#if CONFIG_GMF_EXAMPLE_A2DP_SINK
esp_bt_audio_classic_set_scan_mode(true, true);    // connectable + discoverable
#elif CONFIG_GMF_EXAMPLE_A2DP_SOURCE
esp_bt_audio_classic_set_scan_mode(true, false);   // connectable only
#endif

// 订阅远端通知（play_status / track_change / position / now_playing 等）
esp_bt_audio_playback_reg_notifications(
    ESP_BT_AUDIO_PLAYBACK_EVENT_PLAY_STATUS_CHANGE |
    ESP_BT_AUDIO_PLAYBACK_EVENT_TRACK_CHANGE |
    ESP_BT_AUDIO_PLAYBACK_EVENT_TRACK_REACHED_END |
    ESP_BT_AUDIO_PLAYBACK_EVENT_PLAY_POS_CHANGED |
    ESP_BT_AUDIO_PLAYBACK_EVENT_NOW_PLAYING_CHANGE);
```

> `le.pacs.sink_locations` 取 `ESP_BT_AUDIO_AUDIO_LOC_FRONT_LEFT`(0x01) / `_FRONT_RIGHT`(0x02)；`context` 枚举见 `esp_bt_audio_stream_context_t`：`UNSPECIFIED / CONVERSATIONAL / MEDIA`。

### 4. 发现、连接、媒体传输控制

经典用 `classic_discovery_start/connect`；LE 用 `le_scan_start/le_connect` 或 `le_broadcast_sync`。

```c
// 经典：发现并连接对端
esp_bt_audio_classic_discovery_start();
// 在 ESP_BT_AUDIO_EVENT_DEVICE_DISCOVERED 里拿到 addr 后：
uint8_t peer[6] = { /* from event */ };
esp_bt_audio_classic_connect(ESP_BT_AUDIO_CLASSIC_ROLE_A2DP_SNK, peer);

// LE：扫描后连接（或按 broadcast_name 同步广播）
esp_bt_audio_le_scan_start(10000);                     // 10 s
// 事件 DEVICE_DISCOVERED 拿到 addr_type + addr 后：
esp_bt_audio_le_connect(addr_type, peer, 15000);       // 15 s 超时
// Auracast 接收：
esp_bt_audio_le_broadcast_sync(broadcast_name, broadcast_code, /*BIS bitfield*/0xFFFFFFFF, 15000);

// A2DP Source：连接成功后启动本地音频推送
esp_bt_audio_media_start(ESP_BT_AUDIO_CLASSIC_ROLE_A2DP_SRC, NULL);
// 停止推送：
esp_bt_audio_media_stop(ESP_BT_AUDIO_CLASSIC_ROLE_A2DP_SRC);
```

### 5. 统一事件回调（连接/流/媒体/电话/通讯录/音量）

所有 profile 差异在事件层收敛；按 `event` 强转 `event_data` 为对应结构。

```c
static void bt_event_cb(esp_bt_audio_event_t event, void *data, void *ctx) {
    switch (event) {
    case ESP_BT_AUDIO_EVENT_CONNECTION_STATE_CHG: {
        esp_bt_audio_event_connection_st_t *c = data;
        ESP_LOGI(TAG, "conn %s tech=%d addr=%02x:%02x:%02x:%02x:%02x:%02x",
                 c->connected ? "UP" : "DOWN", c->tech, c->addr[0], c->addr[1], c->addr[2],
                 c->addr[3], c->addr[4], c->addr[5]);
        break;
    }
    case ESP_BT_AUDIO_EVENT_STREAM_STATE_CHG: {
        esp_bt_audio_event_stream_st_t *s = data;
        // 关键：每次流状态变化把 stream_handle 传给 GMF pipeline（见第 6 步）
        stream_proc_state_chg(s->stream_handle, s->state);
        break;
    }
    case ESP_BT_AUDIO_EVENT_MEDIA_CTRL_CMD: {
        // 远端发的媒体键（play/pause/next/prev）——sink 才会收到
        esp_bt_audio_media_ctrl_cmd_t cmd = ((esp_bt_audio_event_media_ctrl_t *)data)->cmd;
        break;
    }
    case ESP_BT_AUDIO_EVENT_PLAYBACK_STATUS_CHG: {
        // 已注册的远端通知：play_status/track_change/position 等
        esp_bt_audio_event_playback_st_t *st = data;
        break;
    }
    case ESP_BT_AUDIO_EVENT_PLAYBACK_METADATA: {
        // title/artist/album/cover_art（cover_art 走特殊子结构）
        esp_bt_audio_event_playback_metadata_t *m = data;
        break;
    }
    case ESP_BT_AUDIO_EVENT_VOL_ABSOLUTE: {
        esp_bt_audio_event_vol_absolute_t *v = data;
        // v->vol / v->mute / v->context
        break;
    }
    case ESP_BT_AUDIO_EVENT_CALL_STATE_CHG: {
        esp_bt_audio_event_call_state_t *call = data;   // idx/dir/state/uri
        if (call->state == ESP_BT_AUDIO_CALL_STATE_INCOMING) {
            esp_bt_audio_call_answer(call->idx);
        }
        break;
    }
    case ESP_BT_AUDIO_EVENT_PHONEBOOK_ENTRY: /* esp_bt_audio_pb_entry_t */ break;
    case ESP_BT_AUDIO_EVENT_PHONEBOOK_HISTORY: /* esp_bt_audio_pb_history_t */ break;
    case ESP_BT_AUDIO_EVENT_BIG_SYNC_LOST:   // LE 广播 BIG 失步
    case ESP_BT_AUDIO_EVENT_PA_SYNC_LOST:    // LE 广播 PA 失步
        break;
    default: break;
    }
}
```

事件枚举见 `esp_bt_audio_event_t`：连接/发现/设备/流/媒体键/播放状态/元数据/绝对音量/相对音量/电话状态/来电状态/通讯录计数/通讯录条目/通话历史/BIG/PA 失步。

### 6. 接入 GMF pipeline（`esp_gmf_io_bt` 把 stream 接到流水线）

音频数据流向由 `esp_bt_audio_stream_get_dir` 决定：`SINK`（接收，需解码→DAC）或 `SOURCE`（上行，需 ADC→编码）。示例在 `STREAM_STATE_STARTED` 时按方向动态建流水线。

```c
#include "esp_gmf_io_bt.h"
#include "esp_bt_audio_stream.h"

// 在 STREAM_STATE_STARTED 事件里：
void stream_proc_state_chg(esp_bt_audio_stream_handle_t stream, esp_bt_audio_stream_state_t state) {
    if (state != ESP_BT_AUDIO_STREAM_STATE_STARTED) return;

    esp_bt_audio_stream_codec_info_t ci = {0};
    esp_bt_audio_stream_get_codec_info(stream, &ci);   // codec_type=SBC|LC3, sample_rate/channels/bits/codec_cfg

    esp_bt_audio_stream_dir_t dir = ESP_BT_AUDIO_STREAM_DIR_UNKNOWN;
    esp_bt_audio_stream_get_dir(stream, &dir);

    if (dir == ESP_BT_AUDIO_STREAM_DIR_SINK) {
        // 接收方向：io_bt(in) → aud_dec → ... → codec_dev(out)
        esp_gmf_io_bt_set_stream(ESP_GMF_PIPELINE_GET_IN_INSTANCE(pipe), stream);
        // 解码器配置用 ci.codec_cfg（codec_type 选 SBC / LC3）
    } else if (dir == ESP_BT_AUDIO_STREAM_DIR_SOURCE) {
        // 上行方向：codec_dev(in) → aud_enc → ... → io_bt(out)
        esp_gmf_io_bt_set_stream(ESP_GMF_PIPELINE_GET_OUT_INSTANCE(pipe), stream);
    }
}
```

> 直读直写方式（不经 pipeline）：用 `esp_bt_audio_stream_acquire_read` / `release_read`（接收 PCM/裸帧）或 `acquire_write` / `release_write`（发送），`packet->data`/`size`/`bad_frame`/`is_done` 字段；LE Audio 还能 `esp_bt_audio_stream_get_iso_interval` 拿 ISO 间隔做时钟同步。

### 7. 媒体控制（远端播放器 + 音量 + 通话）

```c
// 作为 AVRCP Controller（按键→手机）
esp_bt_audio_playback_play();
esp_bt_audio_playback_pause();
esp_bt_audio_playback_stop();
esp_bt_audio_playback_next();
esp_bt_audio_playback_prev();
esp_bt_audio_playback_request_metadata(ESP_BT_AUDIO_PLAYBACK_METADATA_TITLE |
                                       ESP_BT_AUDIO_PLAYBACK_METADATA_ARTIST |
                                       ESP_BT_AUDIO_PLAYBACK_METADATA_COVER_ART);

// 绝对/相对音量（需 AVRC_CT 角色）
esp_bt_audio_vol_set_absolute(80);     // 0-127
esp_bt_audio_vol_set_relative(true);   // true=up, false=down

// 通话控制（需 HFP HF/AG 或 LE 电话）
esp_bt_audio_call_answer(idx);         // 接听（经典 idx 忽略）
esp_bt_audio_call_reject(idx);         // 挂断/拒接
esp_bt_audio_call_dial("10086");       // 拨号（AG/HFP HF 上行）
```

### 8. 销毁

```c
esp_bt_audio_deinit();
esp_bt_audio_host_deinit();
```

## 常见错误

| 错误 | 原因 | 解决方法 |
|---|---|---|
| `esp_bt_audio_init` 返回非 OK | controller 未 init/enable 或 host_config 与编译开关不匹配 | 先 `esp_bt_controller_init/enable`；NimBLE 用 `_NIMBLE_CFG_DEFAULT`、Bluedroid 用 `_BLUEDROID_CFG_DEFAULT` |
| 经典角色 API 返回 `INVALID_STATE` | classic 未启用（`CONFIG_BT_BLUEDROID_ENABLED`+`CONFIG_BT_CLASSIC_ENABLED` 关） | 启用对应 Kconfig；LE 与经典共存的 SoC（如 S31）需按 sdkconfig.defaults 选择 `.classic` 或 `.le` |
| LE API 返回 `INVALID_STATE` | `CONFIG_BT_NIMBLE_ENABLED`+`CONFIG_BT_AUDIO`+`CONFIG_BT_ISO` 未全开 | 全部启用并选支持 BLE ISO 的目标芯片 |
| 没声音（Sink） | 流 STARTED 但没建解码 pipeline / codec_dev 未按 `ci.sample_rate` 重配 | 在 `STREAM_STATE_STARTED` 里建/重配 pipeline；用 `esp_bt_audio_stream_get_codec_info` 取真实采样率 |
| HFP 通话有回声 | 未启用 AEC | 在 GMF pipeline 里加 AEC 元素（见 `gmf_ai_audio.md` / `pipeline_record.md` AEC 路径） |
| LE 组播双耳不同步 | CSIP 未配或 set size 不匹配 | 配 `le.csip.coordinate_set_size`、相同 SIRK、不同 rank |
| 广播接收无声 | BIG/PA sync 丢失或 broadcast_code 错 | 看事件 `BIG_SYNC_LOST`/`PA_SYNC_LOST`；加密广播需正确 `broadcast_code` |
| metadata 取不到 | 未 `request_metadata` 或未 `reg_notifications` | 先调 `esp_bt_audio_playback_request_metadata(mask)`；订阅对应 `ESP_BT_AUDIO_PLAYBACK_EVENT_*` |
| A2DP Source 推流卡顿 | send task 栈/优先级不足 | 调 `classic.a2dp_src_send_task_stack_size`（默认 4096）/ `prio`（默认 10） |

## 参考

- `packages/esp_bt_audio/examples/bt_audio/main/main.c`（init 配置、事件回调、媒体控制）
- `packages/esp_bt_audio/examples/bt_audio/main/stream_proc.c`（`esp_gmf_io_bt` 接入 pipeline、按方向建解码/编码链）
- `packages/esp_bt_audio/examples/bt_audio/main/cmd_reg.c`（串口命令控制台）
- `packages/esp_bt_audio/include/esp_bt_audio.h`、`_classic.h`、`_le.h`、`_host.h`、`_event.h`、`_stream.h`、`_playback.h`、`_tel.h`、`_vol.h`、`_media.h`、`_defs.h`
- `docs/en/gmf-framework/gmf-package/esp-bt-audio.rst`、`docs/zh_CN/gmf-framework/gmf-package/esp-bt-audio.rst`
