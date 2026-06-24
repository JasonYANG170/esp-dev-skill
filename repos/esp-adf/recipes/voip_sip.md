# VoIP / SIP 网络电话：SIP/RTP 通话与 AEC 回声消除

> **适用摘要**: 用 `av_stream_init` 建一条带 AEC 的双向音频管线（mic→算法流→G.711 编码→RTP；RTP→G.711 解码→算法流→喇叭），再用 SIP 服务（基于 `esp_rtc`）注册到 PBX（FreeSWITCH/Asterisk），实现接/打电话。`wifi_service` 管联网，`input_key_service` 把按键映射到 call/answer/hangup/volume。数据流参考 `examples/protocols/voip`。

## 触发意图

- "VoIP / 网络电话 / SIP 通话"
- "SIP 注册 / INVITE / RTP"
- "AEC 回声消除 / 通话有回音"
- "FreeSWITCH / Asterisk / PBX"
- "voip 那个例子"
- "G.711 / PCMA / PCMU 编解码"

## 前置条件

| 条件 | 要求 |
|---|---|
| 参考示例 | `examples/protocols/voip/main/voip_app.c`、`sip_service.c`、`sip_service.h`；`examples/get-started/pipeline_a2dp_sink_and_hfp` |
| PBX 服务器 | FreeSWITCH（推荐）/ Asterisk / Kamailio / Yate，已建好分机号 |
| 网络 | Wi-Fi 联网（`wifi_service` + smart_config 或固定 SSID） |
| 配置 | `menuconfig` 填 `WiFi SSID/PASSWORD`、`SIP_URI`（如 `tcp://100:100@192.168.1.123:5060`）、`SIP Codec`（G711A/PCMA 或 G711U/PCMU） |
| 提示音 | 需烧录 audio flash tone 到 `0x210000`（`audio_flash_tone`，分区大小需与 partitions.csv 对应） |
| 硬件 | 推荐 `ESP32-S3-Korvo-2L` 或 `ESP32-LyraT-Mini`（双麦 AEC 效果好） |
| 文档 | `examples/protocols/voip/README.md`；`docs/en/api-reference/streams/index.rst`（algorithm_stream / raw_stream 引用 voip 为参考应用） |

## 分步说明

### 1. 整体结构（voip_app.c）

```
[PBX: FreeSWITCH]
       ↑↓ SIP (REGISTER/INVITE/BYE) + RTP (G.711)
[Wi-Fi] → wifi_service → sip_service_start(av_stream, SIP_URI) → esp_rtc_handle_t
                            ↓
            av_stream (内部: i2s+algorithm_stream(AEC)+raw_stream+G711)
                            ↓
         input_key_service 按键 → esp_rtc_call/answer/bye
```

### 2. 初始化 av_stream（双向音频 + AEC）

```c
#include "algorithm_stream.h"   // 提供 AEC/AGC/NS

// voip_app.c 关键结构
av_stream_config_t av_stream_config = {
    .enable_aec          = true,                 // 启用回声消除
    .acodec_samplerate   = AUDIO_CODEC_SAMPLE_RATE,  // G.711 通常 8000Hz
    .acodec_type         = AV_ACODEC_G711A,      // 或 AV_ACODEC_G711U（对应 menuconfig SIP_CODEC_*）
    .vcodec_type         = AV_VCODEC_NULL,       // 纯语音，无视频
    .hal = {
        .audio_samplerate = AUDIO_HAL_SAMPLE_RATE,
        .audio_framesize  = PCM_FRAME_SIZE,
    },
};
av_stream_handle_t av_stream = av_stream_init(&av_stream_config);
```

> `av_stream` 内部封装了 mic/speaker 双向管线：用 `algorithm_stream` 做 AEC（参考音取自播放端），用 `raw_stream` 桥接 RTP 数据。这就是 `streams/index.rst` 把 voip 列为 algorithm_stream/raw_stream 参考应用的原因。

### 3. 初始化提示音播放器（通话前的状态提示）

```c
audio_player_int_tone_init(AUDIO_HAL_SAMPLE_RATE, I2S_CHANNELS, I2S_DEFAULT_BITS);
// 通话中也可以插播：audio_player_int_tone_play(tone_uri[TONE_TYPE_...]);
// 切歌/接听时停掉：  audio_player_int_tone_stop();
```

### 4. 建 Wi-Fi 服务（联网后才起 SIP）

```c
#include "wifi_service.h"
#include "smart_config.h"

wifi_service_config_t cfg = WIFI_SERVICE_DEFAULT_CONFIG();
cfg.evt_cb             = wifi_service_cb;   // 见下
cfg.setting_timeout_s  = 300;
cfg.max_retry_time     = 2;
periph_service_handle_t wifi_serv = wifi_service_create(&cfg);

// 配 smart_config（Esptouch）作为备选配网
smart_config_info_t info = SMART_CONFIG_INFO_DEFAULT();
esp_wifi_setting_handle_t h = smart_config_create(&info);
esp_wifi_setting_register_notify_handle(h, (void *)wifi_serv);
int reg_idx = 0;
wifi_service_register_setting_handle(wifi_serv, h, &reg_idx);

wifi_config_t sta_cfg = {0};
strncpy((char *)sta_cfg.sta.ssid,     CONFIG_WIFI_SSID,     sizeof(sta_cfg.sta.ssid));
strncpy((char *)sta_cfg.sta.password, CONFIG_WIFI_PASSWORD, sizeof(sta_cfg.sta.password));
wifi_service_set_sta_info(wifi_serv, &sta_cfg);
wifi_service_connect(wifi_serv);
```

> 注意 voip 用的是高层 `wifi_service`（不是 `periph_wifi`），事件是 `WIFI_SERV_EVENT_CONNECTED/DISCONNECTED/SETTING_TIMEOUT`。

### 5. Wi-Fi 连上后启动 SIP 服务

```c
static esp_rtc_handle_t esp_sip;

static esp_err_t wifi_service_cb(periph_service_handle_t handle,
                                 periph_service_event_t *evt, void *ctx)
{
    if (evt->type == WIFI_SERV_EVENT_CONNECTED) {
        ESP_LOGI(TAG, "[ 5 ] Create SIP Service");
        esp_sip = sip_service_start(av_stream, CONFIG_SIP_URI);   // SIP_URI 来自 menuconfig
    } else if (evt->type == WIFI_SERV_EVENT_DISCONNECTED) {
        audio_player_int_tone_play(tone_uri[TONE_TYPE_PLEASE_SETTING_WIFI]);
    } else if (evt->type == WIFI_SERV_EVENT_SETTING_TIMEOUT) {
        audio_player_int_tone_play(tone_uri[TONE_TYPE_PLEASE_SETTING_WIFI]);
    }
    return ESP_OK;
}
```

> `sip_service_start` 内部调 `esp_rtc_service_init`（见 `sip_service.c`），返回的 `esp_rtc_handle_t` 用于后续 call/answer/bye。SIP URI 格式：`<transport>://<user>:<password>@<server>:<port>`，transport 为 `tcp`/`udp`/`tls`。

### 6. 按键映射到通话操作

```c
#include "input_key_service.h"

static esp_err_t input_key_service_cb(periph_service_handle_t handle,
                                      periph_service_event_t *evt, void *ctx)
{
    if (evt->type == INPUT_KEY_SERVICE_ACTION_CLICK_RELEASE) {
        switch ((int)evt->data) {
            case INPUT_KEY_USER_ID_REC:    esp_rtc_call(esp_sip, "1002");   break; // 去电
            case INPUT_KEY_USER_ID_PLAY:   audio_player_int_tone_stop();
                                            esp_rtc_answer(esp_sip);          break; // 接听
            case INPUT_KEY_USER_ID_MODE:
            case INPUT_KEY_USER_ID_SET:    audio_player_int_tone_stop();
                                            esp_rtc_bye(esp_sip);             break; // 挂断
            case INPUT_KEY_USER_ID_VOLUP:  /* av_audio_set_vol(...+10) */   break;
            case INPUT_KEY_USER_ID_VOLDOWN:/* av_audio_set_vol(...-10) */   break;
        }
    } else if (evt->type == INPUT_KEY_SERVICE_ACTION_PRESS) {
        if ((int)evt->data == INPUT_KEY_USER_ID_SET) {
            // 长按 SET 进 smart_config 配网
            sip_service_stop(esp_sip);
            wifi_service_setting_start(wifi_serv, 0);
            audio_player_int_tone_play(tone_uri[TONE_TYPE_UNDER_SMARTCONFIG]);
        }
    }
    return ESP_OK;
}

// 注册按键服务
input_key_service_cfg_t input_cfg = INPUT_KEY_SERVICE_DEFAULT_CONFIG();
input_cfg.handle = set;
input_cfg.based_cfg.task_stack = 4 * 1024;
input_cfg.based_cfg.extern_stack = true;
periph_service_handle_t input_ser = input_key_service_create(&input_cfg);
input_key_service_info_t input_key_info[] = INPUT_KEY_DEFAULT_INFO();
input_key_service_add_key(input_ser, input_key_info, INPUT_KEY_NUM);
periph_service_set_callback(input_ser, input_key_service_cb, wifi_serv);
```

### 7. AEC 调试（通话有回音时）

若 AEC 效果差，打开 `DEBUG_AEC_INPUT` 宏，把原始双声道数据（左=麦克风采集，右=喇叭播放参考）录到 SD 卡，用音频分析工具量延迟：

```c
// 编译时开 DEBUG_AEC_INPUT，运行时需先挂 SD
#if (DEBUG_AEC_INPUT || DEBUG_AEC_OUTPUT)
audio_board_sdcard_init(set, SD_MODE_1_LINE);
#endif
```

> README 明确：**AEC 内部缓冲机制要求录音信号比对应参考（播放）信号延迟 0–10ms**。超出此范围 AEC 失效。调采样率/缓冲使其落入窗口。

### 8. FreeSWITCH 推荐配置（README 列出）

- 关 `NOTIFY`：`conf/sip_profiles/internal.xml` 设 `<param name="send-message-query-on-register" value="false"/>`
- 关服务器定时器：`<param name="enable-timer" value="false"/>`
- 删不支持的 Video Codec（`conf/vars.xml`）
- 关自动应答：`conf/dialplan/default.xml` 设 `sip_auto_answer=false`
- 注释掉强制 answer+bridge 三行（避免抢答）

### 9. 烧录提示音到 Flash

```bash
python $ADF_PATH/esp-idf/components/esptool_py/esptool/esptool.py \
  --chip esp32 --port /dev/ttyUSB0 --baud 921600 \
  --before default_reset --after hard_reset write_flash -z \
  --flash_mode dio --flash_freq 40m --flash_size detect \
  0x210000 ../components/audio_flash_tone/bin/audio-esp.bin
```

> `0x210000` 必须与 `partitions.csv` 里的 tone 分区 offset 对应。

## 常见错误

| 错误 | 原因 | 解决方法 |
|---|---|---|
| SIP 注册失败（401 后无 200） | 用户名/密码/realm 错 | 核对 SIP_URI 的 `user:password@server`；用 Linphone/MicroSIP 先验证服务器 |
| 注册成功但通话无声 | codec 不匹配 | menuconfig 的 `SIP_CODEC_*` 与 PBX 协商一致（PCMA=G711A / PCMU=G711U） |
| 通话有回音 / 对方听到自己 | AEC 延迟超出 0-10ms 窗口 | 开 `DEBUG_AEC_INPUT` 量延迟，调采样率/缓冲；确认 `enable_aec=true` |
| Esptouch 配网崩溃 | `SC_ACK_TASK_STACK_SIZE` 太小 | 适当增大该栈（README troubleshooting） |
| RTT 长 / 通话卡 | SIP 日志太多 | `esp_log_level_set("SIP", ESP_LOG_WARN)` |
| 提示音不响 | 未烧 audio_flash_tone | 按 step 9 烧到 `0x210000` |
| 联网后不自动注册 | wifi_service 回调没接 SIP | 在 `WIFI_SERV_EVENT_CONNECTED` 里调 `sip_service_start` |
| 内存不足 | av_stream + SIP + Wi-Fi 占用大 | README 给出参考内存：LyraT-Mini 约 392KB total，其他板约 253KB；开 PSRAM |

## 参考项目

- `examples/protocols/voip/main/voip_app.c` — 完整 VoIP 应用主程序（Wi-Fi 服务 + av_stream + 按键）
- `examples/protocols/voip/main/sip_service.c`、`sip_service.h` — 基于 `esp_rtc` 的 SIP 服务封装
- `examples/protocols/voip/README.md` — PBX 配置、SIP 注册日志、AEC 调试、烧录说明
- `examples/get-started/pipeline_a2dp_sink_and_hfp` — A2DP + HFP 共存（相关免提通话）
- `examples/protocols/esp-rtc` — 底层 SIP/RTSP/RTCP 实现
- `docs/en/api-reference/streams/index.rst` — algorithm_stream / raw_stream 文档把 voip 列为参考应用
