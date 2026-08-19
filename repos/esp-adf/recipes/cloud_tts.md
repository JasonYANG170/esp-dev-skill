# 云端 TTS 语音合成：百度 / AWS Polly 在线 TTS

> **适用摘要**: 调云端 TTS 接口把文本转成 MP3，再走 `http_stream → mp3_decoder → i2s_stream` 播放。百度用 API Key/Secret Key 换 access_token 后 POST 表单（`http://tsn.baidu.com/text2audio`）；AWS Polly 需 SNTP 校时 + AWS4-HMAC-SHA256 签名（`https://polly.<region>.amazonaws.com/v1/speech`）。两者都用 `http_stream` 的 `event_handle` 回调在 `HTTP_STREAM_PRE_REQUEST` 阶段注入鉴权头/请求体。数据流：`[cloud TTS] → http_stream(reader) → mp3_decoder → i2s_stream(writer) → codec`。

> Version: selected repo/component version; align with the user project and dependency manifest.
> Evidence: `repos/esp-adf/resources/`, source/examples in `repos/esp-adf/`, and this recipe path `repos/esp-adf/recipes/cloud_tts.md`.
> Validation: draft metadata added from repository routing; verify APIs, Kconfig symbols, and component dependencies against the selected version.

## 触发意图

- "TTS / 文字转语音 / 语音合成"
- "百度语音 / Baidu Speech TTS"
- "AWS Polly / 亚马逊 Polly"
- "Google Translate 念出来"
- "在线 TTS pipeline"
- "pipeline_baidu_speech_mp3 / pipeline_aws_polly_mp3 那个例子"

## 前置条件

| 条件 | 要求 |
|---|---|
| 参考示例 | `examples/cloud_services/pipeline_baidu_speech_mp3/main/play_baidu_speech_mp3_example.c`、`examples/cloud_services/pipeline_aws_polly_mp3/main/play_aws_polly_mp3_example.c`、`examples/cloud_services/google_translate_device` |
| 网络 | Wi-Fi 联网（`periph_wifi`，先 `wait_for_connected` 再 run pipeline） |
| 时间同步（仅 AWS） | SNTP 校时——AWS 签名需要正确系统时间，Polly 示例开机先 `Initializing SNTP` 等 20 次 |
| 配置 | 百度：menuconfig 填 `Baidu speech access key ID` + `Secret Key`；AWS：`Amazon service access key ID` + `access secret`；都填 `WiFi SSID/PASSWORD` |
| 头文件 | `http_stream.h`、`mp3_decoder.h`、`i2s_stream.h`；百度示例自带 `baidu_access_token.h`（`baidu_get_access_token`） |

## 分步说明

### 1. 通用 pipeline 拓扑

```
[cloud TTS server] ---> http_stream(reader) ---> mp3_decoder ---> i2s_stream(writer) ---> [codec]
```

> 与 `play_http_mp3.md` 的区别：URL 不是固定音频文件，而是带鉴权/POST body 的 TTS 请求；需在 `http_stream` 的 `event_handle` 回调里动态生成请求头和请求体。

### 2. 百度语音 TTS

**a. 用 API Key 换 access_token（一次性，token 可缓存）**

```c
#include "baidu_access_token.h"   // 示例自带组件

// CONFIG_BAIDU_ACCESS_KEY / CONFIG_BAIDU_SECRET_KEY 来自 menuconfig
// 返回的字符串用完需 free
char *baidu_access_token = baidu_get_access_token(CONFIG_BAIDU_ACCESS_KEY,
                                                  CONFIG_BAIDU_SECRET_KEY);
```

**b. 在 http_stream 事件回调里注入 POST 表单**

百度用 `HTTP_STREAM_PRE_REQUEST` 事件回调（不是 `ON_REQUEST`），在该回调里拿到 `msg->http_client` 句柄，直接用 `esp_http_client_set_post_field` 写表单、`set_method(POST)`：

```c
#include "http_stream.h"

#define BAIDU_TTS_ENDPOINT  "http://tsn.baidu.com/text2audio"   // 注意是 http，不是 https
#define TTS_TEXT            "欢迎使用乐鑫音频平台，想了解更多方案信息请联系我们"   // 示例默认文本

static int _http_stream_event_handle(http_stream_event_msg_t *msg)
{
    esp_http_client_handle_t http_client = (esp_http_client_handle_t)msg->http_client;

    if (msg->event_id != HTTP_STREAM_PRE_REQUEST) {
        return ESP_OK;
    }
    if (baidu_access_token == NULL) {
        // 用完需 free
        baidu_access_token = baidu_get_access_token(CONFIG_BAIDU_ACCESS_KEY,
                                                    CONFIG_BAIDU_SECRET_KEY);
    }
    if (baidu_access_token == NULL) {
        return ESP_FAIL;
    }

    // 百度 TTS 表单字段：lan=zh & cuid=ESP32 & ctp=1 & tok=<token> & tex=<文本>
    char request_data[1024];
    int data_len = snprintf(request_data, sizeof(request_data),
                            "lan=zh&cuid=ESP32&ctp=1&tok=%s&tex=%s",
                            baidu_access_token, TTS_TEXT);
    esp_http_client_set_post_field(http_client, request_data, data_len);
    esp_http_client_set_method(http_client, HTTP_METHOD_POST);
    return ESP_OK;
}
```

**c. 建 pipeline 并设 URI**

```c
http_stream_cfg_t http_cfg = HTTP_STREAM_CFG_DEFAULT();
http_cfg.event_handle = _http_stream_event_handle;
audio_element_handle_t http_reader = http_stream_init(&http_cfg);
audio_element_set_uri(http_reader, BAIDU_TTS_ENDPOINT);

// mp3_decoder / i2s_stream(writer) 同 play_http_mp3.md
// register + link: {"http", "mp3", "i2s"}
audio_pipeline_run(pipeline);
```

> 百度返回的是 MP3，日志典型为 `sample_rates=16000, bits=16, ch=1`，记得在 `AEL_MSG_CMD_REPORT_MUSIC_INFO` 后 `i2s_stream_set_clk`。

### 3. AWS Polly TTS（需 SNTP + AWS4 签名）

**a. 开机先 SNTP 校时（签名依赖时间）**

Polly 示例定义了 `wait_for_sntp()`，连上 Wi-Fi 后立刻调用，内部 `sntp_setoperatingmode(SNTP_OPMODE_POLL)` + `sntp_setservername(0, "pool.ntp.org")` + `sntp_init()`，然后循环最多 20 次检查 `timeinfo.tm_year` 是否就绪（每 2 秒一次）：

```c
// play_aws_polly_mp3_example.c
static void wait_for_sntp(void) {
    time_t now = 0;
    struct tm timeinfo = { 0 };
    int retry = 0;
    ESP_LOGI(TAG, "Initializing SNTP");
    sntp_setoperatingmode(SNTP_OPMODE_POLL);
    sntp_setservername(0, "pool.ntp.org");
    sntp_init();
    const int retry_count = 20;
    while (timeinfo.tm_year < (2016 - 1900) && ++retry < retry_count) {
        ESP_LOGI(TAG, "Waiting for system time to be set... (%d/%d)", retry, retry_count);
        vTaskDelay(2000 / portTICK_PERIOD_MS);
        time(&now);
        localtime_r(&now, &timeinfo);
    }
}
```

> README 日志：连上 Wi-Fi 后约 2 次重试时间就绪，才开始 codec/pipeline。**没有正确时间，AWS 签名会被拒。**

**b. AWS4-HMAC-SHA256 签名（用 `aws_sig_v4_signing_header` 帮助函数）**

Polly 示例用 `components/aws_sig_v4` 的帮助函数，不手动拼 header。先填 `aws_sig_v4_config_t`（service=`polly`、region=`CONFIG_AWS_POLLY_REGION`、`access_key`/`secret_key`/`host`/`payload`/`amz_date`/`date_stamp`），再调 `aws_sig_v4_signing_header(&sigv4_context, &polly_sigv4_config)` 得到 `Authorization: AWS4-HMAC-SHA256 ...` 头字符串，最后 `esp_http_client_set_header(http_client, "Authorization", auth_header)`：

```c
#define AWS_POLLY_ENDPOINT "https://polly."CONFIG_AWS_POLLY_REGION".amazonaws.com/v1/speech"
// 示例 JSON payload（注意 SampleRate=22050，VoiceId=Joanna）
static const char *polly_payload =
    "{\"OutputFormat\":\"mp3\",\"SampleRate\":\"22050\","
    "\"Text\":\""TTS_TEXT"\",\"TextType\":\"text\",\"VoiceId\":\"Joanna\"}";

aws_sig_v4_config_t polly_sigv4_config = {
    .service_name = "polly",
    .region_name  = CONFIG_AWS_POLLY_REGION,
    .access_key   = CONFIG_AWS_ACCESS_KEY,
    .secret_key   = CONFIG_AWS_SECRET_KEY,
    .host         = "polly."CONFIG_AWS_POLLY_REGION".amazonaws.com",
    // .payload / .payload_len / .amz_date / .date_stamp 在请求时填
};
// 在 http 事件回调里：
char *auth_header = aws_sig_v4_signing_header(&sigv4_context, &polly_sigv4_config);
esp_http_client_set_header(http_client, "Authorization", auth_header);
esp_http_client_set_post_field(http_client, polly_payload, payload_len);
```

> 签名细节（HMAC-SHA256 链、canonical request）封装在 `components/aws_sig_v4`，密钥为 menuconfig 的 `Amazon service access key ID` + `access secret`。示例日志会打印实际签名串（`amz_date=...`、`AWS4-HMAC-SHA256 Credential=...`）。

### 4. 必做：监听 MUSIC_INFO 后重配 I2S

云端返回的 MP3 采样率可能与默认 44100 不同（百度 16kHz、Polly 22050Hz）：

```c
if (msg.cmd == AEL_MSG_CMD_REPORT_MUSIC_INFO && msg.source == mp3_decoder) {
    audio_element_info_t info = {0};
    audio_element_getinfo(mp3_decoder, &info);
    i2s_stream_set_clk(i2s_writer, info.sample_rates, info.bits, info.channels);
}
```

### 5. 流结束处理

HTTP 流读完会发 `AEL_IO_DONE`，pipeline 进入 FINISHED。在事件循环里收到后按停止三连收尾：

```c
if (msg.source == i2s_stream_writer && msg.cmd == AEL_MSG_CMD_REPORT_STATUS) {
    if (audio_element_get_state(i2s_stream_writer) == AEL_STATE_FINISHED) {
        audio_pipeline_stop(pipeline);
        audio_pipeline_wait_for_stop(pipeline);
        audio_pipeline_terminate(pipeline);
    }
}
```

### 6. 三家云 TTS 对照

| 服务 | 鉴权 | 时间依赖 | 默认文本 | 典型采样率 |
|---|---|---|---|---|
| 百度 Speech | API Key + Secret Key → access_token（OAuth2） | 否 | 中文 `欢迎使用乐鑫音频平台...` | 16kHz |
| AWS Polly | Access Key + Secret（AWS4-HMAC-SHA256） | **是（SNTP）** | 英文 Espressif 公司介绍 | 22050Hz |
| Google Translate | URL 参数（见 google_translate_device 示例） | 否 | 翻译结果 | — |

## 常见错误

| 错误 | 原因 | 解决方法 |
|---|---|---|
| AWS 返回 403 Forbidden | 系统时间错 → 签名失效 | 开机先 SNTP 校时（`wait_for_sntp`），等时间就绪再发请求 |
| 百度返回空 / 401 | access_token 失效或 API Key 错 | 重新调 `baidu_get_access_token`；核对 menuconfig 的 key/secret |
| 播放变速 | 没按返回的采样率重配 I2S | 在 `REPORT_MUSIC_INFO` 后 `i2s_stream_set_clk` |
| 编译报 `baidu_access_token.h` 找不到 | 未包含示例组件 | 该头在示例 `components/` 下，工程依赖需包含 |
| HTTP 流卡住 | Wi-Fi 没连上就 run | 先 `periph_wifi_wait_for_connected` 再 `audio_pipeline_run` |
| 内存不足 | TLS/大响应缓冲占用大 | Polly 用 HTTPS 需更大栈；必要时减小 http `buffer_len`，开 PSRAM |
| 中文 TTS 乱码 | 请求文本编码问题 | 百度用 UTF-8；确保源文件以 UTF-8 保存 |
| 反复重连 | token 缓存策略错 | token 有有效期，缓存后到期再换，不要每次请求都换 |

## 参考项目

- `examples/cloud_services/pipeline_baidu_speech_mp3/main/play_baidu_speech_mp3_example.c` — 百度 TTS 完整实现（含 `baidu_access_token` 组件）
- `examples/cloud_services/pipeline_aws_polly_mp3/main/play_aws_polly_mp3_example.c` — AWS Polly 完整实现（SNTP + AWS4 签名）
- `examples/cloud_services/google_translate_device/` — Google 翻译 + TTS 集成
- `docs/en/api-reference/streams/index.rst` — `http_stream` 文档（TTS 的请求注入点）
- `recipes/play_http_mp3.md` — 基础 HTTP MP3 播放（本 recipe 在其上加鉴权与请求体）
